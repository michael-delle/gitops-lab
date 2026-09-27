# gitops-lab — Design & Roadmap

> One platform, delivered two ways: the same apps and infrastructure deployed to local
> Kubernetes clusters by **Flux** and by **Argo CD**, to compare the two GitOps models
> hands-on.

_Design date: 2026-09-27_

## Goals

- Learn Flux and Argo CD by building a small but realistic GitOps platform with each.
- Keep app and infrastructure definitions **tool-neutral**, so both tools deliver identical inputs.
- Cover the topics that matter in real teams: bootstrapping, ordering and health, environments,
  secrets in Git, image automation, and CI validation.
- Keep the repository safe to publish: no credentials, kubeconfigs or machine-specific details.

## Non-goals

- Production-grade clusters (HA, multi-node, cloud load balancers).
- Observability stack (Prometheus/Grafana) — stretch goal after milestone 10.
- Real secrets. The demo secret is a deliberately fake value.

## Architecture

### Clusters

| Cluster | Tool | Created with |
|---|---|---|
| `flux` | Flux | k3d, 1 server node, built-in Traefik disabled |
| `argo` | Argo CD | k3d, 1 server node, built-in Traefik disabled |

- Traefik is installed **by the GitOps tool** from its Helm chart, so the ingress layer is
  managed from Git like everything else.
- Environments `staging` and `production` are **namespaces** within each cluster.

### Traffic path

```
https://podinfo.flux.localhost
  → local-edge Caddy (TLS via local CA, docker network `dev-edge`)
  → k3d node container :80 (reachable only inside dev-edge)
  → Traefik → Ingress → Service → Pod
```

- The cluster node container joins the shared `dev-edge` Docker network; Caddy proxies to it by
  container name. **No host ports 80/443 are claimed by either cluster.**
- Traefik's Service is exposed on the node via k3s's built-in service load balancer.
- The Kubernetes API stays on k3d's default random localhost port.
- TLS terminates at Caddy; everything behind it is plain HTTP (Argo CD server runs in
  insecure mode for this reason).

### Hostnames

| Hostname | Target |
|---|---|
| `podinfo.flux.localhost` | podinfo, production, via Flux |
| `podinfo-staging.flux.localhost` | podinfo, staging, via Flux |
| `podinfo.argo.localhost` | podinfo, production, via Argo CD |
| `podinfo-staging.argo.localhost` | podinfo, staging, via Argo CD |
| `argocd.argo.localhost` | Argo CD UI |

Caddy config lives in `dev/caddy/gitops-lab.caddy` (one wildcard block per cluster) and is
symlinked into the local-edge proxy's `projects/` directory.

## Repository layout

```
gitops-lab/
├── README.md
├── Makefile                    # up-flux / up-argo / down / status / validate
├── dev/caddy/gitops-lab.caddy
├── apps/                       # TOOL-NEUTRAL
│   └── podinfo/
│       ├── base/
│       └── overlays/{staging,production}/
├── infrastructure/             # TOOL-NEUTRAL: Helm values and plain manifests
│   └── traefik/values.yaml
├── clusters/
│   ├── flux/                   # flux-system/, infrastructure.yaml, apps.yaml, secrets
│   └── argo/                   # Argo CD self-management, root app, ApplicationSets, secrets
├── docs/
│   ├── ROADMAP.md              # this file
│   ├── architecture.md
│   ├── flux-vs-argocd.md
│   └── adr/
└── .github/workflows/validate.yaml
```

**Rule:** nothing under `apps/` or `infrastructure/` may reference Flux or Argo CD. Helm
_values_ are neutral and live in `infrastructure/`; Helm _release definitions_ (`HelmRelease`,
`Application`) are tool-specific and live in `clusters/<tool>/`.

## Secrets & safety

### Secrets in Git

| | Flux | Argo CD |
|---|---|---|
| Approach | SOPS + age | Sealed Secrets |
| Committed | SOPS-encrypted YAML; `.sops.yaml` with the **public** age recipient | `SealedSecret` resources |
| Private key | In-cluster (`flux-system/sops-age`) and `~/.config/sops/age/keys.txt` — never in the repo | In-cluster controller only |

Encrypted secrets are cluster-specific and live under `clusters/<tool>/`; apps reference
them by name only. External Secrets Operator is discussed in `flux-vs-argocd.md` as the
common production alternative.

### Guardrails

- `.gitignore`: `kubeconfig*`, `*.agekey`, `keys.txt`, `.env*`, `*.pem`, `*.key`,
  Sealed Secrets key backups, `.DS_Store`, `.idea/`.
