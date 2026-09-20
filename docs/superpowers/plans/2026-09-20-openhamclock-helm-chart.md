# OpenHamClock Helm Chart Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Helm chart that deploys the OpenHamClock web dashboard (image `ghcr.io/accius/openhamclock`) to Kubernetes, in a new standalone repo at `~/src/github.com/adamancini/openhamclock-helm`.

**Architecture:** One Helm 3 chart at `charts/openhamclock/` containing a Deployment (env sourced from a ConfigMap + a Secret/`existingSecret`), a ClusterIP Service for the HTTP UI, an optional Service for the WSJT-X/N1MM UDP integrations, an optional Ingress, and a PVC (or `emptyDir`) mounted at `/data`. Config surfaces as generic `config.env` / `secretEnv` maps rather than one values field per upstream env var, since the app already has 70+ documented vars and adds more over time.

**Tech Stack:** Helm 3 (`v4.3.0` client installed locally — chart itself targets the Helm 3 template API, no `apiVersion: v1` legacy charts), Kubernetes manifests (apps/v1 Deployment, v1 Service/ConfigMap/Secret/PersistentVolumeClaim/ServiceAccount, networking.k8s.io/v1 Ingress).

## Global Constraints

- Chart lives at `charts/openhamclock/` inside the new repo `~/src/github.com/adamancini/openhamclock-helm` (not inside the upstream `openhamclock` checkout — this chart is not being contributed upstream).
- Default image: `ghcr.io/accius/openhamclock`, default tag tracks `appVersion` in `Chart.yaml`.
- HTTP port `3000`, WSJT-X UDP port `2237`, N1MM UDP port `12060` — these match the upstream `Dockerfile`/`docker-compose.yml` and must not be renumbered.
- Health probe path: `/api/health` (matches upstream `Dockerfile` `HEALTHCHECK`).
- UDP services (`udpServices.*`) default to **disabled** — confirmed with the user; a `ClusterIP` can't reach WSJT-X/N1MM from outside the cluster, and forcing a `NodePort`/`LoadBalancer` on everyone has infra/cost implications they should opt into.
- Persistence (`/data`) defaults to **enabled** via PVC, `1Gi`, falls back to `emptyDir` when disabled.
- No HPA, PodDisruptionBudget, NetworkPolicy, or `values.schema.json` in this pass (see spec's "Out of scope").
- Verification throughout is `helm lint` and `helm template` (no live cluster targeted by this plan) — there is no app code under test, so "tests" here mean rendering the chart with specific `--set`/`-f` overrides and asserting on the rendered YAML.

---

## File Structure

```
openhamclock-helm/
├── .gitignore
├── README.md
└── charts/
    └── openhamclock/
        ├── Chart.yaml
        ├── .helmignore
        ├── values.yaml
        ├── README.md
        └── templates/
            ├── _helpers.tpl
            ├── serviceaccount.yaml
            ├── configmap.yaml
            ├── secret.yaml
            ├── deployment.yaml
            ├── service.yaml
            ├── service-udp.yaml
            ├── pvc.yaml
            ├── ingress.yaml
            └── NOTES.txt
```

- `_helpers.tpl` — naming/label helpers, shared by every other template.
- `serviceaccount.yaml` / `configmap.yaml` / `secret.yaml` — small, independent resources the Deployment references by name.
- `deployment.yaml` — the core workload; depends on the helper names and on the ConfigMap/Secret/PVC names.
- `service.yaml` / `service-udp.yaml` / `pvc.yaml` / `ingress.yaml` — independently toggleable resources.
- `NOTES.txt` — user-facing post-install text, written last once all the toggles it needs to describe exist.

---

## Task 1: Chart scaffold, helpers, ServiceAccount

**Files:**
- Create: `openhamclock-helm/.gitignore`
- Create: `openhamclock-helm/charts/openhamclock/Chart.yaml`
- Create: `openhamclock-helm/charts/openhamclock/.helmignore`
- Create: `openhamclock-helm/charts/openhamclock/values.yaml`
- Create: `openhamclock-helm/charts/openhamclock/templates/_helpers.tpl`
- Create: `openhamclock-helm/charts/openhamclock/templates/serviceaccount.yaml`

**Interfaces:**
- Produces: template helpers `openhamclock.name`, `openhamclock.fullname`, `openhamclock.chart`, `openhamclock.labels`, `openhamclock.selectorLabels`, `openhamclock.serviceAccountName` — every later template includes these by name.
- Produces: `values.yaml` top-level keys `nameOverride`, `fullnameOverride`, `replicaCount`, `image.repository`, `image.tag`, `image.pullPolicy`, `serviceAccount.create`, `serviceAccount.name`, `serviceAccount.annotations` — later tasks add more keys to this same file.

- [ ] **Step 1: Create the repo `.gitignore`**

```gitignore
*.tgz
.DS_Store
```

- [ ] **Step 2: Create `Chart.yaml`**

```yaml
apiVersion: v2
name: openhamclock
description: A Helm chart for deploying OpenHamClock, a real-time amateur radio dashboard
type: application
version: 0.1.0
appVersion: "26.7.4"
home: https://openhamclock.com
sources:
  - https://github.com/accius/openhamclock
keywords:
  - amateur-radio
  - ham-radio
  - dashboard
maintainers:
  - name: adamancini
```

- [ ] **Step 3: Create `.helmignore`**

```
.git/
.gitignore
*.swp
*.bak
*.tmp
*.orig
.DS_Store
```

- [ ] **Step 4: Create the start of `values.yaml`**

```yaml
# -- Number of Deployment replicas. OpenHamClock keeps no state that
# coordinates across replicas (settings/stats live on the PVC), so this
# is left at 1 by default rather than adding HPA/PDB machinery.
replicaCount: 1

nameOverride: ""
fullnameOverride: ""

image:
  repository: ghcr.io/accius/openhamclock
  tag: ""
  pullPolicy: IfNotPresent

imagePullSecrets: []

serviceAccount:
  # -- Whether to create a ServiceAccount for the pod. The app never talks
  # to the Kubernetes API, so no RBAC rules are attached either way.
  create: true
  name: ""
  annotations: {}
```

- [ ] **Step 5: Create `templates/_helpers.tpl`**

```
{{- define "openhamclock.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "openhamclock.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- if contains $name .Release.Name -}}
{{- .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}
{{- end -}}

{{- define "openhamclock.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "openhamclock.labels" -}}
helm.sh/chart: {{ include "openhamclock.chart" . }}
{{ include "openhamclock.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}

{{- define "openhamclock.selectorLabels" -}}
app.kubernetes.io/name: {{ include "openhamclock.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}

{{- define "openhamclock.serviceAccountName" -}}
{{- if .Values.serviceAccount.create -}}
{{- default (include "openhamclock.fullname" .) .Values.serviceAccount.name -}}
{{- else -}}
{{- default "default" .Values.serviceAccount.name -}}
{{- end -}}
{{- end -}}
```

- [ ] **Step 6: Create `templates/serviceaccount.yaml`**

```yaml
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "openhamclock.serviceAccountName" . }}
  labels:
    {{- include "openhamclock.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
```

- [ ] **Step 7: Verify with `helm lint`**

Run: `helm lint ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock`
Expected: `0 chart(s) linted, 0 chart(s) failed`

- [ ] **Step 8: Verify with `helm template`**

Run: `helm template test ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock`
Expected: renders a single `ServiceAccount` named `test-openhamclock` with `app.kubernetes.io/name: openhamclock` and `app.kubernetes.io/instance: test` labels, no errors.

- [ ] **Step 9: Commit**

```bash
cd ~/src/github.com/adamancini/openhamclock-helm
git add .gitignore charts/openhamclock/Chart.yaml charts/openhamclock/.helmignore charts/openhamclock/values.yaml charts/openhamclock/templates/_helpers.tpl charts/openhamclock/templates/serviceaccount.yaml
git commit -m "feat: scaffold openhamclock chart with helpers and ServiceAccount"
```

---

## Task 2: ConfigMap and Secret for environment configuration

**Files:**
- Modify: `openhamclock-helm/charts/openhamclock/values.yaml`
- Create: `openhamclock-helm/charts/openhamclock/templates/configmap.yaml`
- Create: `openhamclock-helm/charts/openhamclock/templates/secret.yaml`

**Interfaces:**
- Consumes: `openhamclock.fullname`, `openhamclock.labels` from Task 1.
- Produces: ConfigMap named `<fullname>-env`, Secret named `<fullname>-env` (only rendered when `.Values.existingSecret` is empty) — Task 3's Deployment references both by these exact names via `envFrom`.
- Produces: `values.yaml` keys `config.env` (map[string]string), `secretEnv` (map[string]string, default `{}`), `existingSecret` (string, default `""`).

- [ ] **Step 1: Add `config.env`, `secretEnv`, `existingSecret` to `values.yaml`**

Append to `values.yaml`:

```yaml
# -- Non-secret environment variables, rendered into a ConfigMap and
# wired into the Deployment via envFrom. The upstream app documents 70+
# of these in its .env.example and adds more as features ship, so this
# is a generic map rather than one values field per variable — add any
# key from .env.example you need that isn't already listed here.
config:
  env:
    CALLSIGN: "N0CALL"
    LOCATOR: "FN31"
    PORT: "3000"
    HOST: "0.0.0.0"
    TIME_FORMAT: "12"
    THEME: "dark"
    LAYOUT: "modern"
    DISTUNITS: "imperial"
    TEMPUNITS: "imperial"
    PRESSUNITS: "imperial"
    SHOW_POTA: "true"
    SHOW_SATELLITES: "true"
    SHOW_DX_PATHS: "true"
    SHOW_DX_WEATHER: "true"
    WSJTX_ENABLED: "true"
    WSJTX_UDP_PORT: "2237"
    N1MM_UDP_ENABLED: "false"
    N1MM_UDP_PORT: "12060"
    SPOT_RETENTION_MINUTES: "30"

# -- Secret environment variables (API keys, credentials), rendered into
# a Secret and wired into the Deployment via envFrom. Empty by default —
# add keys like OPENWEATHER_API_KEY, API_WRITE_KEY, WSJTX_RELAY_KEY here,
# or set `existingSecret` instead to reference a Secret you manage
# yourself (e.g. via External Secrets / SealedSecrets).
secretEnv: {}

# -- Name of a pre-existing Secret to use instead of the one this chart
# would otherwise render from `secretEnv`. Takes precedence when set.
existingSecret: ""
```

- [ ] **Step 2: Create `templates/configmap.yaml`**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "openhamclock.fullname" . }}-env
  labels:
    {{- include "openhamclock.labels" . | nindent 4 }}
