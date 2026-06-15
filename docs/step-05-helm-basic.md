# Step 05 — Package as a Helm chart

**Goal:** turn the raw manifests from Step 04 into a parameterized, reusable **Helm chart**. This is the core of the lab — you will write `values.yaml`, helper templates, and conditionals yourself, then practice install / upgrade / rollback / uninstall.

There is **no copy-paste chart here** — you build it from the requirements and hints. (Reference answer: `solved/step-05/movie-chart/`.)

> **Set up:** `helm create movie-chart` scaffolds a chart. Delete the generated files under `templates/` (keep `_helpers.tpl` as a starting point and the `Chart.yaml`/`values.yaml` to edit). You will end up with this layout:
>
> ```
> movie-chart/
> ├── Chart.yaml
> ├── values.yaml
> ├── values-prod.yaml
> └── templates/
>     ├── _helpers.tpl
>     ├── configmap.yaml
>     ├── secret.yaml
>     ├── deployment.yaml
>     ├── service.yaml
>     ├── mongo-service.yaml
>     ├── mongo-statefulset.yaml
>     └── NOTES.txt
> ```

---

## A. `Chart.yaml`

**Task:** set chart metadata.

*Hints:* `apiVersion: v2`, `name: movie-chart`, `type: application`, a `version:` (the chart version, e.g. `0.1.0`) and an `appVersion:` (the image version, e.g. `"1.0"`). Know the difference: bump `version` when the chart changes, `appVersion` when the app image changes.

## B. `values.yaml` — the knobs

**Goal:** make everything that varied between environments a value.

**Task:** define values for: `replicaCount`; an `image` block (`repository`, `tag`, `pullPolicy`); a `service` block (`type`, `port: 80`, `targetPort: 3000`); a `config` map containing `PORT: "3000"`; a `secret.mongoUri`; a `mongo` block (`enabled: true`, `image`, `storage`, `port`); and an `extraEnv` map (e.g. `LOG_LEVEL: info`).

