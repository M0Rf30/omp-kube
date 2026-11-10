# omp-kube

Run the [oh-my-pi](https://github.com/can1357/oh-my-pi) coding agent as a long-lived pod and work with it from anywhere: a browser or phone (Collab), your local `omp` (`omp join`), or the full TUI over `kubectl exec` / Tailscale SSH.

| Path | Contents |
|---|---|
| `image/` | Debian image: omp (prebuilt, checksum-verified), git, dtach, ripgrep, python3, sshd; `omp-pod` entrypoint, `omp-attach` |
| `charts/omp-agent/` | Helm chart: single-replica StatefulSet with a persistent `/home/agent` |

## How it works

- `omp-pod` starts `omp --continue` under dtach (a bare PTY holder: keys, images and colours pass through untouched), so after a pod restart the latest session resumes. Quitting omp restarts it after 5 s; press Enter instead to get a shell.
- `omp.config` is mounted as a settings overlay (`PI_CONFIG_FILES`). The default `collab.autoStart: control` makes every interactive session host a Collab room through `wss://my.omp.sh` (end-to-end encrypted, content-blind relay). The pod opens no inbound ports.
- `/home/agent` is a PVC: sessions, auth, memory, skills, repos and caches survive restarts. The PVC is retained on `helm uninstall`.
- Hardening: no ServiceAccount token, non-root, all capabilities dropped, NetworkPolicy denies ingress and egress to private/link-local/CGNAT ranges (in-cluster services, cloud metadata).

## Build and install

```sh
kubectl create namespace omp
kubectl -n omp create secret generic omp-keys --from-literal=ANTHROPIC_API_KEY=... --from-literal=GH_TOKEN=...
helm install omp oci://ghcr.io/m0rf30/charts/omp-agent -n omp --set omp.existingSecret=omp-keys
```

Images (`ghcr.io/m0rf30/omp-agent:<omp version>`, amd64/arm64) and charts are built by CI and signed with cosign (keyless). The chart `appVersion` is the omp version: bumping it on `main` builds the matching image. A `v*` tag publishes the chart with that version. Local build: `docker build --build-arg OMP_VERSION=18.18.6 -t omp-agent image`. OAuth/subscription logins: `omp auth export` locally, then `kubectl exec -i … -- omp auth import < auth.json` (or `/login` in the TUI).

## Staying on the latest omp (no interaction)

1. **Releases:** `.github/workflows/omp-bump.yml` checks omp's stable channel every 3 hours. On a new version it bumps `appVersion` and the chart patch version, commits `build: bump omp to X`, tags `vX.Y.Z` and dispatches CI, which builds/signs the image and publishes the chart.
2. **Rollout:** deploy with ArgoCD following the newest chart ([`deploy/argocd-application.yaml`](deploy/argocd-application.yaml), `targetRevision: "0.*"`, automated sync). New release → new image → pod restarts → `omp --continue` resumes the session. A turn running at that moment is interrupted.

Without ArgoCD, `omp.autoUpdate: true` installs the latest stable omp into the home volume on every pod start instead (picked up on the next restart; not signed by this repo's CI).

## Access

```sh
# TUI
kubectl -n omp exec -it omp-omp-agent-0 -c omp -- omp-attach   # detach: Ctrl-\
# Collab link for the browser / `omp join`
kubectl -n omp exec omp-omp-agent-0 -c omp -- omp collab list
kubectl -n omp exec omp-omp-agent-0 -c omp -- omp collab link <instanceId|pid>
```

Collab links change on `/new`, `/resume`, `/fork` and pod restarts; fetch a fresh one each time. A control link grants shell access to the pod: share only view links (`--view`) with others.

### Tailscale SSH (relay-independent path)

```sh
kubectl -n omp create secret generic omp-tailscale --from-literal=TS_AUTHKEY=tskey-auth-...
helm upgrade omp charts/omp-agent -n omp --reuse-values \
  --set ssh.enabled=true --set-string 'ssh.authorizedKeys[0]=ssh-ed25519 AAAA… you@laptop' \
  --set tailscale.enabled=true --set tailscale.authKeySecret.name=omp-tailscale
ssh -t agent@omp-agent omp-attach
```

sshd listens on `127.0.0.1` only; the Tailscale sidecar (userspace mode, state on the PVC) forwards tailnet port 22 to it.

## Bringing your omp customization

Keep a git repo laid out like `~/.omp/agent` and point `agentConfig.git.repo` at it. On every pod start an init container (`omp-pod --sync`) clones/hard-resets it and symlinks each entry of `agentConfig.links` into `~/.omp/agent`:

| Repo path | omp feature |
|---|---|
| `skills/`, `agents/`, `commands/`, `prompts/` | Skills, custom task agents, slash commands, prompt templates |
| `rules/`, `RULES.md`, `AGENTS.md` | Rules and user context |
| `SYSTEM.md`, `SYSTEM_TEMPLATE.md`, `APPEND_SYSTEM.md`, `TITLE_SYSTEM.md`, `WATCHDOG.md` | System prompt customization |
| `hooks/`, `tools/`, `extensions/` | Hooks, custom tools, extensions |
| `models.yml`, `mcp.json`, `lsp.json`, `themes/`, `keybindings.yml` | Providers/models, MCP, LSP, themes, keys |
| `config.yml` | Settings overlay (below `omp.config`) |

Missing entries are skipped. Also installed on every start: `agentConfig.marketplaces`, `agentConfig.plugins` (`name@marketplace`, npm, git) and `agentConfig.registrySkills` (Skillshare).

Never committed: credentials (`agent.db`, `secrets.yml`, `.env`) and state omp writes (sessions, memories, `managed-skills`). Put tokens in `omp.existingSecret` and reference them as `${VAR}` in `mcp.json`. Private repos over https use `GH_TOKEN` from that Secret. Changes take effect on the next pod restart (`kubectl rollout restart statefulset/…`), or `git -C ~/.omp-config pull` then `/reload`.

Bootstrap the repo from your laptop:

```sh
mkdir omp-config && cd omp-config && git init
cp -r ~/.omp/agent/{skills,agents,rules,AGENTS.md,models.yml} . 2>/dev/null   # whatever you have
```

## Values

See [`charts/omp-agent/values.yaml`](charts/omp-agent/values.yaml). Key ones:

| Key | Default | Purpose |
|---|---|---|
| `omp.args` | `[]` | Extra omp flags (`--model`, `--thinking`, `--max-cost`) |
| `omp.config` | Collab `control` via `my.omp.sh` | Settings overlay; same schema as `~/.omp/agent/config.yml` |
| `omp.existingSecret` | `""` | Secret exposed as env (provider keys, `GH_TOKEN`) |
| `agentConfig.git.repo` / `.links` | `""` / every customization point | Config repo mirrored into `~/.omp/agent` |
| `agentConfig.marketplaces` / `.plugins` / `.registrySkills` | `[]` | Installed on each start |
| `persistence.size` | `20Gi` | Home volume |
| `networkPolicy.blockedCIDRs` | RFC1918, CGNAT, link-local | Add `networkPolicy.extraEgress` for in-cluster git/registries |
| `ssh.enabled` / `tailscale.enabled` | `false` | TUI over the tailnet |

Not included: container builds inside the pod (no Docker socket; add a rootless BuildKit sidecar if you need it).
