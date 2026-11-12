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
| `STRATA_API_KEY` | API key for the Strata endpoint on port 47832 |
| `UNSLOTH_API_KEY` | API key for the Unsloth Desktop endpoint on port 8888 |

Environment variables (set the required entries before deploying):

| Variable | Purpose |
| --- | --- |
| `TELEGRAM_ALLOWED_USERS` | **Required.** Comma-separated numeric Telegram user IDs authorized to use Hermes |
| `STRATA_API_URL` | **Required.** Set to `http://192.168.1.101:47832/v1` |
| `UNSLOTH_API_URL` | **Required.** Set to `http://192.168.1.101:8888/v1` |
| `TELEGRAM_PROXY` | Override the default `socks5://192.168.1.100:10808` Xray chain endpoint if it changes |
| `OPENROUTER_PROXY` | *Fallback only.* Used when `splitProxy.enabled: false`; the sidecar overrides it. Defaults to the Xray chain endpoint; `direct` disables proxying |
| `NO_PROXY` | *Fallback only.* Exclusions for the env-based proxy; the sidecar bypasses loopback, private, and link-local ranges. Supports domain suffixes, CIDRs, `*.` wildcards, and `host:port` |
| `HERMES_IMAGE_TAG` | Official image tag; defaults to `v0.23.0` |

The workflow recreates the `hermes-env` Kubernetes Secret before Helm installs
the chart: it deletes the old Secret and creates it from the workflow inputs, so
keys removed from GitHub do not linger. No secret values are placed in chart
values. A push to
`master` runs chart checks and deploys after they pass. `workflow_dispatch`
runs the same checks before deploying. Secrets are scoped to this repository's
production environment; Penny's GitHub environment does not share them.
Telegram and cloud LLM traffic use x13's Xray `10808` chain endpoint because
their direct routes are blocked at home. The Strata and Unsloth LAN endpoints
bypass the split proxy through the private-network `NO_PROXY` ranges.

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
  terminal:
    backend: local

# Seeded only when /opt/data/config.yaml does not exist.
seedConfig:
  model:
    provider: openrouter
    default: deepseek/deepseek-v4.1-flash-1017
    base_url: https://openrouter.ai/api/v1
```

Managed keys are leaf-level. `terminal.backend` and the Strata and Unsloth
provider definitions injected by the workflow are managed; `model.provider`,
`model.base_url`, and `model.default` are user-owned. That is what lets
`/model ... --global` change the default from chat. Managed values still win
over the PVC config and the environment. API keys stay in the `hermes-env`
Secret rather than the managed `.env` layer.

Verify the policy from inside the pod:

```bash
kubectl -n hermes exec deploy/hermes -c hermes -- hermes config
kubectl -n hermes exec deploy/hermes -c hermes -- hermes doctor
```

`hermes config` names the managed source and pinned keys; `hermes doctor`
reports the resolved managed directory.

### Models: OpenRouter DeepSeek default, LAN providers available

A fresh volume seeds `/opt/data/config.yaml` with:

```yaml
model:
  provider: openrouter
  default: deepseek/deepseek-v4.1-flash-1017
  base_url: https://openrouter.ai/api/v1
```

Switch to a LAN model from chat, using a model ID served by that endpoint:

```
/model custom:strata:<model-id>
/model custom:unsloth:<model-id>
```

Add `--global` to either command to save it as the default in
`/opt/data/config.yaml`; `/new` then starts on that provider. Without
`--global`, the switch is session-scoped. An existing PVC keeps its manually
configured default when the LAN providers are added; only the managed provider
definitions take effect on deployment.

Switch back to the OpenRouter default the same way:

```
/model deepseek/deepseek-v4.1-flash-1017 --provider openrouter --global
```

If a switch routes to an unexpected endpoint, inspect the saved model config
and clear a stale `model.base_url` if present:

```bash
kubectl -n hermes exec deploy/hermes -c hermes -- hermes config get model
kubectl -n hermes exec deploy/hermes -c hermes -- hermes config unset model.base_url
```

All provider API keys stay in the `hermes-env` Secret. Do not put API keys in
`values.yaml`.

### Local models via Strata and Unsloth Desktop

The deploy workflow injects `providers.strata` and `providers.unsloth` into
`managedConfig`, using the URLs above. Their keys come from the required
`STRATA_API_KEY` and `UNSLOTH_API_KEY` GitHub production secrets. Update either
URL variable and redeploy to change its endpoint; no PVC edit is needed. Hermes
discovers available models from each authenticated `/v1/models` endpoint, so
there is no model ID to maintain in the workflow. If an endpoint does not
provide that listing, declare its models in the provider config instead.

The workflow resets Helm values on upgrade, so a previously injected
`providers.lmstudio-lan` entry is removed from managed config. If you added an
LM Studio provider directly to `/opt/data/config.yaml`, remove that manual
entry separately. If your saved default still selects LM Studio, switch it to
Strata, Unsloth, or OpenRouter with `/model ... --global`.

All private and link-local networks (`10.0.0.0/8`, `172.16.0.0/12`,
`192.168.0.0/16`, `169.254.0.0/16`) bypass the sidecar, so LAN requests connect
directly.

`seedConfig` still defaults to OpenRouter DeepSeek on a fresh PVC. Use the
`/model custom:<provider>:<model-id>` commands above to select a LAN provider.

Requirements:

- Both servers must listen on the LAN rather than localhost only.
- The Hermes pod must stay pinned to x13; the cloud worker has no route to the
  home LAN.
- Load a model before switching. For tool use, choose a model and server that
  support OpenAI-compatible tool calls; a ~64k context length is recommended.

Verify from the pod:

Run `hermes doctor` in the pod, then query each server's `/v1/models` with its
respective bearer token. Do not print the API keys into the pod logs.

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
  noProxy:
    - 127.0.0.1
    - localhost
    - "::1"
    - 10.0.0.0/8
    - 172.16.0.0/12
    - 192.168.0.0/16
    - 169.254.0.0/16
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