data:
  {{- range $key, $value := .Values.config.env }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
```

- [ ] **Step 3: Create `templates/secret.yaml`**

```yaml
{{- if not .Values.existingSecret }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "openhamclock.fullname" . }}-env
  labels:
    {{- include "openhamclock.labels" . | nindent 4 }}
type: Opaque
stringData:
  {{- range $key, $value := .Values.secretEnv }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
{{- end }}
```

- [ ] **Step 4: Verify default render includes both resources**

Run: `helm template test ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock`
Expected: a `ConfigMap` named `test-openhamclock-env` with `data.CALLSIGN: "N0CALL"`, and a `Secret` named `test-openhamclock-env` with empty `stringData`.

- [ ] **Step 5: Verify `existingSecret` suppresses the rendered Secret**

Run: `helm template test ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock --set existingSecret=my-secret`
Expected: no `kind: Secret` in the output.

- [ ] **Step 6: Commit**

```bash
cd ~/src/github.com/adamancini/openhamclock-helm
git add charts/openhamclock/values.yaml charts/openhamclock/templates/configmap.yaml charts/openhamclock/templates/secret.yaml
git commit -m "feat: add ConfigMap/Secret for environment configuration"
```

---

## Task 3: Deployment

**Files:**
- Modify: `openhamclock-helm/charts/openhamclock/values.yaml`
- Create: `openhamclock-helm/charts/openhamclock/templates/deployment.yaml`

**Interfaces:**
- Consumes: `openhamclock.fullname`, `openhamclock.labels`, `openhamclock.selectorLabels`, `openhamclock.serviceAccountName` (Task 1); ConfigMap/Secret names `<fullname>-env` and `.Values.existingSecret` (Task 2).
- Produces: Deployment named `<fullname>` with selector labels `app.kubernetes.io/name`/`app.kubernetes.io/instance`, container named `openhamclock` listening on container port name `http` (3000) — Task 4's Service selects on these same selector labels and targets port name `http`. Volume named `data` mounted at `.Values.persistence.mountPath` — Task 6's PVC is referenced by this volume when `.Values.persistence.enabled` is true.
- Produces: `values.yaml` keys `resources`, `podSecurityContext`, `securityContext`, `livenessProbe`, `readinessProbe`, `persistence.enabled`, `persistence.mountPath`, `nodeSelector`, `tolerations`, `affinity`.

- [ ] **Step 1: Add Deployment-related values to `values.yaml`**

Append to `values.yaml`:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    memory: 2Gi

# -- Pod-level securityContext. Not forcing runAsNonRoot: the upstream
# Dockerfile has no USER directive (runs as root by default) and this
# hasn't been verified against a non-root UID, so it's left off by
# default and available for users who've confirmed it works for them.
podSecurityContext: {}

# -- Container-level securityContext.
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: false

livenessProbe:
  httpGet:
    path: /api/health
    port: http
  initialDelaySeconds: 10
  periodSeconds: 30
  timeoutSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /api/health
    port: http
  initialDelaySeconds: 5
  periodSeconds: 15
  timeoutSeconds: 10
  failureThreshold: 3

persistence:
  # -- Persist /data (stats, settings sync, push-subscription state)
  # across pod restarts via a PVC. When false, an emptyDir is used
  # instead and this state is lost on every pod restart.
  enabled: true
  mountPath: /data

nodeSelector: {}
tolerations: []
affinity: {}
```

- [ ] **Step 2: Create `templates/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "openhamclock.fullname" . }}
  labels:
    {{- include "openhamclock.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "openhamclock.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "openhamclock.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "openhamclock.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: openhamclock
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          ports:
            - name: http
              containerPort: 3000
              protocol: TCP
            - name: wsjtx-udp
              containerPort: {{ .Values.config.env.WSJTX_UDP_PORT | default "2237" }}
              protocol: UDP
            - name: n1mm-udp
              containerPort: {{ .Values.config.env.N1MM_UDP_PORT | default "12060" }}
              protocol: UDP
          envFrom:
            - configMapRef:
                name: {{ include "openhamclock.fullname" . }}-env
            - secretRef:
                name: {{ .Values.existingSecret | default (printf "%s-env" (include "openhamclock.fullname" .)) }}
          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 12 }}
          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 12 }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          volumeMounts:
            - name: data
              mountPath: {{ .Values.persistence.mountPath }}
      volumes:
        - name: data
          {{- if .Values.persistence.enabled }}
          persistentVolumeClaim:
            claimName: {{ include "openhamclock.fullname" . }}-data
          {{- else }}
          emptyDir: {}
          {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

- [ ] **Step 3: Verify default render**

Run: `helm template test ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock`
Expected: a `Deployment` named `test-openhamclock` with image `ghcr.io/accius/openhamclock:26.7.4`, `envFrom` referencing `ConfigMap test-openhamclock-env` and `Secret test-openhamclock-env`, both probes on path `/api/health` port `http`, and a `data` volume with `persistentVolumeClaim.claimName: test-openhamclock-data`.

- [ ] **Step 4: Verify `persistence.enabled=false` falls back to emptyDir**

Run: `helm template test ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock --set persistence.enabled=false`
Expected: the `data` volume has `emptyDir: {}` and no `persistentVolumeClaim` key.

- [ ] **Step 5: Verify `existingSecret` is used instead of the chart's own Secret**

Run: `helm template test ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock --set existingSecret=my-secret`
Expected: `envFrom` has `secretRef.name: my-secret`.

- [ ] **Step 6: Commit**

```bash
cd ~/src/github.com/adamancini/openhamclock-helm
git add charts/openhamclock/values.yaml charts/openhamclock/templates/deployment.yaml
git commit -m "feat: add openhamclock Deployment"
```

---

## Task 4: HTTP Service

**Files:**
- Modify: `openhamclock-helm/charts/openhamclock/values.yaml`
- Create: `openhamclock-helm/charts/openhamclock/templates/service.yaml`

**Interfaces:**
- Consumes: `openhamclock.fullname`, `openhamclock.labels`, `openhamclock.selectorLabels` (Task 1); selector must match the pod labels from Task 3; targets container port name `http`.
- Produces: Service named `<fullname>` — Task 7's Ingress references this Service name and `.Values.service.port`.
- Produces: `values.yaml` keys `service.type`, `service.port`.

- [ ] **Step 1: Add Service values**

Append to `values.yaml`:

```yaml
service:
  type: ClusterIP
  port: 3000
```

- [ ] **Step 2: Create `templates/service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "openhamclock.fullname" . }}
  labels:
    {{- include "openhamclock.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "openhamclock.selectorLabels" . | nindent 4 }}
```

- [ ] **Step 3: Verify render**

Run: `helm template test ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock`
Expected: a `Service` named `test-openhamclock`, `type: ClusterIP`, port `3000` named `http` targeting port `http`, selector `app.kubernetes.io/name: openhamclock` / `app.kubernetes.io/instance: test`.

- [ ] **Step 4: Commit**

```bash
cd ~/src/github.com/adamancini/openhamclock-helm
git add charts/openhamclock/values.yaml charts/openhamclock/templates/service.yaml
git commit -m "feat: add openhamclock HTTP Service"
```

---

## Task 5: UDP Services (WSJT-X / N1MM)

**Files:**
- Modify: `openhamclock-helm/charts/openhamclock/values.yaml`
- Create: `openhamclock-helm/charts/openhamclock/templates/service-udp.yaml`

**Interfaces:**
- Consumes: `openhamclock.fullname`, `openhamclock.labels`, `openhamclock.selectorLabels` (Task 1); targets container port names `wsjtx-udp`/`n1mm-udp` from Task 3's Deployment.
- Produces: Service named `<fullname>-udp`, rendered only when `.Values.udpServices.enabled` is true and at least one of `.Values.udpServices.wsjtx.enabled` / `.Values.udpServices.n1mm.enabled` is true.
- Produces: `values.yaml` keys `udpServices.enabled`, `udpServices.type`, `udpServices.wsjtx.enabled`, `udpServices.wsjtx.port`, `udpServices.wsjtx.nodePort`, `udpServices.n1mm.enabled`, `udpServices.n1mm.port`, `udpServices.n1mm.nodePort`.

- [ ] **Step 1: Add UDP service values**

Append to `values.yaml`:

```yaml
udpServices:
  # -- Exposes the WSJT-X/N1MM UDP listeners outside the cluster. A plain
  # ClusterIP can't be reached by an operator's WSJT-X/N1MM software, so
  # enabling this also requires picking a `type` (NodePort or
  # LoadBalancer) — left off by default so nobody gets a NodePort
  # reserved or a cloud LoadBalancer provisioned without asking for it.
  enabled: false
  type: NodePort
  wsjtx:
    enabled: true
    port: 2237
    nodePort: null
  n1mm:
    enabled: false
    port: 12060
    nodePort: null
```

- [ ] **Step 2: Create `templates/service-udp.yaml`**

```yaml
{{- if and .Values.udpServices.enabled (or .Values.udpServices.wsjtx.enabled .Values.udpServices.n1mm.enabled) }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "openhamclock.fullname" . }}-udp
  labels:
    {{- include "openhamclock.labels" . | nindent 4 }}
spec:
  type: {{ .Values.udpServices.type }}
  ports:
    {{- if .Values.udpServices.wsjtx.enabled }}
    - port: {{ .Values.udpServices.wsjtx.port }}
      targetPort: wsjtx-udp
      protocol: UDP
      name: wsjtx-udp
      {{- if .Values.udpServices.wsjtx.nodePort }}
      nodePort: {{ .Values.udpServices.wsjtx.nodePort }}
      {{- end }}
    {{- end }}
    {{- if .Values.udpServices.n1mm.enabled }}
    - port: {{ .Values.udpServices.n1mm.port }}
      targetPort: n1mm-udp
      protocol: UDP
      name: n1mm-udp
      {{- if .Values.udpServices.n1mm.nodePort }}
      nodePort: {{ .Values.udpServices.n1mm.nodePort }}
      {{- end }}
    {{- end }}
  selector:
    {{- include "openhamclock.selectorLabels" . | nindent 4 }}
{{- end }}
```

- [ ] **Step 3: Verify default render omits the UDP service**

Run: `helm template test ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock`
Expected: no `test-openhamclock-udp` Service in the output.

- [ ] **Step 4: Verify enabling renders only the WSJT-X port**

Run: `helm template test ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock --set udpServices.enabled=true`
Expected: a `Service` named `test-openhamclock-udp`, `type: NodePort`, one port named `wsjtx-udp` (2237/UDP), no `n1mm-udp` port.

- [ ] **Step 5: Verify enabling both renders both ports**

Run: `helm template test ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock --set udpServices.enabled=true --set udpServices.n1mm.enabled=true`
Expected: `test-openhamclock-udp` Service has both `wsjtx-udp` (2237/UDP) and `n1mm-udp` (12060/UDP) ports.

- [ ] **Step 6: Commit**

```bash
cd ~/src/github.com/adamancini/openhamclock-helm
git add charts/openhamclock/values.yaml charts/openhamclock/templates/service-udp.yaml
git commit -m "feat: add optional WSJT-X/N1MM UDP Service"
```

---

## Task 6: Persistence PVC

**Files:**
- Modify: `openhamclock-helm/charts/openhamclock/values.yaml`
- Create: `openhamclock-helm/charts/openhamclock/templates/pvc.yaml`

**Interfaces:**
- Consumes: `openhamclock.fullname`, `openhamclock.labels` (Task 1); PVC name `<fullname>-data` must match the `claimName` already referenced by Task 3's Deployment volume.
- Produces: PersistentVolumeClaim named `<fullname>-data`, rendered only when `.Values.persistence.enabled` is true.
- Produces: `values.yaml` keys `persistence.size`, `persistence.storageClass`, `persistence.accessMode`.

- [ ] **Step 1: Extend the `persistence` block in `values.yaml`**

Find the `persistence:` block added in Task 3 and replace it with:

```yaml
persistence:
  # -- Persist /data (stats, settings sync, push-subscription state)
  # across pod restarts via a PVC. When false, an emptyDir is used
  # instead and this state is lost on every pod restart.
  enabled: true
  mountPath: /data
  size: 1Gi
  # -- Leave empty to use the cluster's default StorageClass.
  storageClass: ""
  accessMode: ReadWriteOnce
```

- [ ] **Step 2: Create `templates/pvc.yaml`**

```yaml
{{- if .Values.persistence.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "openhamclock.fullname" . }}-data
  labels:
    {{- include "openhamclock.labels" . | nindent 4 }}
spec:
  accessModes:
    - {{ .Values.persistence.accessMode }}
  resources:
    requests:
      storage: {{ .Values.persistence.size }}
  {{- if .Values.persistence.storageClass }}
  storageClassName: {{ .Values.persistence.storageClass }}
  {{- end }}
{{- end }}
```

- [ ] **Step 3: Verify default render includes the PVC**

Run: `helm template test ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock`
Expected: a `PersistentVolumeClaim` named `test-openhamclock-data`, `accessModes: [ReadWriteOnce]`, `requests.storage: 1Gi`, no `storageClassName` key.

- [ ] **Step 4: Verify `persistence.enabled=false` omits the PVC**

Run: `helm template test ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock --set persistence.enabled=false`
Expected: no `PersistentVolumeClaim` in the output (and, per Task 3 Step 4, the Deployment's `data` volume is `emptyDir`).

- [ ] **Step 5: Verify `storageClass` override**

Run: `helm template test ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock --set persistence.storageClass=fast-ssd`
Expected: the PVC has `storageClassName: fast-ssd`.

- [ ] **Step 6: Commit**

```bash
cd ~/src/github.com/adamancini/openhamclock-helm
git add charts/openhamclock/values.yaml charts/openhamclock/templates/pvc.yaml
git commit -m "feat: add persistence PVC for /data"
```

---

## Task 7: Ingress

**Files:**
- Modify: `openhamclock-helm/charts/openhamclock/values.yaml`
- Create: `openhamclock-helm/charts/openhamclock/templates/ingress.yaml`

**Interfaces:**
- Consumes: `openhamclock.fullname`, `openhamclock.labels` (Task 1); Service name `<fullname>` and `.Values.service.port` (Task 4).
- Produces: Ingress named `<fullname>`, rendered only when `.Values.ingress.enabled` is true.
- Produces: `values.yaml` keys `ingress.enabled`, `ingress.className`, `ingress.annotations`, `ingress.host`, `ingress.path`, `ingress.pathType`, `ingress.tls`.

- [ ] **Step 1: Add Ingress values**

Append to `values.yaml`:

```yaml
ingress:
  enabled: false
  className: ""
  annotations: {}
  host: openhamclock.example.com
  path: /
  pathType: Prefix
  # -- TLS configuration, same shape as the native Ingress spec.
  # Example:
  # tls:
  #   - secretName: openhamclock-tls
  #     hosts:
  #       - openhamclock.example.com
  tls: []
```

- [ ] **Step 2: Create `templates/ingress.yaml`**

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "openhamclock.fullname" . }}
  labels:
    {{- include "openhamclock.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- with .Values.ingress.tls }}
  tls:
    {{- toYaml . | nindent 4 }}
  {{- end }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: {{ .Values.ingress.path }}
            pathType: {{ .Values.ingress.pathType }}
            backend:
              service:
                name: {{ include "openhamclock.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

- [ ] **Step 3: Verify default render omits the Ingress**

Run: `helm template test ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock`
Expected: no `kind: Ingress` in the output.

- [ ] **Step 4: Verify enabling renders the Ingress**

Run: `helm template test ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock --set ingress.enabled=true --set ingress.host=clock.example.com`
Expected: an `Ingress` named `test-openhamclock`, one rule for host `clock.example.com` routing `/` to Service `test-openhamclock` port `3000`.

- [ ] **Step 5: Verify TLS block**

Run: `helm template test ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock --set ingress.enabled=true --set ingress.tls\[0\].secretName=openhamclock-tls --set ingress.tls\[0\].hosts\[0\]=clock.example.com`
Expected: the Ingress has a `tls` block with `secretName: openhamclock-tls` and `hosts: [clock.example.com]`.

- [ ] **Step 6: Commit**

```bash
cd ~/src/github.com/adamancini/openhamclock-helm
git add charts/openhamclock/values.yaml charts/openhamclock/templates/ingress.yaml
git commit -m "feat: add optional Ingress"
```

---

## Task 8: NOTES.txt, chart README, repo README, final verification

**Files:**
- Create: `openhamclock-helm/charts/openhamclock/templates/NOTES.txt`
- Create: `openhamclock-helm/charts/openhamclock/README.md`
- Create: `openhamclock-helm/README.md`

**Interfaces:**
- Consumes: every value/template name defined in Tasks 1–7 (this task only writes documentation and the post-install notes; no new template logic).

- [ ] **Step 1: Create `templates/NOTES.txt`**

```
OpenHamClock has been deployed as {{ include "openhamclock.fullname" . }}.

{{- if .Values.ingress.enabled }}

Web UI:
  http{{ if .Values.ingress.tls }}s{{ end }}://{{ .Values.ingress.host }}{{ .Values.ingress.path }}
{{- else if eq .Values.service.type "LoadBalancer" }}

Web UI:
  Run the following to get the external IP, it may take a few minutes:
    kubectl get svc -w {{ include "openhamclock.fullname" . }} -n {{ .Release.Namespace }}
{{- else if eq .Values.service.type "NodePort" }}

Web UI:
  export NODE_PORT=$(kubectl get svc {{ include "openhamclock.fullname" . }} -n {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}")
  export NODE_IP=$(kubectl get nodes -o jsonpath="{.items[0].status.addresses[0].address}")
  echo "http://$NODE_IP:$NODE_PORT"
{{- else }}

Web UI (port-forward):
  kubectl port-forward svc/{{ include "openhamclock.fullname" . }} -n {{ .Release.Namespace }} 3000:{{ .Values.service.port }}
  Then open http://localhost:3000
{{- end }}

{{- if .Values.udpServices.enabled }}

WSJT-X / N1MM UDP:
  Service {{ include "openhamclock.fullname" . }}-udp is type {{ .Values.udpServices.type }}.
  Point WSJT-X/N1MM at that Service's address on the port(s) you enabled
  ({{- if .Values.udpServices.wsjtx.enabled }} wsjtx-udp={{ .Values.udpServices.wsjtx.port }}{{- end }}{{- if .Values.udpServices.n1mm.enabled }} n1mm-udp={{ .Values.udpServices.n1mm.port }}{{- end }}).
{{- else }}

WSJT-X / N1MM UDP integration is disabled (udpServices.enabled=false).
Set udpServices.enabled=true and udpServices.type to NodePort or
LoadBalancer to reach it from outside the cluster.
{{- end }}
```

- [ ] **Step 2: Create `charts/openhamclock/README.md`**

```markdown
# openhamclock

A Helm chart for deploying [OpenHamClock](https://github.com/accius/openhamclock),
a real-time amateur radio dashboard, to Kubernetes.

This is a personal/downstream chart — it packages the upstream
`ghcr.io/accius/openhamclock` image only. The optional companion
microservices from the upstream repo (dxspider-proxy, iturhfprop-service,
ohc-cluster, rig-bridge, etc.) are not included.

## Installing

\`\`\`bash
helm install my-openhamclock ./charts/openhamclock \
  --set config.env.CALLSIGN=K0CJH \
  --set config.env.LOCATOR=FN20
\`\`\`

## Configuration

| Key | Default | Description |
| --- | --- | --- |
| `replicaCount` | `1` | Number of pod replicas. |
| `image.repository` | `ghcr.io/accius/openhamclock` | Container image. |
| `image.tag` | `""` (uses `Chart.yaml` `appVersion`) | Image tag override. |
| `config.env` | see `values.yaml` | Non-secret env vars, rendered into a ConfigMap. Add any key from upstream's `.env.example`. |
| `secretEnv` | `{}` | Secret env vars (API keys, credentials), rendered into a Secret. |
| `existingSecret` | `""` | Use a pre-existing Secret instead of the one rendered from `secretEnv`. |
| `service.type` | `ClusterIP` | Type of the HTTP Service. |
| `service.port` | `3000` | HTTP Service port. |
| `ingress.enabled` | `false` | Create an Ingress for the web UI. |
| `persistence.enabled` | `true` | Persist `/data` via a PVC (else `emptyDir`). |
| `persistence.size` | `1Gi` | PVC size. |
| `udpServices.enabled` | `false` | Expose the WSJT-X/N1MM UDP listeners outside the cluster. Requires also setting `udpServices.type` to `NodePort` or `LoadBalancer`. |
| `udpServices.wsjtx.enabled` | `true` | Include the WSJT-X UDP port (2237) when `udpServices.enabled=true`. |
| `udpServices.n1mm.enabled` | `false` | Include the N1MM UDP port (12060) when `udpServices.enabled=true`. |

See `values.yaml` for the complete list, and upstream's `.env.example` for
every environment variable the app understands.
```

- [ ] **Step 3: Create the repo-level `README.md`**

```markdown
# openhamclock-helm

A personal Helm chart for deploying [OpenHamClock](https://github.com/accius/openhamclock)
to Kubernetes. Not affiliated with, and not intended for contribution back
to, the upstream project.

See [`charts/openhamclock/README.md`](charts/openhamclock/README.md) for
usage and configuration.
```

- [ ] **Step 4: Run `helm lint` on the finished chart**

Run: `helm lint ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock`
Expected: `0 chart(s) linted, 0 chart(s) failed`

- [ ] **Step 5: Run a kitchen-sink `helm template` render**

Run:
```bash
helm template test ~/src/github.com/adamancini/openhamclock-helm/charts/openhamclock \
  --set ingress.enabled=true \
  --set ingress.host=clock.example.com \
  --set udpServices.enabled=true \
  --set udpServices.n1mm.enabled=true \
  --set existingSecret=my-secret \
  --set persistence.enabled=false
```
Expected: renders without error; output contains `Deployment`, `Service` (x2: HTTP and UDP), `Ingress`, `ConfigMap`, `ServiceAccount`; does **not** contain `kind: Secret` (existingSecret set) or `kind: PersistentVolumeClaim` (persistence disabled); the Deployment's `data` volume is `emptyDir`; `envFrom.secretRef.name` is `my-secret`.

- [ ] **Step 6: Commit**

```bash
cd ~/src/github.com/adamancini/openhamclock-helm
git add charts/openhamclock/templates/NOTES.txt charts/openhamclock/README.md README.md
git commit -m "docs: add chart README, repo README, and install NOTES"
```

---

## Self-Review Notes

- **Spec coverage:** Deployment (Task 3), HTTP Service (Task 4), UDP Services (Task 5), Ingress (Task 7), Persistence (Task 6), ConfigMap/Secret (Task 2), ServiceAccount (Task 1), NOTES.txt (Task 8) — every resource from the design spec has a task. HPA/PDB/NetworkPolicy/`values.schema.json` are explicitly out of scope per the spec and have no task, as intended.
- **Placeholder scan:** no TBD/TODO markers; every step has literal file content and literal `helm template`/`helm lint` commands with concrete expected output.
- **Type/name consistency:** ConfigMap/Secret name `<fullname>-env` is identical across Tasks 2 and 3; PVC name `<fullname>-data` is identical across Tasks 3 and 6; container port names `http`/`wsjtx-udp`/`n1mm-udp` are identical between Task 3's Deployment and Tasks 4/5's Services; `openhamclock.*` helper names are used identically in every template.
