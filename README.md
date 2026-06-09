# ellington-web-manifests

Kubernetes manifests for [`siege-analytics/ellington-web`](https://github.com/siege-analytics/ellington-web). Watched by ArgoCD on the cyberpower cluster.

Companion epic: [ellington-systems#12](https://github.com/siege-analytics/ellington-systems/issues/12). This repo owns the deployment substrate (sub-1).

## Layout

```
base/                 — k8s resources for the ellington namespace (watched by ArgoCD)
  namespace.yaml      — the `ellington` namespace
  configmap.yaml      — app env (sub-2 will expand)
  deployment.yaml     — web Deployment (placeholder image until sub-2)
  service.yaml        — ClusterIP Service
  ingressroute.yaml   — traefik IngressRoute with authentik-forwardauth middleware
  certificate.yaml    — cert-manager Certificate for ellington.siegeanalytics.com

tekton/               — Tekton CI pipeline (lives in tekton-pipelines namespace)
  pipeline.yaml       — Pipeline (clone → buildx-bake → restart Deployment)
  trigger.yaml        — TriggerBinding + TriggerTemplate + EventListener for GitHub push events

argocd/               — ArgoCD Application (lives in argocd namespace)
  application.yaml    — points at base/ on main; auto-sync + self-heal enabled
```

## Conventions

- **Image registry:** `cyberpower:32000/ellington-web:<branch>-<short-sha>`
- **Domain:** `ellington.siegeanalytics.com` → DNS A `74.196.224.156` (Dreamhost)
- **TLS:** cert-manager `ClusterIssuer/letsencrypt-prod`; Secret `ellington-tls`
- **Auth:** traefik `authentik-forwardauth` middleware (cluster convention; every IngressRoute routes through it)
- **GPU pods** (sub-4 audio + sub-5 LLM): nodeSelector `node-role.kubernetes.io/gpu: "true"`; web tier has no nodeSelector

## Bootstrap (one-time operator steps)

These are not in ArgoCD's watch loop and must be done manually the first time:

### 1. Verify DNS propagation

```bash
dig +short ellington.siegeanalytics.com @8.8.8.8
# expected: 74.196.224.156
```

cert-manager's HTTP-01 challenge needs this to resolve from LetsEncrypt's validation servers. Wait 5-60 min after the A record was added if it doesn't resolve immediately.

### 2. Configure Authentik for `ellington.siegeanalytics.com`

Cluster convention is to route every public IngressRoute through Authentik forwardauth (see `electinfo/*` IngressRoutes for the canonical pattern). Before traffic can flow:

1. Log into Authentik admin (cluster-internal, see Authentik's IngressRoute)
2. Create a new **Application** named `ellington-web`
3. Create a new **Provider** of type Proxy → Forward auth (single application) with:
   - External host: `https://ellington.siegeanalytics.com`
   - Authentication flow: default-authentication-flow
   - Authorization flow: default-provider-authorization-implicit-consent
4. Bind the Provider to the Application
5. (Optional) Add a Group binding restricting who can access

The `authentik-forwardauth` middleware in the `traefik` namespace handles the actual header-forwarding; no changes to it are needed.

### 3. Apply the ArgoCD Application (once)

```bash
kubectl apply -f argocd/application.yaml
```

After this, ArgoCD watches `base/` on `main` and applies any changes automatically. The Tekton Pipeline in `tekton/` is applied by ArgoCD too if you add it under `base/`, OR applied manually:

```bash
kubectl apply -f tekton/
```

(Recommendation: keep `tekton/` outside ArgoCD's watch for now so the Pipeline can be revised without triggering full app re-sync.)

### 4. Wire the GitHub webhook

The `tekton/` manifests (Pipeline, EventListener, TriggerBinding, TriggerTemplate, IngressRoute, Certificate) live outside ArgoCD's watch. Apply them once:

```bash
kubectl apply -f tekton/
```

This brings up:
- `el-ellington-web-listener` service in `tekton-pipelines` ns
- `ellington-web-webhook` IngressRoute in `traefik` ns at `https://ellington-web.webhook.elect.info` (DNS wildcard `*.webhook.elect.info` already resolves to the cluster)
- cert-manager Certificate `ellington-web-webhook-electinfo-tls` (LetsEncrypt prod)

Then register the webhook on GitHub: `siege-analytics/ellington-web → Settings → Webhooks → Add webhook`:
- **Payload URL:** `https://ellington-web.webhook.elect.info`
- **Content-Type:** `application/json`
- **SSL verification:** Enable
- **Events:** Just the push event
- **Secret:** _(none yet — see [#6](https://github.com/siege-analytics/ellington-web-manifests/issues/6) for HMAC hardening sweep)_

Once registered, pushing to `main` on `siege-analytics/ellington-web` fires a `PipelineRun` visible in `kubectl -n tekton-pipelines get pr`.

### 5. Provision the PostGIS database (sub-2c)

Ellington uses a dedicated DB on the cluster's existing `default/db-postgis-master` instance — same pattern as `authentik`, `cms`, `electinfo`, `nominatim` etc.

```bash
# Generate a strong password
PW=$(python3 -c 'import secrets; print(secrets.token_urlsafe(24))')
echo "Save this to a password manager BEFORE running the next steps: $PW"

# Create role + DB + extension
POD=db-postgis-master-0
kubectl -n default exec $POD -c postgis -- psql -U postgres -c \
  "CREATE ROLE ellington_web WITH LOGIN PASSWORD '$PW';"
kubectl -n default exec $POD -c postgis -- psql -U postgres -c \
  "CREATE DATABASE ellington_web OWNER ellington_web;"
kubectl -n default exec $POD -c postgis -- psql -U postgres -d ellington_web -c \
  "CREATE EXTENSION IF NOT EXISTS postgis;"

# Apply the cluster Secret consumed by base/deployment.yaml
kubectl -n ellington create secret generic ellington-web-postgres \
  --from-literal=SQL_PASSWORD="$PW"

# Verify
kubectl -n default exec $POD -c postgis -- env PGPASSWORD="$PW" \
  psql -U ellington_web -d ellington_web -h localhost \
  -tAc 'SELECT current_user, current_database(), postgis_version();'
# expected: ellington_web|ellington_web|3.x ...
```

The Secret shape is documented in `base/secret-postgres.yaml.example` (committed as documentation only — never put the live password in git). The Deployment's `envFrom` picks up `SQL_PASSWORD` from the Secret alongside the non-secret `SQL_*` keys from `base/configmap.yaml`.

To rotate the password: regenerate, `ALTER ROLE ellington_web WITH PASSWORD '<new>';`, then `kubectl create secret generic ellington-web-postgres --from-literal=SQL_PASSWORD='<new>' --dry-run=client -o yaml | kubectl apply -f -`.

## Day-to-day

After bootstrap, all changes flow through git → ArgoCD:

1. Edit a manifest in `base/`
2. Push to `main` (or open a PR + merge)
3. ArgoCD picks up the change within minutes and applies it

ArgoCD `auto-prune` is enabled — manifests removed from this repo are deleted from the cluster. ArgoCD `self-heal` is enabled — out-of-band cluster edits are reverted.

## Verifying the deploy

```bash
# After ArgoCD syncs once:
kubectl -n ellington get all
kubectl -n ellington get certificate ellington-tls -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
# expected: True
curl -sS https://ellington.siegeanalytics.com -I
# expected: HTTP/2 200 (or 302 to authentik login if SSO is configured)
```

If the Certificate stays `Ready=False` > 5 min, check `kubectl describe certificate ellington-tls -n ellington` for the cert-manager challenge events; most often it's DNS propagation.

## License

Apache 2.0. See [LICENSE](LICENSE).
