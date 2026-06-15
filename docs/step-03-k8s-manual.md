# Step 03 — Deploy to Kubernetes (manual manifests)

**Goal:** deploy your image to Kubernetes by hand. Externalize configuration into a **ConfigMap** and the database URI into a **Secret**. MongoDB stays **outside** the cluster for now (the app connects to a Mongo on your host).

You write every manifest yourself from the requirements and hints below. There is **no copy-paste YAML here** — figure out the fields. (If you get stuck, the reference answer is in `solved/step-03/`.)

> **Set up:** create a `step-03/` working folder, copy the provided app into it (`cp -r movie-api step-03/app-src`) if you like to keep things together, and make a `step-03/k8s/` folder for the manifests below. You also need the image from Step 02 pushed to Docker Hub (`<dockerhub-user>/movie-api:1.0`).

---

## A. ConfigMap — non-secret config

**Goal:** hold the app's non-sensitive setting `PORT=3000`.

**Task:** write `k8s/configmap.yaml` defining a `ConfigMap` named `movie-api-config` whose `data` has a `PORT` key set to `"3000"`.

*Hints:*
- `apiVersion: v1`, `kind: ConfigMap`.
- Values under `data:` must be **strings** — quote the number.

## B. Secret — the database URI

**Goal:** keep `MONGO_URI` out of the ConfigMap because it can carry credentials.

**Task:** write `k8s/secret.yaml` defining an `Opaque` Secret named `movie-api-secret` that holds `MONGO_URI`. For now point it at a Mongo reachable from the cluster on your host.

*Hints:*
- `kind: Secret`, `type: Opaque`.
- Use `stringData:` (not `data:`) so you can write the value in plain text — Kubernetes base64-encodes it for you.
- From inside minikube / Docker Desktop, your host is reachable as `host.docker.internal`. So the value looks like `mongodb://host.docker.internal:27017/movies`.
- Make sure a Mongo is actually running on your host: the `docker run -d --name mongo -p 27017:27017 mongo:7` container from Step 01 works.

## C. Deployment — the app

**Goal:** run 2 replicas of your image, inject the ConfigMap + Secret as env vars, and add health probes.

**Task:** write `k8s/deployment.yaml` for a `Deployment` named `movie-api`.

*Requirements & hints:*
- `apiVersion: apps/v1`, `kind: Deployment`, `replicas: 2`.
- Label the pods (e.g. `app: movie-api`) and make sure the `selector.matchLabels` matches the pod template labels — a mismatch is the #1 reason a Deployment won't come up.
- Container image: `<dockerhub-user>/movie-api:1.0`, container port `3000`.
- Inject **all** keys from the ConfigMap and Secret at once — look up `envFrom` with `configMapRef` and `secretRef` (simpler than listing each `env` var).
- Add a `livenessProbe` and a `readinessProbe`, both an `httpGet` on path `/health`, port `3000`.

*Self-check:* what happens to the pod if `/health` starts failing? (liveness vs. readiness)

## D. Service — stable access

**Goal:** give the Deployment a stable in-cluster address.

**Task:** write `k8s/service.yaml` for a `Service` named `movie-api`.

*Hints:*
- `type: ClusterIP`.
- `selector` must match the pod labels from the Deployment.
- Expose `port: 80` and forward to the container's `targetPort: 3000`.

---

## E. Apply and verify

**Tasks (work out the commands):**
1. Apply the whole `k8s/` folder at once.
2. List the app pods and confirm they reach `Running`/ready.
3. Tail the Deployment's logs and confirm it printed that it connected to your host Mongo.
4. Port-forward the Service to your machine and curl `/health`, then create and list a movie.

*Hints:*
- `kubectl apply` takes `-f <file-or-dir>`.
- `kubectl get pods -l app=movie-api`, `kubectl logs deploy/movie-api`.
- `kubectl port-forward svc/movie-api 8080:80`, then curl `http://localhost:8080/...`.

> **Local image note (minikube/kind):** if you didn't push to Docker Hub and instead loaded the image locally (`minikube image load movie-api:1.0`), set the container `image:` to that local tag and add `imagePullPolicy: Never` so the kubelet doesn't try to pull from a registry.

---

## What you learned

- Config and secrets are externalized from the image: the same image runs anywhere, configured by a ConfigMap + Secret via `envFrom`.
- A Service gives the Deployment a stable name and port inside the cluster.

## Next

→ [Step 04 — MongoDB on Kubernetes](step-04-k8s-mongodb.md)
