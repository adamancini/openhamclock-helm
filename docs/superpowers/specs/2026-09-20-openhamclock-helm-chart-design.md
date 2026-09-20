# OpenHamClock Helm Chart — Design

## Purpose

Package [OpenHamClock](https://github.com/accius/openhamclock) (the amateur
radio dashboard web app) as a Helm chart so it can be deployed to a personal
Kubernetes cluster, mirroring the topology already defined in the upstream
project's `docker-compose.yml`. This is a personal/downstream chart, not a
contribution back to the `accius/openhamclock` project.

## Scope

In scope: the main OpenHamClock web app (image `ghcr.io/accius/openhamclock`)
as a single Deployment + supporting resources.

Out of scope: the optional companion microservices in the upstream repo
(`dxspider-proxy`, `iturhfprop-service`, `ohc-cluster`, `rig-bridge`,
`wsjtx-relay`, `rig-listener`, `dx-relay`, `fletcher`). Each already has its
own image/README upstream and would warrant its own chart (or subchart) later
if needed. Nothing in this design precludes adding them as dependency charts
in the future.

## Reference behavior (from upstream)

- Single container, `ghcr.io/accius/openhamclock:latest`, built from a
  `node:22-alpine` multi-stage Dockerfile with no `USER` directive (runs as
  root by default).
- Listens on:
  - `3000/tcp` — HTTP web UI + API
  - `2237/udp` — WSJT-X/JTDX relay (LAN protocol, points at the app from an
    operator's radio-logging software)
  - `12060/udp` — N1MM/DXLog contest logger relay (same LAN pattern)
- Health check: `GET /api/health`, used by both the Dockerfile
  `HEALTHCHECK` and `docker-compose.yml`.
- Writable data directory at `/data` (also falls back to `./data` or `/tmp`)
  for stats, settings sync, and push-notification subscription state.
- Configuration is entirely environment-variable driven — 70+ documented
  vars in `.env.example`, split roughly into: station info, display
  preferences, feature toggles, DX cluster settings, WSJT-X/N1MM UDP
  settings, and a long tail of optional integrations (OpenWeather, QRZ,
  HamQTH, AmbientWeather, Winlink, VAPID push keys, Space-Track, rotator
  control, etc.). New vars are added over time as features ship.

## Chart design

Standard Helm 3 chart at the repo root: `charts/openhamclock/`.

### Deployment

- `replicaCount` (default `1`) — this is a single-operator dashboard, not
  a horizontally-scaled service; no HPA/PDB in this design (YAGNI — the app
  has no shared state to coordinate across replicas anyway).
- Image repository/tag/pullPolicy configurable; defaults to
  `ghcr.io/accius/openhamclock` / chart's own `appVersion` / `IfNotPresent`.
- Container env built from two sources, both rendered as `envFrom`:
  - a `ConfigMap` populated from a flat `config.env` map in `values.yaml`
    (non-secret settings; pre-populated with the same essentials as
    `stack.env.example` — `CALLSIGN`, `LOCATOR`, `TZ`, `THEME`,
    `TIME_FORMAT`, `LAYOUT`, `SHOW_*` feature toggles, etc.)
  - a `Secret` populated from a `secretEnv` map in `values.yaml` (for
    `API_WRITE_KEY`, `METRICS_AUTH_KEY`, `OPENWEATHER_API_KEY`,
    `WSJTX_RELAY_KEY`, `VITE_AMBIENT_API_KEY`, QRZ/HamQTH credentials, VAPID
    keys, etc.) — or `existingSecret` naming a pre-created Secret instead,
    which takes precedence when set.

  A generic map (rather than one typed values field per variable) is used
  deliberately: the upstream app already exposes 70+ env vars and adds more
  as it ships features, and hand-mirroring each one in `values.yaml`/
  `templates/` would drift immediately. Users needing a var this chart
  doesn't call out by name just add it to `config.env` or `secretEnv`.
- Probes: liveness + readiness both `GET /api/health` on the HTTP port,
  matching the upstream Docker `HEALTHCHECK`.
- `resources` requests/limits — configurable, conservative defaults.
- `securityContext` — `allowPrivilegeEscalation: false`, drop `ALL`
  capabilities, `readOnlyRootFilesystem: false` (the app writes to
  `/app/data` and possibly elsewhere at runtime). Not forcing
  `runAsNonRoot: true` by default since the upstream image has no `USER`
  directive and hasn't been verified to run as non-root; left as an
  overridable value for users who verify it works for their setup.
- Volume mount at `/data` backed by the persistence volume (PVC or
  `emptyDir`, see below).

### Service (HTTP)

- `ClusterIP`, port `3000/tcp`, always created — this is how the Ingress
  (or a user's own port-forward/LoadBalancer override) reaches the app.

### UDP services (WSJT-X / N1MM)

- Disabled by default (`udpServices.enabled: false`). A `ClusterIP`
  Service cannot be reached by devices outside the cluster (e.g. WSJT-X
  running on an operator's desktop), so a meaningful default here is
  either a `NodePort` or a `LoadBalancer` — both of which have
  infrastructure implications (reserved node ports, or an actual cloud LB
  and its cost) that shouldn't be forced on users who don't need this
  integration. Users who want it enable it and pick `udpServices.type`
  themselves.
- When enabled, renders one Service exposing `2237/udp` (WSJT-X, toggle:
  `udpServices.wsjtx.enabled`) and/or `12060/udp` (N1MM, toggle:
  `udpServices.n1mm.enabled`), each independently switchable since a user
  may only use one of the two integrations.

### Ingress

- Optional (`ingress.enabled: false` by default), standard single-host +
  TLS template pointing at the HTTP Service.

### Persistence

- PVC for `/data`, enabled by default (`persistence.enabled: true`,
  `1Gi`, `storageClass` configurable) so stats/settings survive pod
  restarts. Falls back to `emptyDir` when disabled.

### Other resources

- `ServiceAccount` — created by default, no RBAC rules (the app never
  talks to the Kubernetes API).
- `NOTES.txt` — post-install instructions (how to reach the UI given
  ingress/service settings, how to enable UDP services).

## Testing / verification

- `helm lint` on the chart.
- `helm template` rendered for: defaults, ingress enabled, persistence
  disabled, UDP services enabled — checked for valid output and expected
  resources present/absent.
- No cluster-based install test in this pass (no cluster targeted yet by
  this design); can be added later via `ct` or `helm unittest` if this
  chart grows.

## Out of scope / explicitly deferred

- Companion microservice charts (see Scope).
- HPA, PodDisruptionBudget, NetworkPolicy — no current need for a
  single-operator dashboard.
- `values.schema.json` — deferred until the chart's values surface
  stabilizes.
