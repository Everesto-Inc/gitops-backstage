# gitops-backstage

What ArgoCD deploys for Backstage (developer portal) — https://backstage.everesto.live

```
backstage-app (code) ──CI──▶ ghcr.io/everesto-inc/backstage:sha-xxxxxxx ──bumps tag──▶ values.yaml (here)
                                                                                    │
ArgoCD ◀────────────────────────────────────────────────────────────────────────────┘
  ├─ official Helm chart backstage 2.10.2 + values.yaml   → Deployment + Service (NodePort 30700)
  └─ manifests/external-secret.yaml                       → secret "backstage-secrets" from Infisical
     (all in namespace prod-backstage)

ESO ── ClusterSecretStore infisical-backstage (platform/) ── login in ns external-secrets ──▶ Infisical

Backstage ──SSL──▶ PostgreSQL on the GCP VM (database + user "backstage", one schema per plugin)
```

| File | What |
|---|---|
| `values.yaml` | our settings for the official chart (image, env, resources, placement, NodePort) |
| `manifests/external-secret.yaml` | ESO: copies 5 keys from Infisical into `backstage-secrets` |
| `platform/cluster-secret-store.yaml` | ClusterSecretStore `infisical-backstage`, usable only by ns `prod-backstage` — applied once by hand |
| `argocd/appproject.yaml` | ArgoCD project `backstage`: only namespace `prod-backstage`, only this repo + the chart — applied once by hand |
| `argocd/application.yaml` | the ArgoCD Application — applied once by hand |

## This repo is public — rules
- **No secret values, ever.** Passwords, client secrets, the database host → **Infisical**.
- **No internal IPs / hostnames** of the platform. The Traefik route (with private IPs) lives in
  the private ansible repo.
- Turn on GitHub **Settings → Code security → Secret scanning + Push protection** (free for
  public repos): GitHub then blocks a push that contains a token by mistake.

## One-time setup (in this order)

**1. Database on the GCP VM** (`sudo -u postgres psql` there, like for Infisical):
```sql
CREATE ROLE backstage LOGIN;
\password backstage          -- asks twice, hidden: never type a password inside a command
CREATE DATABASE backstage OWNER backstage;
```
The owner may create schemas → Backstage makes one per plugin inside `backstage`.

**2. GitHub OAuth App** (login) — GitHub → Settings → Developer settings → OAuth Apps → New
- Homepage URL: `https://backstage.everesto.live`
- Authorization callback URL: `https://backstage.everesto.live/api/auth/github/handler/frame`

**3. Infisical** (https://infisical.everesto.live)
- Project **`backstage`** → environment **`prod`** → root `/` with:
  `POSTGRES_HOST` (the DB's DNS name — must match the cert's CN/SAN) · `POSTGRES_PASSWORD` · `AUTH_GITHUB_CLIENT_ID` ·
  `AUTH_GITHUB_CLIENT_SECRET` · `POSTGRES_CA` (full PEM of the DB server cert, public part only)
- Put the project's **slug** (Settings) into `platform/cluster-secret-store.yaml` → `projectSlug`
- Organization → Access Control → **Identities** → create `k8s-eso` (Universal Auth) →
  add it to project `backstage` with role **Viewer** (read only) → copy **Client ID / Client Secret**

**4. Cluster** (on control-plane) — the only secrets ever typed by hand:
```bash
# Infisical login → platform namespace, NOT the app namespace (app users can't read it)
kubectl -n external-secrets create secret generic infisical-backstage-auth \
  --from-literal=clientId='<client id>' --from-literal=clientSecret='<client secret>'
kubectl apply -f platform/cluster-secret-store.yaml
kubectl get clustersecretstore infisical-backstage          # STATUS Valid

# App namespace + ghcr.io pull secret (pull secrets must live next to the pod)
kubectl create namespace prod-backstage
kubectl -n prod-backstage create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io --docker-username=Jacer7 --docker-password='<classic PAT, read:packages>'
```

**5. CI secret** — `backstage-app` repo → Settings → Secrets → Actions → `GITOPS_PAT`.

**6. ArgoCD** — project first, then the app:
`kubectl apply -n argocd -f argocd/appproject.yaml` · `kubectl apply -n argocd -f argocd/application.yaml`, then check:
`kubectl -n prod-backstage get externalsecret` → `SecretSynced` and `backstage-secrets` exists.

**7. Route + DNS** — ansible repo `files/traefik/conf.d/backstage.yml` → AWX
**sync_traefik_routes**; Cloudflare record `backstage` (proxied).

## Watch: the database is on another cloud
- **GCP egress:** query results travel GCP → OCI and count against the VM's ~1 GB/month free
  egress. Check GCP Billing after the first week; if it climbs, move the database next to Backstage.
- **Connections:** the VM allows ~100; each plugin pool is capped at 3 (app-config.production.yaml).
- **Latency:** first start runs all migrations across clouds → slower start (5-minute grace set).

## Who can log in
Only GitHub users listed as `User` in `backstage-app/examples/org.yaml`
(`metadata.name` = GitHub username). Everyone else is refused.
