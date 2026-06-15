# Step 06 — MongoDB as a Helm dependency

**Goal:** stop maintaining your own MongoDB manifests and consume the community **Bitnami MongoDB** chart as a dependency (subchart).

You modify your Step 05 chart yourself from the hints. No copy-paste chart. (Reference answer: `solved/step-06/movie-chart/`.)

> **Set up:** start from a copy of your Step 05 `movie-chart/`.

---

## A. Remove the local Mongo templates

**Task:** delete `templates/mongo-service.yaml` and `templates/mongo-statefulset.yaml` — the subchart replaces them. You can also drop the `movie-chart.mongoName` helper, since the Mongo Service name now comes from the subchart (`<release>-mongodb`).

## B. Declare the dependency in `Chart.yaml`

**Goal:** make Helm pull the Bitnami MongoDB chart as a subchart.

**Task:** add a `dependencies:` list to `Chart.yaml` with one entry for `mongodb`.

*Hints / required fields:*
- `name: mongodb`
- `version: "15.x.x"` (a version range, or pin an exact one like `15.6.26`)
- `repository: "https://charts.bitnami.com/bitnami"`
- `condition: mongodb.enabled` — lets you toggle the whole dependency on/off from values.
- Bump the chart `version` (e.g. `0.2.0`) since the chart changed.

## C. Move Mongo settings under the subchart key in `values.yaml`

**Goal:** configure the subchart and point the app's `MONGO_URI` at it.

**Task 1 — `secret.mongoUri`:** the Bitnami chart names its Service `<release>-mongodb`. Update the URI to connect there as the root user:
```
mongodb://root:secretpass@{{ .Release.Name }}-mongodb:27017/movies?authSource=admin
```

**Task 2 — the `mongodb:` block.** Anything under a top-level key named after the subchart (`mongodb:`) is passed straight into it. You need:
- `enabled: true`
- `auth.rootPassword: "secretpass"` (must match the password in `mongoUri`)
- `architecture: standalone`
- `persistence.enabled: true`, `persistence.size: 1Gi`
- **the image override below**

> ### Two non-obvious gotchas — read before you run
>
> **1. The default Bitnami image no longer pulls.** As of 2025, Bitnami removed the old free tags from `docker.io/bitnami/*`, so the chart's default image fails with `manifest unknown` → `ImagePullBackOff`. The free images moved to `docker.io/bitnamilegacy/*`. You must override the registry **and** allow the override:
> ```yaml
> mongodb:
>   image:
>     registry: docker.io
>     repository: bitnamilegacy/mongodb
>   global:
>     security:
>       allowInsecureImages: true
> ```
> Without `allowInsecureImages: true` the chart refuses a non-default image.
>
> **2. Don't set `auth.databases` alone.** The Bitnami chart requires `auth.usernames` **and** `auth.databases` together (equal length); setting only one fails `helm lint`/install. We use only the root credentials and let the app create the `movies` DB on first write.

**Task 3:** the rest of your templates (`configmap.yaml`, `secret.yaml`, `deployment.yaml`, `service.yaml`, `NOTES.txt`) are unchanged from Step 05. Optionally tweak `NOTES.txt` to print the subchart Service name (`{{ .Release.Name }}-mongodb`) when `.Values.mongodb.enabled`.

---

## D. Pull the dependency and deploy

**Tasks (work out the commands):**
1. Add the Bitnami Helm repo and update it.
2. Download the dependency into `charts/` (this also writes `Chart.lock`).
3. Lint and render to confirm both the API and the mongodb subchart appear.
4. Install the release.

*Hints:*
- `helm repo add bitnami https://charts.bitnami.com/bitnami`, then `helm repo update`.
- `helm dependency update ./movie-chart` (fetches + writes `Chart.lock`). Later, `helm dependency build` installs exactly what `Chart.lock` pins without re-refreshing the index.
- `helm lint`, `helm template`, `helm install demo ./movie-chart`.

> **Troubleshooting — `helm repo update` / `helm dependency update` times out.**
> The Bitnami index is large and sometimes rate-limited (`context deadline exceeded`). Retry `helm repo update bitnami`, or pin an exact version in `Chart.yaml` and use `helm dependency build`, or pull a single chart via OCI: `helm pull oci://registry-1.docker.io/bitnamicharts/mongodb --version <v>`.
>
> **Local image note (minikube/kind):** add `--set image.repository=movie-api --set image.tag=1.0 --set image.pullPolicy=Never` for *your* app image if you loaded it locally. (The `mongodb` legacy-image override lives in `values.yaml`, so you don't need to repeat it on the command line.)

## E. Verify

**Tasks:**
1. Wait for all pods (the app + `<release>-mongodb`) to be ready. The app may restart a couple of times until Mongo is up — that's expected.
2. Confirm the app logs show it connected to `<release>-mongodb`.
3. Port-forward and create/list a movie.

*Hints:*
- `kubectl get pods -l app.kubernetes.io/instance=demo`
- `kubectl logs deploy/<app-deploy> | grep "connected to"`
- `kubectl port-forward svc/demo-movie-chart 8080:80`

## F. Upgrade / rollback / uninstall

**Task:** repeat the `helm upgrade --set ...`, `helm history`, `helm rollback`, `helm uninstall` workflow from Step 05 to confirm it all still works with a dependency in play.

---

## What you learned

- A chart can depend on other charts via `Chart.yaml` `dependencies`; `helm dependency update` vendors them into `charts/` and pins them in `Chart.lock`.
- Subchart configuration lives under a top-level key named after the subchart (`mongodb:`), and a `condition` toggles the whole dependency.
- Real charts drift: image registries move, values get stricter. Reading the chart's own values and error messages is part of the job.

## Done

You took a provided app from local run → container → manual Kubernetes → in-cluster database → a Helm chart → a chart with a managed dependency. 🎉
