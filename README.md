# Hermes Agent on x13

This chart runs the official [Hermes Agent](https://github.com/NousResearch/hermes-agent)
image as one Telegram gateway on the homelab k3s node `x13`. It uses the same
GitHub Actions → SSH tunnel → k3s deployment route as Penny. There is no
Service, Ingress, or public listener. Telegram uses outbound long polling.

## Deployment shape

- The pod selects `homelab.morjoff.com/workload=home` and tolerates
  `dedicated=home:NoSchedule`. If x13 is offline it stays pending.
- `/opt/data` is a 5 GiB `local-path` PVC on x13. It contains Hermes config,
  sessions, skills, memories, credentials written by Hermes, and its SQLite
  database. Back it up outside the cluster; deleting the PVC deletes its data.
- `managedConfig` is rendered to a ConfigMap mounted at `/etc/hermes/config.yaml`
  and enabled through Hermes' managed scope (`HERMES_MANAGED_DIR=/etc/hermes`).
  Pinned keys win over `/opt/data/config.yaml` and the environment on every
  start, so chart changes apply without touching the PVC. Keys not listed there
  stay editable and persist in `/opt/data/config.yaml`.
- A native `split-proxy` sidecar listens on loopback and is an HTTP CONNECT
  proxy: only `splitProxy.domains` (default `openrouter.ai`) go through x13's
  SOCKS chain; all other traffic goes direct. It uses the Hermes image, so no
  extra proxy image is needed.
- Terminal and code tools run inside the Hermes pod using the `local` backend.
  The pod has no host mounts, Docker socket, host networking, or Kubernetes
  API token. Tools can still access files and environment variables inside
  their own pod, including the gateway credentials.
- The official image starts as root for s6 initialization, then runs the
  supervised Hermes service as UID 10000. Its startup hook fixes ownership of
  the data directory for that UID.

The chart defaults to `nousresearch/hermes-agent:v0.23.0`. Change
`HERMES_IMAGE_TAG` in the production GitHub environment to upgrade to a tested
release. The official image supports amd64 and arm64; no image build is needed.

## GitHub production environment

Create the `production` environment in this repository and add these secrets:

| Secret | Purpose |
| --- | --- |
| `KUBECONFIG` | k3s kubeconfig pointing at `https://127.0.0.1:6443` |
| `CI_DEPLOY_SSH_KEY` | Private key for homelab's `ci-deploy` SSH account |
| `K3S_HOST` | Cloud k3s server reachable from GitHub Actions |
| `TELEGRAM_BOT_TOKEN` | New bot's token from BotFather; do not reuse Penny's token |
| `DEEPSEEK_API_KEY` | DeepSeek API key; used as the selectable fallback model |
| `OPENROUTER_API_KEY` | OpenRouter API key; the default provider |

Optional environment variables:

| Variable | Purpose |
| --- | --- |
| `TELEGRAM_ALLOWED_USERS` | Comma-separated numeric Telegram user IDs authorized to use Hermes. Required; not sensitive, so it lives as a variable |
| `TELEGRAM_PROXY` | Override the default `socks5://192.168.1.100:10808` Xray chain endpoint if it changes |
| `OPENROUTER_PROXY` | *Fallback only.* Used when `splitProxy.enabled: false`; the sidecar overrides it. Defaults to the Xray chain endpoint; `direct` disables proxying |
| `NO_PROXY` | *Fallback only.* Exclusions for the env-based proxy; the sidecar sets its own loopback-only list. Supports domain suffixes, CIDRs, `*.` wildcards, and `host:port` |
| `HERMES_IMAGE_TAG` | Official image tag; defaults to `v0.23.0` |

The workflow recreates the `hermes-env` Kubernetes Secret before Helm installs
the chart: it deletes the old Secret and creates it from the workflow inputs, so
keys removed from GitHub do not linger. No secret values are placed in chart
values. A push to
`master` runs chart checks and deploys after they pass. `workflow_dispatch`
runs the same checks before deploying. Secrets are scoped to this repository's
production environment; Penny's GitHub environment does not share them.
Telegram and LLM API traffic (OpenRouter and DeepSeek) use x13's Xray `10808`
chain endpoint because their direct routes are blocked at home. Hermes applies
`OPENROUTER_PROXY` as `HTTPS_PROXY`/`HTTP_PROXY` for LLM calls and sets
`NO_PROXY=127.0.0.1,localhost,::1`, so only loopback bypasses the proxy.

## First deployment and checks

1. Create a separate bot with BotFather and populate the production environment.
2. Push the chart to `master` or run the Deploy workflow manually.
3. Check `kubectl -n hermes get pod -o wide` and confirm the node is `x13`.
4. Check `kubectl -n hermes logs deployment/hermes -c hermes --tail=100` for
   Telegram connection and model configuration errors.
5. Send the bot a DM from an allowed user, then from an unlisted account.
   Only the allowed user should get an agent response.

Run `helm lint .` and `helm template hermes . --namespace hermes` locally to
inspect the chart before deployment.

The chart does not add a health endpoint. Hermes' official image supervises its
gateway process inside the container, so Kubernetes may report a running pod
while Telegram is disconnected. Check the gateway logs and perform the DM test
after first deployment and image upgrades.

## Storage and configuration

Hermes stores its state database under `/opt/data` in SQLite WAL mode. The
`local-path` PVC uses x13's local filesystem. Do not copy a live `state.db`
alone as a backup; use a consistent SQLite backup or stop Hermes before copying
the volume. The volume also contains sensitive conversations and credentials.

`managedConfig` in `values.yaml` is the source of truth for the keys it lists.
It renders to the `hermes-managed` ConfigMap, mounted read-only at
`/etc/hermes`; `HERMES_MANAGED_DIR` enables managed scope. A checksum
annotation on the pod makes a config change trigger a rollout, so a Helm upgrade
applies it immediately.

```yaml
managedConfig:
  model:
    provider: openrouter
    base_url: https://openrouter.ai/api/v1
  terminal:
    backend: local

# Seeded only when /opt/data/config.yaml does not exist.
seedConfig:
  model:
    default: nvidia/nemotron-3.5-super-vl:free
```

Managed keys are leaf-level: pinning `model.provider` and `model.base_url`
does not freeze `model.default`. Hermes refuses `hermes config set` on a
pinned key, and managed values win over the PVC config and the environment.
`model.default` is deliberately **not** managed, so the CLI can change it.
Everything else remains editable on the volume and survives redeploys. Because
the managed file is world-readable by design, API keys stay in the
`hermes-env` Secret rather than the managed `.env` layer.

Verify the policy from inside the pod:

```bash
kubectl -n hermes exec deploy/hermes -c hermes -- hermes config
kubectl -n hermes exec deploy/hermes -c hermes -- hermes doctor
```

`hermes config` names the managed source and pinned keys; `hermes doctor`
reports the resolved managed directory.

### Models: OpenRouter default and DeepSeek fallback

A fresh volume seeds `/opt/data/config.yaml` with:

```yaml
model:
  default: nvidia/nemotron-3.5-super-vl:free
```

OpenRouter's provider and base URL are pinned by managed scope, so the key lives
in the `hermes-env` Secret as `OPENROUTER_API_KEY` and a stale base URL on the
PVC cannot redirect traffic. The default model itself is not managed.

Change the default through the CLI:

```bash
kubectl -n hermes exec deploy/hermes -c hermes -- \
  hermes config set model.default stealth/crouton-alpha
```

The gateway reads the new default for new sessions; restart the deployment if
you want to be certain. `/model openrouter:<model-id>` inside a chat changes
only that session; `/model deepseek:deepseek-flash` switches a session to
DeepSeek. Because `model.provider` and `model.base_url` are pinned, changing
the provider itself still requires a chart change. Do not put API keys in
`values.yaml`.

### Split proxy sidecar

The chart runs a native sidecar (`split-proxy`) using the same Hermes image. It
listens on `127.0.0.1:8118` and speaks HTTP CONNECT:

- hosts in `splitProxy.domains` (default `openrouter.ai`) go through x13's
  SOCKS chain (`splitProxy.socksEndpoint`);
- every other host connects directly.

The main container receives:

```
HTTPS_PROXY=http://127.0.0.1:8118
HTTP_PROXY=http://127.0.0.1:8118
NO_PROXY=127.0.0.1,localhost,::1
```

These explicit `env` values override any proxy keys in the `hermes-env` Secret.
Because the workload speaks HTTP to the sidecar, `requests` and `httpx` need no
PySocks. This is what fixes `Missing dependencies for SOCKS support.` in Exa,
Parallel, and Keenable, while keeping their traffic on the direct route.

```yaml
splitProxy:
  enabled: true
  port: 8118
  socksEndpoint: 192.168.1.100:10808
  domains:
    - openrouter.ai
```

Add `deepseek.com` or search domains if they should be tunneled. Set
`enabled: false` to fall back to the env-based proxy from `OPENROUTER_PROXY`
(global, and `requests`-based tools then need PySocks). Telegram keeps its own
`TELEGRAM_PROXY` and is unaffected.

The sidecar is declared as an `initContainer` with `restartPolicy: Always` and a
startup probe, so the main container does not start before the listener is
ready. It runs as UID 10000 with a read-only root filesystem and dropped
capabilities.

### Environment-based proxy fallback

With `splitProxy.enabled: false`, the workflow's `OPENROUTER_PROXY` is applied
globally as `HTTPS_PROXY`/`HTTP_PROXY`; `NO_PROXY` is the only exclusion
mechanism, and `requests`-based backends need PySocks. `OPENROUTER_PROXY=direct`
disables the proxy entirely. `NO_PROXY` supports exact hosts, domain suffixes,
CIDR ranges, `*.` wildcards, and `host:port` entries.