*Hints:*
- For `secret.mongoUri`, you want the Mongo host to depend on the release name. Put a Helm expression **inside the string**: `"mongodb://{{ .Release.Name }}-mongo:27017/movies"`. (You'll render it with `tpl` in the Secret template — see step F.)
- `mongo.enabled` will gate the whole in-cluster Mongo via a conditional (step I/J).

## C. `values-prod.yaml` — an environment override

**Task:** create a second values file with production overrides — e.g. `replicaCount: 4`, `service.type: LoadBalancer`, `extraEnv.LOG_LEVEL: warn`. Applied later with `-f`.

## D. `templates/_helpers.tpl` — named templates

**Goal:** define reusable name/label snippets so every resource is named and labelled consistently.

**Task:** define these named templates with `define`/`end`:
- `movie-chart.name` — the chart name, allowing a `nameOverride`.
- `movie-chart.fullname` — `<release>-<name>` (e.g. `demo-movie-chart`).
- `movie-chart.labels` — common labels (`app.kubernetes.io/name`, `/instance`, `/managed-by`, `helm.sh/chart`).
- `movie-chart.selectorLabels` — the **stable** subset used in selectors (name + instance only).
- `movie-chart.mongoName` — `<release>-mongo`.

*Hints:*
- Use `printf` to build names, and pipe through `trunc 63 | trimSuffix "-"` to stay within k8s name limits.
- **Never** put `helm.sh/chart` or a version label in `selectorLabels` — selectors are immutable, and a version bump would break upgrades.
- Reference a helper elsewhere with `{{ include "movie-chart.fullname" . }}`.

## E. `templates/configmap.yaml`

**Task:** render a ConfigMap named `{{ include "movie-chart.fullname" . }}-config`, with every key under `.Values.config` turned into a `data` entry.

*Hints:* iterate with `{{- range $key, $val := .Values.config }}` … `{{- end }}`, and `quote` the value. Add the common labels with `{{- include "movie-chart.labels" . | nindent 4 }}`.

## F. `templates/secret.yaml`

**Task:** render an `Opaque` Secret named `...-secret` with `MONGO_URI` in `stringData`.

*Hint:* the value in `values.yaml` itself contains `{{ .Release.Name }}`, so a plain `{{ .Values.secret.mongoUri }}` would emit that text literally. Re-render it with `tpl`: `{{ tpl .Values.secret.mongoUri . | quote }}`.

## G. `templates/deployment.yaml`

**Task:** port your Step 03/04 Deployment to templated form.

*Hints:*
- Name: `{{ include "movie-chart.fullname" . }}`; `replicas: {{ .Values.replicaCount }}`.
- `selector.matchLabels` and the pod template labels both use `{{ include "movie-chart.selectorLabels" . | nindent N }}` — mind the indentation (`nindent 6` under `matchLabels`, `nindent 8` under the template `metadata.labels`).
- Image: `"{{ .Values.image.repository }}:{{ .Values.image.tag }}"`, `imagePullPolicy: {{ .Values.image.pullPolicy }}`, container port `{{ .Values.service.targetPort }}`.
- Keep `envFrom` pointing at the templated ConfigMap and Secret names.
- Render `extraEnv` only if present: wrap an `env:` block in `{{- with .Values.extraEnv }}` … `{{- end }}` and `range` over it.
- Probes hit `/health` on `{{ .Values.service.targetPort }}`.

## H. `templates/service.yaml`

**Task:** Service named `{{ include "movie-chart.fullname" . }}`, `type: {{ .Values.service.type }}`, selector = `selectorLabels`, `port: {{ .Values.service.port }}` → `targetPort: {{ .Values.service.targetPort }}`.

## I. `templates/mongo-service.yaml` (conditional)

**Task:** the headless Mongo Service from Step 04, named `{{ include "movie-chart.mongoName" . }}`, **wrapped in a conditional** so it only renders when in-cluster Mongo is enabled.

*Hint:* wrap the whole file in `{{- if .Values.mongo.enabled }}` … `{{- end }}`. Add a component label (`app.kubernetes.io/component: mongo`) to the labels and selector so it doesn't collide with the app's Service selector.

## J. `templates/mongo-statefulset.yaml` (conditional)

**Task:** the Mongo StatefulSet from Step 04, templated and wrapped in the same `{{- if .Values.mongo.enabled }}` guard. Use `{{ .Values.mongo.image }}` and `{{ .Values.mongo.storage }}`.

## K. `templates/NOTES.txt`

**Task:** print post-install help — the `port-forward` command (using the fullname helper and `.Values.service.port`), and a line that differs depending on `{{ if .Values.mongo.enabled }}`.

---

## L. The Helm workflow

**Tasks (run and observe):**
1. Lint the chart.
2. Render templates locally **without installing** to inspect the output.
3. Install the release as `demo`.
4. Inspect the applied manifests and the pods.
5. Upgrade with an inline override (e.g. bump `replicaCount`).
6. Upgrade again using your `values-prod.yaml`.
7. View release history, then roll back to revision 1.
8. Uninstall.

*Hints (commands to discover):* `helm lint`, `helm template`, `helm install`, `helm get manifest`, `helm upgrade` (with `--set` and with `-f`), `helm history`, `helm rollback`, `helm uninstall`.

> **Verify the conditional works:** render with `--set mongo.enabled=false` and confirm the Mongo Service/StatefulSet disappear from the output.
>
> **Local image note (minikube/kind):** add `--set image.repository=movie-api --set image.tag=1.0 --set image.pullPolicy=Never` if you loaded the image locally instead of pushing to Docker Hub.

---

## What you learned

- **values.yaml** parameterizes a chart; `--set` and `-f` override per environment.
- **`_helpers.tpl`** named templates (`include`) keep names/labels DRY and consistent.
- **Conditionals** (`{{- if .Values.mongo.enabled }}`) and **range/with** make templates flexible.
- `helm upgrade` / `rollback` / `history` give you versioned, reversible releases.

## Next

→ [Step 06 — MongoDB as a Helm dependency](step-06-helm-mongodb-dependency.md)
