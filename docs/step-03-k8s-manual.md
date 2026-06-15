# Step 03 — Deploy to Kubernetes (manual manifests)

**Goal:** deploy your image to Kubernetes by hand, with configuration in a ConfigMap and the database URI in a Secret. MongoDB stays **outside** the cluster for now.

You write these manifests yourself. Put them in a `k8s/` folder (anywhere convenient, e.g. `step-03/k8s/`).

---

## 1. ConfigMap — non-secret config

`configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: movie-api-config
data:
  PORT: "3000"
```

## 2. Secret — the database URI

The `MONGO_URI` can contain credentials, so it belongs in a Secret. `stringData` lets you write plain text; Kubernetes base64-encodes it for you.

`secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: movie-api-secret
type: Opaque
stringData:
  # Point this at a Mongo reachable from the cluster.
  # On Docker Desktop / minikube you can reach your host via host.docker.internal.
  MONGO_URI: "mongodb://host.docker.internal:27017/movies"
```

> **The Mongo container must publish its port to the host.** The pods run *inside* the cluster and reach your host through `host.docker.internal`, so Mongo has to be listening on the host network — not just on Docker's internal bridge. Run the `mongo:7` container with the host port published (`-p 27017:27017`); without it the container runs but the cluster can't connect, and the app pods fail their readiness probe.
>
> _Hint:_ confirm the port is published before deploying — `docker ps` should show the Mongo container mapping `0.0.0.0:27017->27017/tcp`.
>
> _On minikube/Linux:_ if `host.docker.internal` doesn't resolve, point `MONGO_URI` at your host's LAN IP instead and open port `27017` in the firewall.

## 3. Deployment — the app

Note `envFrom`: it injects every key from the ConfigMap and the Secret as environment variables. The probes hit `/health`.

`deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: movie-api
  labels:
    app: movie-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: movie-api
  template:
    metadata:
      labels:
        app: movie-api
    spec:
      containers:
        - name: movie-api
          image: <dockerhub-user>/movie-api:1.0
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: movie-api-config
            - secretRef:
                name: movie-api-secret
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 3
            periodSeconds: 5
```

## 4. Service — stable access

`service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: movie-api
spec:
  type: ClusterIP
  selector:
    app: movie-api
  ports:
    - port: 80
      targetPort: 3000
```

---

## 5. Apply and verify

```bash
kubectl apply -f k8s/

kubectl get pods -l app=movie-api
kubectl logs deploy/movie-api          # should show "[db] connected to ..."
```

Now reach the service from your machine and hit `/health`. You have two options:

- **Option A — `minikube service` (preferred on minikube).** Ask minikube for the service URL (`minikube service movie-api --url`), then `curl` the URL it prints. Because the cluster runs inside a VM/container, this is the reliable way to reach a `ClusterIP`/`NodePort` Service.
- **Option B — `kubectl port-forward` (any cluster).** Forward a local port to the Service and `curl localhost`. Works everywhere but ties up the terminal while the tunnel stays open.

> **`ImagePullBackOff` / `ErrImagePull`?** If the pod can't pull the image — e.g. an anonymous Docker Hub pull-rate limit (`pull access denied` / `toomanyrequests` for an anonymous user), or the image only exists locally — you can sidestep the registry entirely: pull (or build) the image on your host, then load it straight into minikube's container runtime with `minikube image load`. Confirm it landed with `minikube image ls`.
>
> _Hint:_ also set `imagePullPolicy: IfNotPresent` on the container so Kubernetes uses the loaded image instead of pulling again. Re-apply the manifests and restart the rollout (`kubectl rollout restart deploy/movie-api`) to pick up the change.

---

## What you learned

- Config and secrets are externalized from the image: the same image runs anywhere, configured by ConfigMap + Secret via `envFrom`.
- A Service gives the Deployment a stable name and port inside the cluster.

## Next

→ [Step 04 — MongoDB on Kubernetes](step-04-k8s-mongodb.md)
