# Step 04 — MongoDB on Kubernetes

**Goal:** move MongoDB *inside* the cluster with persistent storage, and repoint the app at it. You add two new manifests and change one line in the Secret.

Write the new manifests yourself from the hints. No copy-paste YAML. (Reference answer: `solved/step-04/`.)

> **Set up:** start from a copy of your Step 03 `k8s/` folder (e.g. `cp -r step-03 step-04`). You will reuse `configmap.yaml`, `deployment.yaml`, and `service.yaml` unchanged, edit `secret.yaml`, and add two Mongo manifests.

---

## A. Headless Service for MongoDB

**Goal:** give the MongoDB pod a stable network identity so other pods can reach it by name (`mongo`).

**Task:** write `k8s/mongo-service.yaml` for a `Service` named `mongo`.

*Hints:*
- Make it **headless** — set `clusterIP: None`. (A headless Service is the standard partner for a StatefulSet; it gives stable per-pod DNS like `mongo-0.mongo`.)
- `selector` should match the Mongo pod labels (e.g. `app: mongo`).
- Port `27017`.

## B. StatefulSet for MongoDB (with persistent storage)

**Goal:** run MongoDB with storage that survives pod restarts.

**Task:** write `k8s/mongo-statefulset.yaml` for a `StatefulSet` named `mongo`.

*Requirements & hints:*
- `apiVersion: apps/v1`, `kind: StatefulSet`, `replicas: 1`.
- `serviceName` must point at the headless Service (`mongo`).
- Pod labels (`app: mongo`) must match both the StatefulSet `selector` and the headless Service `selector`.
- Container: image `mongo:7`, port `27017`.
- For durable storage, use `volumeClaimTemplates` (not a plain `volumes:` entry) — this creates a PersistentVolumeClaim per replica. Mount it at `/data/db` (where MongoDB stores its files). Request e.g. `1Gi`, `accessModes: ["ReadWriteOnce"]`.

*Self-check:* why a StatefulSet + `volumeClaimTemplates` instead of a Deployment with one PVC? (stable identity + per-replica storage)

## C. Repoint the app's Secret at the in-cluster Mongo

**Task:** edit `k8s/secret.yaml` so `MONGO_URI` uses the in-cluster Service name as the host instead of `host.docker.internal`.

*Hint:* the host part becomes the Service name `mongo` → `mongodb://mongo:27017/movies`.

> `configmap.yaml`, `deployment.yaml`, and `service.yaml` are unchanged from Step 03.

---

## D. Apply in the right order

**Goal:** bring MongoDB up *before* the app, so the app finds the database on startup. (If the app starts first it will fail to connect and restart until Mongo is ready — not fatal, but bring up Mongo first to avoid the churn.)

**Tasks (work out the commands):**
1. Apply the Mongo Service and StatefulSet, then wait for the StatefulSet to be ready.
2. Apply the app's ConfigMap, Secret, Deployment, and Service.

*Hints:*
- `kubectl apply -f <file>` (you can pass multiple `-f`).
- Wait with `kubectl rollout status statefulset/mongo`.

## E. Verify

**Tasks:**
1. Confirm the app logs show it connected to `mongodb://mongo:27017/...`.
2. Port-forward and create/list a movie.
3. (Optional) exec into the Mongo pod and count documents to prove the data lives in the in-cluster DB.

*Hints:*
- `kubectl logs deploy/movie-api | grep "connected to"`.
- `kubectl exec mongo-0 -- mongosh movies --quiet --eval "db.movies.countDocuments()"`.

---

## What you learned

- StatefulSet + headless Service + `volumeClaimTemplates` (PVC) is the standard pattern for stateful workloads like databases on Kubernetes.
- The app didn't change at all — only the `MONGO_URI` in the Secret. Same image, new database target.

## Next

→ [Step 05 — Package as a Helm chart](step-05-helm-basic.md)
