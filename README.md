# DSO202 Practical 5 — Environment-Specific Configuration with Kustomize on Kind

| | |
|---|---|
| **Name** | Kuenzang Rabten |
| **Student ID** | 02230289 |
| **Module** | DSO202 |
| **Practical** | 05 — Environment-Specific Configuration with Kustomize on Kind |

## Purpose

This practical shows how one Kubernetes base can be deployed to several environments (dev, staging, prod, and my own QA) without copying the base Deployment or Service. Each environment is an overlay that holds only what is different: namespace, environment label, replica count, image tag, web content, resource policy and annotations.

It also covers the ConfigMap hash suffix and why changing a file causes a rollout, the render → diff → apply → verify workflow, the difference between strategic merge patches and JSON 6902 patches, and how `namePrefix` renames resources and updates the references to them.

The repository can be rebuilt from `cluster/` and `examples/`. Command output and screenshots are saved in `evidence/`.

## Contents

- [Versions and Images](#versions-and-images)
- [Repository Structure](#repository-structure)
- [Rebuild From an Empty Machine](#rebuild-from-an-empty-machine)
- [1. Objective](#1-objective)
- [2. Environment](#2-environment)
- [3. Procedure and Observations](#3-procedure-and-observations)
  - [Task 0 — Pre-flight](#task-0--pre-flight)
  - [Task 1 — Read the repository before running it](#task-1--read-the-repository-before-running-it)
  - [Task 2 — Render the base](#task-2--render-the-base)
  - [Task 3 — Compare dev and prod without touching the cluster](#task-3--compare-dev-and-prod-without-touching-the-cluster)
  - [Task 4 — Deploy dev safely](#task-4--deploy-dev-safely)
  - [Task 5 — Reach the application](#task-5--reach-the-application)
  - [Task 6 — Prove the ConfigMap hash/rollout chain](#task-6--prove-the-configmap-hashrollout-chain)
  - [Task 7 — Deploy staging and prod](#task-7--deploy-staging-and-prod)
  - [Task 8 — Inspect what the prod patch changed](#task-8--inspect-what-the-prod-patch-changed)
  - [Task 9 — Create my own QA overlay](#task-9--create-my-own-qa-overlay)
  - [Task 10 — Cleanup](#task-10--cleanup)
  - [Challenge Extension — namePrefix sandbox overlay](#challenge-extension--nameprefix-sandbox-overlay)
- [4. Analysis — Strategic Merge vs JSON 6902](#4-analysis--strategic-merge-vs-json-6902)
- [5. Reflection](#5-reflection)
- [6. Practical Report Evidence Checklist](#6-practical-report-evidence-checklist)
- [7. References](#7-references)

---

## Versions and Images

- Host operating system: Linux (the commands also work in WSL 2 with Docker access).
- Docker Engine: 24.0 or newer.
- kind: v0.32.0.
- kubectl: v1.32.2 (built-in Kustomize v5.5.0).
- Kubernetes node image: `kindest/node:v1.36.1`.
- Cluster name: `dso202-p5` (kubectl context `kind-dso202-p5`).
- Namespaces created by the overlays: `webapp-dev`, `webapp-staging`, `webapp-prod`, `webapp-qa` (and `webapp-sandbox` for the challenge).
- Application images: `nginx:1.30-alpine` (base, dev, qa) and `nginx:1.31-alpine` (staging and prod, set with the `images` transformer).

> kubectl v1.32 is older than the v1.36 server. kubectl may print a version-skew warning, but every command used in this practical works.

## Repository Structure

```text
practical-05/
├── README.md                      # this document (guide + practical report)
├── cluster/
│   └── kind-cluster.yaml          # 1 control-plane + 2 workers
├── evidence/                      # screenshots and saved command output
└── examples/
    └── webapp/
        ├── base/
        │   ├── deployment.yaml    # the only Deployment definition
        │   ├── service.yaml       # the only Service definition
        │   ├── index.html         # default page (replaced by every overlay)
        │   └── kustomization.yaml # resources, labels, configMapGenerator
        └── overlays/
            ├── dev/               # namespace, label, replicas=1, dev page
            ├── staging/           # replicas=2, image tag, strategic-merge annotation patch
            ├── prod/              # replicas=3, image tag, resource patch, commonAnnotations
            ├── qa/                # my overlay: replicas=2, JSON 6902 annotation patch
            └── sandbox/           # challenge: namePrefix "sandbox-"
```

### Lab topology

```text
Kind cluster (dso202-p5)
├── control-plane
├── worker-node-1
└── worker-node-2
Namespaces created by overlays
├── webapp-dev
├── webapp-staging
├── webapp-prod
└── webapp-qa
```

The workload is plain NGINX. The point of the lab is Kustomize, not the application code.

## Rebuild From an Empty Machine

Run these from the repository root (`practical-05/`).

```sh
docker info --format '{{.ServerVersion}}'
kind version
kubectl version --client

kind create cluster --config cluster/kind-cluster.yaml
kubectl config use-context kind-dso202-p5
kubectl get nodes -o wide

# render → diff → apply → verify for each environment
for env in dev staging prod qa; do
  kubectl kustomize examples/webapp/overlays/$env > /dev/null && echo "$env renders OK"
  kubectl diff -k examples/webapp/overlays/$env || true
  kubectl apply -k examples/webapp/overlays/$env
  kubectl rollout status deployment/webapp -n webapp-$env
done

kubectl get deploy -A -l app.kubernetes.io/name=webapp
```

To remove everything:

```sh
for env in dev staging prod qa; do kubectl delete -k examples/webapp/overlays/$env; done
kind delete cluster --name dso202-p5
```

---

## 1. Objective

By the end of this practical I should be able to:

1. Render a base and several overlays.
2. Deploy dev, staging and prod without copying the base Deployment or Service.
3. Use `namespace`, `labels`, `replicas`, `images`, generators and patches.
4. Observe how the ConfigMap hash suffix behaves.
5. Follow the render → diff → apply → verify workflow.
6. Find a Kustomize problem by reading the rendered output.

## 2. Environment

| Item | Value |
|---|---|
| Cluster | kind `dso202-p5` — `control-plane`, `worker-node-1`, `worker-node-2` |
| Kubernetes | v1.36.1 (`kindest/node:v1.36.1`) |
| kubectl / Kustomize | v1.32.2 / v5.5.0 (built in, used with `kubectl kustomize` and `-k`) |
| Workload | Deployment `webapp` + Service `webapp` + generated ConfigMap `web-content-<hash>` |
| Environments | `webapp-dev`, `webapp-staging`, `webapp-prod`, `webapp-qa` |

Create the cluster first:

```sh
kind create cluster --config cluster/kind-cluster.yaml
```

> **Screenshot evidence:** `evidence/00-kind-create-cluster.png`
>
> ![kind create cluster](evidence/00-kind-create-cluster.png)

---

## 3. Procedure and Observations

### Task 0 — Pre-flight

```sh
kubectl cluster-info
kubectl get nodes -o wide
kubectl version --client -o yaml
kubectl kustomize --help | head -5
kubectl apply --help | grep -- '-k, --kustomize'
```

Checks:

- [ ] The current context (`kind-dso202-p5`) reaches the Kind cluster (`cluster-info` shows `https://127.0.0.1:<port>`).
- [ ] All three nodes (`control-plane`, `worker-node-1`, `worker-node-2`) are `Ready`.
- [ ] `kubectl version --client -o yaml` shows a `kustomizeVersion` (v5.5.0), so `kubectl kustomize` and `kubectl apply -k` are supported.

> **Screenshot evidence:** `evidence/task0-cluster-info.png`
>
> ![Task 0 - cluster-info](evidence/task0-cluster-info.png)

> **Screenshot evidence:** `evidence/task0-get-nodes.png`
>
> ![Task 0 - get nodes](evidence/task0-get-nodes.png)

> **Screenshot evidence:** `evidence/task0-kubectl-version.png`
>
> ![Task 0 - kubectl version with kustomizeVersion](evidence/task0-kubectl-version.png)

---

### Task 1 — Read the repository before running it

```sh
tree examples/webapp
```

Output:

```text
examples/webapp
├── base
│   ├── deployment.yaml
│   ├── index.html
│   ├── kustomization.yaml
│   └── service.yaml
└── overlays
    ├── dev
    │   ├── index.html
    │   ├── kustomization.yaml
    │   └── namespace.yaml
    ├── prod
    │   ├── index.html
    │   ├── kustomization.yaml
    │   ├── namespace.yaml
    │   └── patch-resources.yaml
    ├── qa
    │   ├── index.html
    │   ├── kustomization.yaml
    │   ├── namespace.yaml
    │   └── patch-annotation.yaml
    ├── sandbox
    │   ├── index.html
    │   ├── kustomization.yaml
    │   └── namespace.yaml
    └── staging
        ├── index.html
        ├── kustomization.yaml
        ├── namespace.yaml
        └── patch-annotations.yaml
```

> **Screenshot evidence:** `evidence/task1-tree.png`
>
> ![Task 1 - repository tree](evidence/task1-tree.png)

#### Base files

`examples/webapp/base/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

# Added to every resource's metadata only (not to selectors).
labels:
  - pairs:
      app.kubernetes.io/part-of: kustomize-practical
      app.kubernetes.io/managed-by: kustomize

# Generates ConfigMap "web-content-<hash>" from index.html and rewrites
# every reference to "web-content" (the Deployment volume) to that name.
configMapGenerator:
  - name: web-content
    files:
      - index.html
```

`examples/webapp/base/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app.kubernetes.io/name: webapp
  annotations:
    training.example.com/description: "NGINX demo workload for the Kustomize practical"
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: webapp
  template:
    metadata:
      labels:
        app.kubernetes.io/name: webapp
    spec:
      containers:
        - name: nginx
          image: nginx:1.30-alpine
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 2
            periodSeconds: 5
          volumeMounts:
            - name: web-content
              mountPath: /usr/share/nginx/html
              readOnly: true
      volumes:
        - name: web-content
          configMap:
            # Logical name. Kustomize rewrites this to the hashed generated name.
            name: web-content
```

`examples/webapp/base/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
  labels:
    app.kubernetes.io/name: webapp
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: webapp
  ports:
    - name: http
      port: 80
      targetPort: http
```

#### Overlay files (dev, staging, prod)

`examples/webapp/overlays/dev/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: webapp-dev

resources:
  - ../../base
  - namespace.yaml

# environment label on metadata and on the Pod template (not selectors).
labels:
  - pairs:
      environment: dev
    includeTemplates: true

replicas:
  - name: webapp
    count: 1

# Replace the base page with the dev page. Same logical name, new content,
# new hash.
configMapGenerator:
  - name: web-content
    behavior: replace
    files:
      - index.html
```

`examples/webapp/overlays/staging/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: webapp-staging

resources:
  - ../../base
  - namespace.yaml

labels:
  - pairs:
      environment: staging
    includeTemplates: true

replicas:
  - name: webapp
    count: 2

images:
  - name: nginx
    newTag: 1.31-alpine

configMapGenerator:
  - name: web-content
    behavior: replace
    files:
      - index.html

patches:
  - path: patch-annotations.yaml
```

`examples/webapp/overlays/staging/patch-annotations.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  annotations:
    training.example.com/change-window: "weekdays-09:00-17:00"
```

`examples/webapp/overlays/prod/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: webapp-prod

resources:
  - ../../base
  - namespace.yaml

labels:
  - pairs:
      environment: prod
    includeTemplates: true

commonAnnotations:
  training.example.com/tier: production

replicas:
  - name: webapp
    count: 3

images:
  - name: nginx
    newTag: 1.31-alpine

configMapGenerator:
  - name: web-content
    behavior: replace
    files:
      - index.html

patches:
  - path: patch-resources.yaml
```

`examples/webapp/overlays/prod/patch-resources.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  annotations:
    training.example.com/change-window: "sunday-02:00-04:00"
spec:
  template:
    spec:
      containers:
        - name: nginx
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 256Mi
```

Each overlay also has a `namespace.yaml` (a `Namespace` named `webapp-<env>`) and its own `index.html`.

#### Questions

**Which files exist only once for all environments?**
`base/deployment.yaml`, `base/service.yaml` and `base/kustomization.yaml`. There is one Deployment definition and one Service definition. No overlay contains a copy of either. `base/index.html` is also stored once, as the default page, and each overlay replaces it with `behavior: replace`.

**Which values differ between environments?**

| Value | dev | staging | prod |
|---|---|---|---|
| Namespace | `webapp-dev` | `webapp-staging` | `webapp-prod` |
| `environment` label | `dev` | `staging` | `prod` |
| Replicas | 1 | 2 | 3 |
| Image | `nginx:1.30-alpine` (base) | `nginx:1.31-alpine` | `nginx:1.31-alpine` |
| Web page (`index.html`) | DEV page | STAGING page | PROD page |
| Resources | base (50m/64Mi → 250m/128Mi) | base | requests 100m/128Mi, limits 250m/256Mi |
| Extra annotations | — | `change-window: weekdays` | `change-window: sunday`, `tier: production` |

**Where are those differences represented?**
Only in `overlays/<env>/`. The `kustomization.yaml` fields `namespace`, `labels`, `replicas`, `images`, `commonAnnotations` and `configMapGenerator` (with `behavior: replace`) hold most of them. The patch files `staging/patch-annotations.yaml` and `prod/patch-resources.yaml` hold the rest.

---

### Task 2 — Render the base

```sh
kubectl kustomize examples/webapp/base
```

The base is **not** applied. It is only rendered.

```sh
kubectl kustomize examples/webapp/base | grep '^kind:'
kubectl kustomize examples/webapp/base | grep 'name: web-content'
```

Output:

```text
kind: ConfigMap
kind: Service
kind: Deployment
```

```text
  name: web-content-gmg9hgkk25
          name: web-content
          name: web-content-gmg9hgkk25
        name: web-content
```

Findings:

| Item | Where it appears in the rendered output |
|---|---|
| Deployment | `kind: Deployment`, `name: webapp` |
| Service | `kind: Service`, `name: webapp` |
| Generated ConfigMap | `kind: ConfigMap`, `name: web-content-gmg9hgkk25` |
| Hash suffix | `gmg9hgkk25` |
| Rewritten reference | `spec.template.spec.volumes[0].configMap.name: web-content-gmg9hgkk25` (the `name: web-content` lines left over are the volume name and the volumeMount name, which are not ConfigMap references) |

> **Screenshot evidence:** `evidence/task2-render-base.png`
>
> ![Task 2 - render base](evidence/task2-render-base.png)

> **Screenshot evidence:** `evidence/task2-grep-kind-and-configmap.png`
>
> ![Task 2 - grep kind and web-content](evidence/task2-grep-kind-and-configmap.png)

#### Checkpoint — why is the ConfigMap name not exactly `web-content`?

`configMapGenerator` builds the ConfigMap from `index.html` and adds a suffix to its name. The suffix is a hash of the ConfigMap's content. Kustomize then updates every reference to the logical name `web-content` (here, the Deployment's `configMap` volume) to the hashed name.

The ConfigMap name is therefore tied to its content. If the content changes, the name changes, the Deployment's Pod template changes, and Kubernetes rolls out new Pods. Without the hash, editing a ConfigMap in place would not restart Pods, and the running Pods would keep serving the old configuration. The old ConfigMap also stays in the cluster until it is removed, so rolling back to the previous ReplicaSet still has its matching configuration.

---

### Task 3 — Compare dev and prod without touching the cluster

```sh
kubectl kustomize examples/webapp/overlays/dev  > /tmp/webapp-dev.yaml
kubectl kustomize examples/webapp/overlays/prod > /tmp/webapp-prod.yaml
diff -u /tmp/webapp-dev.yaml /tmp/webapp-prod.yaml || true

# keep a copy as evidence
cp /tmp/webapp-dev.yaml /tmp/webapp-prod.yaml evidence/
diff -u /tmp/webapp-dev.yaml /tmp/webapp-prod.yaml > evidence/task3-dev-vs-prod.diff || true
```

Diff excerpt:

```diff
 kind: Namespace
 metadata:
+  annotations:
+    training.example.com/tier: production
   labels:
-    environment: dev
-  name: webapp-dev
+    environment: prod
+  name: webapp-prod
 ...
-        <h1>DEV - development environment</h1>
-        <p>Namespace: webapp-dev | Replicas: 1</p>
+        <h1>PROD - production environment</h1>
+        <p>Namespace: webapp-prod | Replicas: 3</p>
 ...
-  name: web-content-fb4bcc97t6
-  namespace: webapp-dev
+  name: web-content-tg96t2cg8b
+  namespace: webapp-prod
 ...
 kind: Deployment
 metadata:
   annotations:
+    training.example.com/change-window: sunday-02:00-04:00
     training.example.com/description: NGINX demo workload for the Kustomize practical
+    training.example.com/tier: production
 ...
-  replicas: 1
+  replicas: 3
 ...
-      - image: nginx:1.30-alpine
+      - image: nginx:1.31-alpine
 ...
           limits:
             cpu: 250m
-            memory: 128Mi
+            memory: 256Mi
           requests:
-            cpu: 50m
-            memory: 64Mi
+            cpu: 100m
+            memory: 128Mi
 ...
       - configMap:
-          name: web-content-fb4bcc97t6
+          name: web-content-tg96t2cg8b
```

> **Screenshot evidence:** `evidence/task3-dev-prod-diff.png`
>
> ![Task 3 - dev vs prod diff](evidence/task3-dev-prod-diff.png)

#### Environment differences found (more than five)

| # | Category | dev | prod | Source in prod overlay |
|---|---|---|---|---|
| 1 | Namespace | `webapp-dev` | `webapp-prod` | `namespace:` + `namespace.yaml` |
| 2 | Environment label | `environment: dev` | `environment: prod` | `labels:` (also on Pod template) |
| 3 | Replica count | 1 | 3 | `replicas:` |
| 4 | Image tag | `nginx:1.30-alpine` | `nginx:1.31-alpine` | `images:` → `newTag` |
| 5 | Resource requests/limits | 50m/64Mi req, 128Mi mem limit | 100m/128Mi req, 256Mi mem limit | `patch-resources.yaml` |
| 6 | Generated web content | DEV page, `web-content-fb4bcc97t6` | PROD page, `web-content-tg96t2cg8b` | `configMapGenerator` `behavior: replace` |
| 7 | Annotations | description only | + `change-window`, + `tier: production` (also on the Pod template) | `patch-resources.yaml`, `commonAnnotations` |

**Principle:** I can explain every production difference from about 40 lines of overlay (`kustomization.yaml` + `patch-resources.yaml`). I do not have to review a second full copy of the Deployment manifest.

---

### Task 4 — Deploy dev safely

**1. Inspect (render)**

```sh
kubectl kustomize examples/webapp/overlays/dev
```

Rendered dev output excerpt (Deployment):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    training.example.com/description: NGINX demo workload for the Kustomize practical
  labels:
    app.kubernetes.io/managed-by: kustomize
    app.kubernetes.io/name: webapp
    app.kubernetes.io/part-of: kustomize-practical
    environment: dev
  name: webapp
  namespace: webapp-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: webapp
  template:
    metadata:
      labels:
        app.kubernetes.io/name: webapp
        environment: dev
    spec:
      containers:
      - image: nginx:1.30-alpine
        name: nginx
        ...
      volumes:
      - configMap:
          name: web-content-fb4bcc97t6
        name: web-content
```

> **Screenshot evidence:** `evidence/task4-render-dev.png`
>
> ![Task 4 - render dev](evidence/task4-render-dev.png)

**2. Diff**

```sh
kubectl diff -k examples/webapp/overlays/dev || true
```

Every object is new at this point, so the diff shows each one (Namespace, ConfigMap, Service, Deployment) as fully added (`+`). `kubectl diff` exits with code 1 when there are differences, and `|| true` stops that from being treated as an error.

> **Screenshot evidence:** `evidence/task4-diff-dev.png`
>
> ![Task 4 - diff dev](evidence/task4-diff-dev.png)

**3. Apply**

```sh
kubectl apply -k examples/webapp/overlays/dev
```

Expected:

```text
namespace/webapp-dev created
configmap/web-content-fb4bcc97t6 created
service/webapp created
deployment.apps/webapp created
```

> **Screenshot evidence:** `evidence/task4-apply-dev.png`
>
> ![Task 4 - apply dev](evidence/task4-apply-dev.png)

**4. Verify**

```sh
kubectl get all -n webapp-dev
kubectl get configmap -n webapp-dev
kubectl rollout status deployment/webapp -n webapp-dev

kubectl get all -n webapp-dev -o wide > evidence/task4-get-all-webapp-dev.txt
```

> **Screenshot evidence:** `evidence/task4-get-all-dev.png`
>
> ![Task 4 - get all -n webapp-dev](evidence/task4-get-all-dev.png)

> **Screenshot evidence:** `evidence/task4-configmap-rollout-dev.png`
>
> ![Task 4 - configmap and rollout status](evidence/task4-configmap-rollout-dev.png)

**Observation:** The Deployment is still called `webapp`, the same name as in the base. It has no `dev-` prefix. The environments are kept apart by the namespace (`webapp-dev`), not by resource names. The ConfigMap in the cluster is `web-content-fb4bcc97t6`, which matches the rendered output exactly.

---

### Task 5 — Reach the application

Terminal 1:

```sh
kubectl port-forward -n webapp-dev service/webapp 8080:80
```

Terminal 2:

```sh
curl http://127.0.0.1:8080
```

Expected response:

```html
<!DOCTYPE html>
<html>
  <head><title>webapp - dev</title></head>
  <body style="background:#e8f5e9">
    <h1>DEV - development environment</h1>
    <p>Namespace: webapp-dev | Replicas: 1</p>
  </body>
</html>
```

The response identifies the **development** environment. I stopped the port-forward with `Ctrl+C` afterwards.

> **Screenshot evidence:** `evidence/task5-port-forward.png`
>
> ![Task 5 - port-forward](evidence/task5-port-forward.png)

> **Screenshot evidence:** `evidence/task5-curl-dev.png`
>
> ![Task 5 - curl dev page](evidence/task5-curl-dev.png)

---

### Task 6 — Prove the ConfigMap hash/rollout chain

**Before the change**

```sh
kubectl get configmap -n webapp-dev
kubectl get pods -n webapp-dev -o wide

kubectl get configmap -n webapp-dev > evidence/task6-configmap-before.txt
kubectl get pods -n webapp-dev -o wide > evidence/task6-pods-before.txt
```

> **Screenshot evidence:** `evidence/task6-before.png`
>
> ![Task 6 - configmap and pods before](evidence/task6-before.png)

**Change the page**

Edit `examples/webapp/overlays/dev/index.html` and change the heading to:

```html
<h1>DEV v2 — configuration changed</h1>
```

(or with one command:)

```sh
sed -i 's#<h1>DEV - development environment</h1>#<h1>DEV v2 — configuration changed</h1>#' \
  examples/webapp/overlays/dev/index.html
grep '<h1>' examples/webapp/overlays/dev/index.html
```

**Render again before applying**

```sh
kubectl kustomize examples/webapp/overlays/dev | grep 'name: web-content'
```

Output:

```text
  name: web-content-6h85f4dhfc
          name: web-content
          name: web-content-6h85f4dhfc
        name: web-content
```

The render shows the new hash before anything reaches the cluster: `fb4bcc97t6` → `6h85f4dhfc`.

> **Screenshot evidence:** `evidence/task6-render-new-hash.png`
>
> ![Task 6 - render shows new hash](evidence/task6-render-new-hash.png)

**Apply and watch the rollout**

```sh
kubectl diff -k examples/webapp/overlays/dev || true    # shows the volume configMap name change
kubectl apply -k examples/webapp/overlays/dev
kubectl rollout status deployment/webapp -n webapp-dev
```

Expected apply output:

```text
namespace/webapp-dev unchanged
configmap/web-content-6h85f4dhfc created
service/webapp unchanged
deployment.apps/webapp configured
```

> **Screenshot evidence:** `evidence/task6-apply-rollout.png`
>
> ![Task 6 - apply and rollout](evidence/task6-apply-rollout.png)

**After the change**

```sh
kubectl get configmap -n webapp-dev
kubectl get pods -n webapp-dev -o wide
kubectl rollout history deployment/webapp -n webapp-dev

kubectl get configmap -n webapp-dev > evidence/task6-configmap-after.txt
kubectl get pods -n webapp-dev -o wide > evidence/task6-pods-after.txt
```

> **Screenshot evidence:** `evidence/task6-after.png`
>
> ![Task 6 - configmap and pods after](evidence/task6-after.png)

Optional check that the new page is being served:

```sh
kubectl port-forward -n webapp-dev service/webapp 8080:80 &
sleep 2; curl -s http://127.0.0.1:8080 | grep '<h1>'; kill %1
```

> **Screenshot evidence:** `evidence/task6-curl-v2.png`
>
> ![Task 6 - curl shows DEV v2](evidence/task6-curl-v2.png)

**Recorded evidence** (fill in from the screenshots)

| | Before | After |
|---|---|---|
| Generated ConfigMap | `web-content-fb4bcc97t6` | `web-content-6h85f4dhfc` |
| Pod name(s) | `webapp-<rs-hash-1>-<pod-id>` ← *fill in* | `webapp-<rs-hash-2>-<pod-id>` ← *fill in* |
| ReplicaSet hash in the Pod name | *fill in* | *fill in* (different) |

`kubectl get configmap` now lists **both** ConfigMaps. `kubectl apply -k` does not delete the old one, because nothing is pruned. The old Pod was terminated and a new Pod with a new ReplicaSet hash in its name replaced it.

#### The chain, in my own words

```text
file content changed
→ generated ConfigMap content changed
→ generated ConfigMap name hash changed
→ Deployment reference changed
→ Deployment pod template changed
→ rollout occurred
```

1. **File content changed.** I edited `overlays/dev/index.html`. That file is the input to `configMapGenerator`.
2. **Generated ConfigMap content changed.** The `data.index.html` key of the generated ConfigMap now holds the new HTML.
3. **Generated ConfigMap name hash changed.** Kustomize computes the suffix from the ConfigMap content. New content gave a new suffix: `web-content-fb4bcc97t6` became `web-content-6h85f4dhfc`.
4. **Deployment reference changed.** Kustomize knows that `spec.template.spec.volumes[].configMap.name` refers to a ConfigMap, so it rewrote that field to the new hashed name.
5. **Deployment Pod template changed.** That field is inside `spec.template`. A change there gives the template a new hash, which is what the Deployment controller compares.
6. **Rollout occurred.** The Deployment controller created a new ReplicaSet for the new template, started a Pod that mounts the new ConfigMap, waited for it to be Ready, and then scaled the old ReplicaSet down to zero.

If the ConfigMap had a fixed name and I edited it in place, the Pod template would not change and no rollout would happen. The running Pod would keep serving old content, or would pick it up later with no controlled rollout. The hash turns a configuration change into a normal rolling update that I can watch and roll back.

---

### Task 7 — Deploy staging and prod

```sh
kubectl diff -k examples/webapp/overlays/staging || true
kubectl apply -k examples/webapp/overlays/staging

kubectl diff -k examples/webapp/overlays/prod || true
kubectl apply -k examples/webapp/overlays/prod

kubectl rollout status deployment/webapp -n webapp-staging
kubectl rollout status deployment/webapp -n webapp-prod
```

> **Screenshot evidence:** `evidence/task7-diff-apply-staging.png`
>
> ![Task 7 - diff and apply staging](evidence/task7-diff-apply-staging.png)

> **Screenshot evidence:** `evidence/task7-diff-apply-prod.png`
>
> ![Task 7 - diff and apply prod](evidence/task7-diff-apply-prod.png)

Verify all namespaces:

```sh
kubectl get deploy -A -l app.kubernetes.io/name=webapp
kubectl get pods -A -l app.kubernetes.io/name=webapp -o wide

kubectl get deploy -A -l app.kubernetes.io/name=webapp > evidence/task7-deployments.txt
kubectl get pods -A -l app.kubernetes.io/name=webapp -o wide > evidence/task7-pods.txt
```

> **Screenshot evidence:** `evidence/task7-get-deploy-all.png`
>
> ![Task 7 - deployments across namespaces](evidence/task7-get-deploy-all.png)

> **Screenshot evidence:** `evidence/task7-get-pods-all.png`
>
> ![Task 7 - pods across namespaces](evidence/task7-get-pods-all.png)

**Replica counts by environment**

| Environment | Namespace | Desired replicas | Ready (from `kubectl get deploy`) | Image | Nodes used |
|---|---|---|---|---|---|
| dev | `webapp-dev` | 1 | *1/1* | `nginx:1.30-alpine` | *fill in* |
| staging | `webapp-staging` | 2 | *2/2* | `nginx:1.31-alpine` | *fill in* |
| prod | `webapp-prod` | 3 | *3/3* | `nginx:1.31-alpine` | *fill in* |

The label selector `app.kubernetes.io/name=webapp` finds the Deployment in every namespace because the base sets that label and every overlay inherits it. The overlays only add `environment=<env>` beside it. `kubectl get pods -o wide` shows the scheduler spreading Pods across `worker-node-1` and `worker-node-2`.

---

### Task 8 — Inspect what the prod patch changed

```sh
cat examples/webapp/overlays/prod/patch-resources.yaml
kubectl kustomize examples/webapp/overlays/prod | sed -n '/resources:/,/volumeMounts:/p'
```

The patch (see Task 1) sets only `requests.cpu`, `requests.memory` and `limits.memory`.

Rendered container resources for prod:

```yaml
        resources:
          limits:
            cpu: 250m        # from the base (not in the patch)
            memory: 256Mi    # from the prod patch (base was 128Mi)
          requests:
            cpu: 100m        # from the prod patch (base was 50m)
            memory: 128Mi    # from the prod patch (base was 64Mi)
```

> **Screenshot evidence:** `evidence/task8-patch-file.png`
>
> ![Task 8 - patch-resources.yaml](evidence/task8-patch-file.png)

> **Screenshot evidence:** `evidence/task8-rendered-resources.png`
>
> ![Task 8 - rendered prod resources](evidence/task8-rendered-resources.png)

Also confirm on the live object:

```sh
kubectl get deploy webapp -n webapp-prod -o jsonpath='{.spec.template.spec.containers[0].resources}{"\n"}'
```

> **Screenshot evidence:** `evidence/task8-live-resources.png`
>
> ![Task 8 - live prod resources](evidence/task8-live-resources.png)

#### Questions

**Were the base resource values deleted entirely or merged?**
They were **merged**. A strategic merge patch matches the container by its `name` (`nginx`) and merges maps key by key. The three keys in the patch replaced the base values. `limits.cpu: 250m` was not in the patch, so it stayed as it was in the base. Nothing else in the container (image, ports, probe, volumeMounts) was touched.

**Which environment owns the production-specific resource policy?**
The **prod overlay** owns it, in `overlays/prod/patch-resources.yaml`. The base keeps safe defaults. dev and staging use those defaults and never see the production values.

**Why is a patch better than copying `deployment.yaml` into `prod/`?**

- **One source of truth.** A fix to the base (a probe, port, label or security setting) reaches prod automatically. With a copy, prod slowly drifts away from the other environments.
- **Reviewability.** The patch is about 15 lines and contains only what makes prod different. A reviewer does not have to diff a full 50-line copy to find the change.
- **Clear intent.** The patch file name and contents say exactly what prod changes and why.
- **Less duplication and fewer mistakes.** With a copy, every change has to be made in N places, and one will eventually be missed.

---

### Task 9 — Create my own QA overlay

Requirements and how they are met:

| Requirement | How it is met |
|---|---|
| Namespace `webapp-qa` | `namespace: webapp-qa` + `namespace.yaml` |
| Replicas: 2 | `replicas:` → `count: 2` |
| Environment label `qa` | `labels:` → `environment: qa` (`includeTemplates: true`) |
| Unique `index.html` | `overlays/qa/index.html` with `behavior: replace` |
| Same base | `resources: - ../../base` |
| Annotation `training.example.com/owner: qa-team` via JSON 6902 | `patch-annotation.yaml` + `patches[].target` |
| No copied `deployment.yaml` / `service.yaml` | Only the 4 files below exist in `qa/` |

```text
overlays/qa/
├── index.html
├── kustomization.yaml
├── namespace.yaml
└── patch-annotation.yaml
```

`examples/webapp/overlays/qa/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: webapp-qa

resources:
  - ../../base
  - namespace.yaml

labels:
  - pairs:
      environment: qa
    includeTemplates: true

replicas:
  - name: webapp
    count: 2

configMapGenerator:
  - name: web-content
    behavior: replace
    files:
      - index.html

patches:
  - path: patch-annotation.yaml
    target:
      group: apps
      version: v1
      kind: Deployment
      name: webapp
```

`examples/webapp/overlays/qa/patch-annotation.yaml`

```yaml
# JSON 6902 patch: an explicit list of operations on explicit paths.
# The "/" inside the annotation key must be escaped as "~1"
# (RFC 6901), otherwise it is read as a path separator.
- op: add
  path: /metadata/annotations/training.example.com~1owner
  value: qa-team
```

`examples/webapp/overlays/qa/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: webapp-qa
```

`examples/webapp/overlays/qa/index.html`

```html
<!DOCTYPE html>
<html>
  <head><title>webapp - qa</title></head>
  <body style="background:#f3e5f5">
    <h1>QA - quality assurance environment</h1>
    <p>Namespace: webapp-qa | Replicas: 2 | Owner: qa-team</p>
  </body>
</html>
```

> **Screenshot evidence:** `evidence/task9-qa-files.png`
>
> ![Task 9 - QA overlay files](evidence/task9-qa-files.png)

**Render and confirm the annotation before going further**

```sh
kubectl kustomize examples/webapp/overlays/qa
kubectl kustomize examples/webapp/overlays/qa | sed -n '/kind: Deployment/,/^spec:/p'
```

Output:

```yaml
kind: Deployment
metadata:
  annotations:
    training.example.com/description: NGINX demo workload for the Kustomize practical
    training.example.com/owner: qa-team
  labels:
    app.kubernetes.io/managed-by: kustomize
    app.kubernetes.io/name: webapp
    app.kubernetes.io/part-of: kustomize-practical
    environment: qa
  name: webapp
  namespace: webapp-qa
spec:
```

The annotation `training.example.com/owner: qa-team` is present. The base `description` annotation is also still there, which shows the patch added a key and did not replace the whole map.

> **Screenshot evidence:** `evidence/task9-render-qa-annotation.png`
>
> ![Task 9 - rendered QA annotation](evidence/task9-render-qa-annotation.png)

**Diff, apply, verify**

```sh
kubectl diff -k examples/webapp/overlays/qa || true
kubectl apply -k examples/webapp/overlays/qa
kubectl rollout status deployment/webapp -n webapp-qa
kubectl get deployment webapp -n webapp-qa -o yaml

kubectl get deployment webapp -n webapp-qa -o yaml > evidence/task9-qa-deployment.yaml
kubectl get deployment webapp -n webapp-qa -o jsonpath='{.metadata.annotations}{"\n"}{.spec.replicas}{"\n"}'
```

> **Screenshot evidence:** `evidence/task9-diff-apply-qa.png`
>
> ![Task 9 - diff and apply qa](evidence/task9-diff-apply-qa.png)

> **Screenshot evidence:** `evidence/task9-get-deploy-qa-yaml.png`
>
> ![Task 9 - deployment yaml in webapp-qa](evidence/task9-get-deploy-qa-yaml.png)

Verify with `curl` that the QA page is served:

```sh
kubectl port-forward -n webapp-qa service/webapp 8081:80 &
sleep 2; curl -s http://127.0.0.1:8081 | grep '<h1>'; kill %1
```

> **Screenshot evidence:** `evidence/task9-curl-qa.png`
>
> ![Task 9 - curl qa page](evidence/task9-curl-qa.png)

---

### Task 10 — Cleanup

Save the final state before deleting anything:

```sh
kubectl get all -A -l app.kubernetes.io/name=webapp -o wide > evidence/final-state-all.txt
kubectl get configmap -A | grep web-content > evidence/final-state-configmaps.txt
```

Delete every environment:

```sh
kubectl delete -k examples/webapp/overlays/dev
kubectl delete -k examples/webapp/overlays/staging
kubectl delete -k examples/webapp/overlays/prod
kubectl delete -k examples/webapp/overlays/qa
```

> **Screenshot evidence:** `evidence/task10-delete.png`
>
> ![Task 10 - delete overlays](evidence/task10-delete.png)

Verify:

```sh
kubectl get ns | grep 'webapp-' || true
```

No output (or namespaces still in `Terminating` for a few seconds) means cleanup worked. Deleting an overlay deletes its `Namespace` object, and deleting a namespace removes everything inside it. That includes the old `web-content-fb4bcc97t6` ConfigMap in dev, which `delete -k` would not have removed on its own because it is no longer in the rendered output.

> **Screenshot evidence:** `evidence/task10-verify-ns.png`
>
> ![Task 10 - no webapp namespaces](evidence/task10-verify-ns.png)

Optionally remove the cluster:

```sh
kind delete cluster --name dso202-p5
kind get clusters
```

---

### Challenge Extension — namePrefix sandbox overlay

A new overlay, `overlays/sandbox/`, adds `namePrefix: sandbox-` and does **not** copy any base resource.

`examples/webapp/overlays/sandbox/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: webapp-sandbox

# Only this overlay adds a prefix. Kustomize renames the resources AND
# updates every name reference that points to them.
namePrefix: sandbox-

resources:
  - ../../base
  - namespace.yaml

labels:
  - pairs:
      environment: sandbox
    includeTemplates: true

configMapGenerator:
  - name: web-content
    behavior: replace
    files:
      - index.html
```

#### Prediction (written before rendering)

| Field | Before | Predicted after | Reason |
|---|---|---|---|
| Deployment `metadata.name` | `webapp` | `sandbox-webapp` | namePrefix applies to all namespaced resources |
| Service `metadata.name` | `webapp` | `sandbox-webapp` | same |
| ConfigMap `metadata.name` | `web-content-<hash>` | `sandbox-web-content-<hash>` | the prefix goes before the name, and the hash suffix stays at the end |
| Deployment `volumes[].configMap.name` | `web-content-<hash>` | `sandbox-web-content-<hash>` | **reference-aware**: Kustomize knows this field points to a ConfigMap |
| Namespace `webapp-sandbox` | `webapp-sandbox` | **unchanged** | the `namespace:` field and Namespace objects are not prefixed |
| Container name `nginx`, volume name `web-content`, port name `http` | — | **unchanged** | these are not names of Kubernetes objects |
| Labels / Service selector `app.kubernetes.io/name: webapp` | — | **unchanged** | these are label values, not object names. The Service still selects the Pods |

#### Render and compare

```sh
kubectl kustomize examples/webapp/overlays/sandbox
kubectl kustomize examples/webapp/overlays/sandbox | grep -nE '^kind:|^  name:|configMap:|          name:'
```

Output (key lines):

```text
kind: Namespace
  name: webapp-sandbox
kind: ConfigMap
  name: sandbox-web-content-d4dk9d924k
kind: Service
  name: sandbox-webapp
    app.kubernetes.io/name: webapp          # selector unchanged
kind: Deployment
  name: sandbox-webapp
      - configMap:
          name: sandbox-web-content-d4dk9d924k   # reference rewritten
        name: web-content                    # volume name unchanged
```

> **Screenshot evidence:** `evidence/challenge-render-sandbox.png`
>
> ![Challenge - rendered sandbox](evidence/challenge-render-sandbox.png)

**Result:** The output matched every prediction. Kustomize prefixed the three namespaced objects and rewrote the one cross-object reference (Deployment → ConfigMap). It left the Namespace, label values, the selector, and the container, volume and port names alone.

**What this teaches:** Doing this with `sed` (find/replace `webapp` → `sandbox-webapp`) would also have changed the label values and selector, and could have missed the hashed ConfigMap reference. Kustomize changes the name and every field that refers to that name together, so references stay correct. That is what "reference-aware transformation" means.

Optional deploy and cleanup:

```sh
kubectl apply -k examples/webapp/overlays/sandbox
kubectl get all -n webapp-sandbox
kubectl delete -k examples/webapp/overlays/sandbox
```

> **Screenshot evidence:** `evidence/challenge-apply-sandbox.png`
>
> ![Challenge - sandbox deployed](evidence/challenge-apply-sandbox.png)

---

## 4. Analysis — Strategic Merge vs JSON 6902

| | Strategic merge patch | JSON 6902 patch |
|---|---|---|
| Form | A partial Kubernetes object (apiVersion/kind/metadata.name + the fields to change) | A list of operations: `add`, `remove`, `replace`, `move`, `copy`, `test` |
| Targeting | By the `kind` + `name` inside the patch | By `patches[].target` (group/version/kind/name, or label selectors) |
| Lists | Understands Kubernetes list keys (for example, containers merged by `name`) | Works on array **indexes** (`/spec/template/spec/containers/0/...`) |
| Maps | Merges key by key | Changes exactly one path. `add` on an existing map **replaces the whole map** |
| Special characters | None | `/` in a key → `~1`, `~` → `~0` |
| Deleting a field | Needs `$patch: delete` or a `null` value | `op: remove` |
| Used in this practical | `staging/patch-annotations.yaml`, `prod/patch-resources.yaml` | `qa/patch-annotation.yaml` |
| Best for | "Make this object look like this": resources, env vars, adding sidecars | Precise, surgical edits, removals, fields with no merge key, CRDs without strategic-merge schema |

In prod, the strategic merge patch was the natural choice. I wanted three resource values changed and the rest of the container left alone, and the merge matched the container by name without me needing to know its index. In QA, the JSON 6902 patch made me state the exact path of the one key I was adding. That is more precise, but also less forgiving, as the reflection below shows.

Other observations:

- **Rendering is the safety net.** Every task rendered first, then ran `kubectl diff`, and only then applied. Each environment difference, and each mistake, was visible in plain YAML before it reached the cluster.
- **Generated names plus apply leave old ConfigMaps behind.** `kubectl apply -k` does not prune, so old `web-content-*` ConfigMaps build up over time. That is useful for rollbacks, but they need cleaning up (namespace deletion did it here).
- **Labels vs selectors.** I used `labels:` with `includeTemplates: true` instead of the deprecated `commonLabels`. The `environment` label reaches Pods, but it is not added to the immutable `spec.selector`. Adding a label to the selector of a Deployment that already exists would make the apply fail.

## 5. Reflection

**One Kustomize mistake I made, and how the rendered output exposed it:**

On my first attempt at the QA JSON 6902 patch, I wrote the path the way the annotation looks in YAML:

```yaml
- op: add
  path: /metadata/annotations/training.example.com/owner
  value: qa-team
```

`kubectl kustomize examples/webapp/overlays/qa` failed immediately:

```text
Error: add operation does not apply: doc is missing path: "/metadata/annotations/training.example.com/owner": missing value
```

The error showed that the `/` in `training.example.com/owner` was being read as a path separator. Kustomize was looking for an object called `training.example.com` with a child called `owner`. JSON Pointer (RFC 6901) needs `/` to be escaped as `~1`.

My second attempt removed the escaping problem by adding the whole map:

```yaml
- op: add
  path: /metadata/annotations
  value:
    training.example.com/owner: qa-team
```

This rendered with no error, but the rendered Deployment showed only `owner: qa-team`. The base annotation `training.example.com/description` had **disappeared**, because `add` on an existing path replaces the whole map. Without a render I would not have noticed that a base annotation had been silently deleted.

The final version, `path: /metadata/annotations/training.example.com~1owner`, rendered both annotations. I only applied it after checking that.

Both mistakes can be reproduced safely in a scratch copy:

```sh
cp -r examples/webapp /tmp/webapp-mistake
sed -i 's#example.com~1owner#example.com/owner#' /tmp/webapp-mistake/overlays/qa/patch-annotation.yaml
kubectl kustomize /tmp/webapp-mistake/overlays/qa      # shows the "missing path" error
rm -rf /tmp/webapp-mistake
```

> **Screenshot evidence:** `evidence/reflection-json6902-error.png`
>
> ![Reflection - JSON 6902 path error](evidence/reflection-json6902-error.png)

**What I learned**

- Render before every apply. `kubectl kustomize` is free and does not touch the cluster.
- JSON 6902 is precise but literal. It does exactly what the path says, including overwriting a whole map.
- The ConfigMap hash is not cosmetic. It is how a configuration change becomes a rollout.
- An overlay should read as a short list of differences. If an overlay starts to look like a copy of the base, something is in the wrong place.

## 6. Practical Report Evidence Checklist

| Required evidence | Where in this report | Screenshot / file |
|---|---|---|
| Repository tree | [Task 1](#task-1--read-the-repository-before-running-it) | `evidence/task1-tree.png` |
| Rendered dev output excerpt | [Task 4](#task-4--deploy-dev-safely) | `evidence/task4-render-dev.png`, `evidence/webapp-dev.yaml` |
| dev/prod diff excerpt | [Task 3](#task-3--compare-dev-and-prod-without-touching-the-cluster) | `evidence/task3-dev-prod-diff.png`, `evidence/task3-dev-vs-prod.diff` |
| `kubectl get all -n webapp-dev` | [Task 4](#task-4--deploy-dev-safely) | `evidence/task4-get-all-dev.png`, `evidence/task4-get-all-webapp-dev.txt` |
| ConfigMap names before and after the content change | [Task 6](#task-6--prove-the-configmap-hashrollout-chain) | `evidence/task6-before.png`, `evidence/task6-after.png` |
| Pod names before and after rollout | [Task 6](#task-6--prove-the-configmap-hashrollout-chain) | `evidence/task6-pods-before.txt`, `evidence/task6-pods-after.txt` |
| Hash → rollout chain explanation | [Task 6](#the-chain-in-my-own-words) | — |
| Replica counts by environment | [Task 7](#task-7--deploy-staging-and-prod) | `evidence/task7-get-deploy-all.png` |
| Prod patch questions | [Task 8](#task-8--inspect-what-the-prod-patch-changed) | `evidence/task8-rendered-resources.png` |
| QA overlay files | [Task 9](#task-9--create-my-own-qa-overlay) | `evidence/task9-qa-files.png` |
| Strategic merge vs JSON 6902 | [Section 4](#4-analysis--strategic-merge-vs-json-6902) | — |
| Reflection: one mistake + how the render exposed it | [Section 5](#5-reflection) | `evidence/reflection-json6902-error.png` |
| Cleanup | [Task 10](#task-10--cleanup) | `evidence/task10-verify-ns.png` |
| Challenge: namePrefix prediction vs output | [Challenge](#challenge-extension--nameprefix-sandbox-overlay) | `evidence/challenge-render-sandbox.png` |

## 7. References

- Kubernetes documentation — *Declarative Management of Kubernetes Objects Using Kustomize*: <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/>
- Kustomize reference — `configMapGenerator`, `labels`, `replicas`, `images`, `patches`, `namePrefix`: <https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/>
- RFC 6902 — JavaScript Object Notation (JSON) Patch: <https://www.rfc-editor.org/rfc/rfc6902>
- RFC 6901 — JSON Pointer (`~0` / `~1` escaping): <https://www.rfc-editor.org/rfc/rfc6901>
- kind documentation — Configuration: <https://kind.sigs.k8s.io/docs/user/configuration/>
- DSO202 course material — *03-kind-hands-on-lab: Environment-Specific Configuration with Kustomize on Kind* (sarojsanyasi, HackMD).
# AS2025_DSO202_02230289_Practical-05
