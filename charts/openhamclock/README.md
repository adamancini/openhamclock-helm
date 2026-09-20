# openhamclock

A Helm chart for deploying [OpenHamClock](https://github.com/accius/openhamclock),
a real-time amateur radio dashboard, to Kubernetes.

This is a personal/downstream chart — it packages the upstream
`ghcr.io/accius/openhamclock` image only. The optional companion
microservices from the upstream repo (dxspider-proxy, iturhfprop-service,
ohc-cluster, rig-bridge, etc.) are not included.

## Installing

```bash
helm install my-openhamclock ./charts/openhamclock \
  --set config.env.CALLSIGN=K0CJH \
  --set config.env.LOCATOR=FN20
```

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
| `udpServices.type` | `NodePort` | Service type for the UDP listeners (`NodePort` or `LoadBalancer`). |
| `udpServices.wsjtx.enabled` | `true` | Include the WSJT-X UDP port (2237) when `udpServices.enabled=true`. |
| `udpServices.n1mm.enabled` | `false` | Include the N1MM UDP port (12060) when `udpServices.enabled=true`. |

See `values.yaml` for the complete list, and upstream's `.env.example` for
every environment variable the app understands.