- gitleaks as a pre-commit hook and in CI.
- Repo-local `user.email` set to the GitHub noreply address.
- `flux bootstrap` uses a fine-grained GitHub token scoped to this repo only, short expiry,
  supplied via environment variable. Flux then uses an in-cluster deploy key (read-write,
  required for image automation).
- Argo CD initial admin password is read once, changed, and the initial secret deleted.
- Only `*.localhost` hostnames appear in the repo — no LAN IPs, tailnet names or employer names.

### Before going public

- [ ] Full-history gitleaks scan
- [ ] `git log --format='%an <%ae>' | sort -u` shows only the noreply identity
- [ ] `grep` for IP addresses and non-`localhost` hostnames
- [ ] Switch repository visibility to public

## Validation

`.github/workflows/validate.yaml` runs on pull requests and pushes to `main`, sharing one script
with `make validate`:

1. `kustomize build` on every overlay and every `clusters/*` entry point.
2. `kubeconform` on the rendered output, including Flux and Argo CD CRD schemas.
3. gitleaks.

From milestone 1 onward, every change lands via branch → pull request → green CI → merge.

## Milestones

Each milestone ends in a working, committed state.

### 0 — Repo hygiene
- [ ] `git init`, repo-local noreply `user.email`
- [ ] `.gitignore`, gitleaks pre-commit hook
- **Done when:** a commit containing a fake token is blocked by the hook.

### 1 — Flux cluster & bootstrap
- [ ] Create private GitHub repo `gitops-lab`
- [ ] k3d cluster `flux` (Traefik disabled), node joined to `dev-edge`
- [ ] `flux bootstrap github` into `clusters/flux`
- **Done when:** `flux check` passes and `flux get kustomizations` shows `flux-system` Ready.

### 2 — Infrastructure via Flux
- [ ] Traefik `HelmRelease` using `infrastructure/traefik/values.yaml`
- [ ] `infrastructure` Kustomization with health checks; `apps` Kustomization `dependsOn` it
- [ ] Caddy wildcard block for `*.flux.localhost`
- **Done when:** Traefik is Ready and `https://anything.flux.localhost` returns Traefik's 404.

### 3 — podinfo, two environments
- [ ] `apps/podinfo` base + `staging` / `production` overlays (different replicas, hostnames, UI message)
- [ ] Drift demo: manual `kubectl` change is reverted; deleted file is pruned
- **Done when:** both podinfo hostnames load, showing different environment messages.

### 4 — CI validation
- [ ] `make validate` + GitHub Actions workflow
- **Done when:** a PR with a deliberately broken manifest fails CI; fixing it goes green.

### 5 — Secrets with SOPS + age (Flux)
- [ ] age key, `.sops.yaml`, in-cluster decryption key
- [ ] Encrypted demo secret consumed by podinfo
- **Done when:** the secret exists decrypted in-cluster, and only ciphertext is in Git.

### 6 — Flux image automation
- [ ] `ImageRepository`, `ImagePolicy` (semver range), `ImageUpdateAutomation`
- **Done when:** Flux commits a podinfo tag bump to Git and the new version rolls out.

### 7 — Argo CD cluster
- [ ] k3d cluster `argo`, node joined to `dev-edge`
- [ ] Argo CD installed from `clusters/argo`, then managing itself
- [ ] Root app (App-of-Apps); UI at `argocd.argo.localhost` (insecure mode behind Caddy)
- **Done when:** the UI loads via Caddy and Argo CD shows itself Synced/Healthy.

### 8 — Same platform on Argo CD
- [ ] Traefik from the same `infrastructure/traefik/values.yaml`
- [ ] podinfo overlays via an `ApplicationSet`; ordering with sync waves
- **Done when:** both `*.argo.localhost` podinfo hostnames load; drift is self-healed.

### 9 — Argo CD secrets & image updates
- [ ] Sealed Secrets controller; demo secret as a `SealedSecret`
- [ ] Argo CD Image Updater with Git write-back
- **Done when:** same outcomes as milestones 5 and 6, on the Argo CD cluster.

### 10 — Polish & publish
- [ ] README: pitch, CI badge, Mermaid diagram, quickstart, roadmap status, screenshots
- [ ] `docs/flux-vs-argocd.md` comparison write-up
- [ ] ADRs for key decisions (Traefik, SOPS vs Sealed Secrets, namespaces as environments)
- [ ] "Before going public" checklist complete
- **Done when:** the repo is public and a newcomer can run `make up-flux` from the README.

### Stretch
- Observability: kube-prometheus-stack via GitOps, dashboards and alerts for reconciliation failures.
- Gateway API instead of Ingress.
- Progressive delivery: Flagger vs Argo Rollouts.
