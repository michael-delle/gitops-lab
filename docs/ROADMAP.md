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
- Use Helm both ways: consume upstream charts for infrastructure, and author a first-party chart.
- Keep the repository safe to publish: no credentials, kubeconfigs or machine-specific details.

## Non-goals

- Production-grade clusters (HA, multi-node, cloud load balancers).
- Observability stack (Prometheus/Grafana) — stretch goal after milestone 11.
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
| `whoami.flux.localhost` / `whoami-staging.flux.localhost` | whoami (own Helm chart), via Flux |
| `whoami.argo.localhost` / `whoami-staging.argo.localhost` | whoami (own Helm chart), via Argo CD |
| `argocd.argo.localhost` | Argo CD UI |

Caddy config lives in `dev/caddy/gitops-lab.caddy` (one wildcard block per cluster) and is
symlinked into the local-edge proxy's `projects/` directory.

## Repository layout

```
gitops-lab/
├── README.md
├── lab                         # task runner: ./lab up-flux / up-argo / down / status / validate / commit
├── dev/caddy/gitops-lab.caddy
├── charts/                     # TOOL-NEUTRAL: first-party Helm charts
│   └── whoami/                 # Chart.yaml, values.yaml, templates/
├── apps/                       # TOOL-NEUTRAL
│   ├── podinfo/                # Kustomize
│   │   ├── base/
│   │   └── overlays/{staging,production}/
│   └── whoami/                 # per-environment values for charts/whoami
│       └── values-{staging,production}.yaml
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

**Rule:** nothing under `apps/`, `charts/` or `infrastructure/` may reference Flux or Argo CD. Helm
_values_ are neutral and live in `infrastructure/`; Helm _release definitions_ (`HelmRelease`,
`Application`) are tool-specific and live in `clusters/<tool>/`.

## Kustomize and Helm

Both packaging styles are used deliberately, each where it fits best:

| Workload | Packaging | Why |
|---|---|---|
| Upstream infrastructure (Traefik, Sealed Secrets, …) | Official Helm charts + values in `infrastructure/` | How vendors ship software; upgrades are a version bump |
| podinfo | Kustomize base + overlays | Plain YAML, patch-based environments, no templating |
| whoami | First-party Helm chart in `charts/whoami` | Learn chart authoring: templates, helpers, values, `helm lint` |

- Charts come from the projects' own repositories. Bitnami-packaged charts are avoided: most
  moved behind a paid subscription in 2025.
- A chart is a reusable package; environment configuration is not part of it. Per-environment
  values live in `apps/whoami/`, not inside the chart.
- The same chart is delivered differently by each tool, which is one of the key comparisons:
  Flux's `HelmRelease` performs a real Helm release (`helm ls`, `helm history` and rollback
  work), while Argo CD renders the chart with `helm template` and applies plain manifests, so
  Helm itself has no record of the release.

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
- GitHub secret scanning with push protection (free for public repos) as a server-side backstop.
- If a real secret is ever pushed: **rotate it immediately** — rewriting history is not a fix.

### Publishing checks

The repository is public from milestone 1. Run these before the first push, and again at
milestone 11:

- [ ] Full-history gitleaks scan
- [ ] `git log --format='%an <%ae>' | sort -u` shows only the noreply identity
- [ ] `grep` for IP addresses and non-`localhost` hostnames
- [ ] Secret scanning push protection enabled (Settings → Code security)

## Validation

`.github/workflows/validate.yaml` runs on pull requests and pushes to `main`, sharing one script
with `./lab validate`:

1. `kustomize build` on every overlay and every `clusters/*` entry point.
2. `helm lint` on every chart in `charts/`, and `helm template` with each environment's values.
3. `kubeconform` on all rendered output, including Flux and Argo CD CRD schemas.
4. gitleaks.

From milestone 1 onward, every change lands via branch → pull request → green CI → merge,
enforced by a branch ruleset on `main` once CI exists (milestone 5).

## Milestones

Each milestone ends in a working, committed state.

### 0 — Repo hygiene
- [ ] `git init`, repo-local noreply `user.email`
- [ ] `.gitignore`, gitleaks pre-commit hook
- **Done when:** a commit containing a fake token is blocked by the hook.

### 1 — Flux cluster & bootstrap
- [ ] Create public GitHub repo `gitops-lab` (publishing checks first)
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

### 4 — Your own Helm chart (Flux)
- [ ] `charts/whoami`: `Chart.yaml`, `values.yaml`, `_helpers.tpl`, Deployment, Service, Ingress
- [ ] `apps/whoami/values-{staging,production}.yaml` (replicas, hostname)
- [ ] `HelmRelease` per environment, sourcing the chart from the Git repository
- [ ] `helm lint` and `helm template` pass locally
- **Done when:** both whoami hostnames load, and `helm ls -A` / `helm history` show the releases.

### 5 — CI validation
- [ ] `./lab validate` + GitHub Actions workflow
- [ ] Branch ruleset on `main`: require a pull request and passing CI
- **Done when:** a PR with a deliberately broken manifest or chart fails CI; fixing it goes green.

### 6 — Secrets with SOPS + age (Flux)
- [ ] age key, `.sops.yaml`, in-cluster decryption key
- [ ] Encrypted demo secret consumed by podinfo
- **Done when:** the secret exists decrypted in-cluster, and only ciphertext is in Git.

### 7 — Flux image automation
- [ ] `ImageRepository`, `ImagePolicy` (semver range), `ImageUpdateAutomation`
- **Done when:** Flux commits a podinfo tag bump to Git and the new version rolls out.

### 8 — Argo CD cluster
- [ ] k3d cluster `argo`, node joined to `dev-edge`
- [ ] Argo CD installed from `clusters/argo`, then managing itself
- [ ] Root app (App-of-Apps); UI at `argocd.argo.localhost` (insecure mode behind Caddy)
- **Done when:** the UI loads via Caddy and Argo CD shows itself Synced/Healthy.

### 9 — Same platform on Argo CD
- [ ] Traefik from the same `infrastructure/traefik/values.yaml`
- [ ] podinfo overlays via an `ApplicationSet`; ordering with sync waves
- [ ] whoami from the same `charts/whoami` and `apps/whoami` values
- **Done when:** all `*.argo.localhost` app hostnames load; drift is self-healed; `helm ls -A`
  shows no whoami release (and you can explain why).

### 10 — Argo CD secrets & image updates
- [ ] Sealed Secrets controller; demo secret as a `SealedSecret`
- [ ] Argo CD Image Updater with Git write-back
- **Done when:** same outcomes as milestones 6 and 7, on the Argo CD cluster.

### 11 — Polish & publish
- [ ] README: pitch, CI badge, Mermaid diagram, quickstart, roadmap status, screenshots
- [ ] `docs/flux-vs-argocd.md` comparison write-up, including how each tool handles Helm
- [ ] ADRs for key decisions (Traefik, SOPS vs Sealed Secrets, namespaces as environments,
      Kustomize vs Helm for first-party apps)
- [ ] Publishing checks re-run
- **Done when:** a newcomer can run `./lab up-flux` from the README and understand the project
  from the README and docs alone.

### Stretch
- Observability: kube-prometheus-stack via GitOps, dashboards and alerts for reconciliation failures.
- Gateway API instead of Ingress.
- Progressive delivery: Flagger vs Argo Rollouts.
