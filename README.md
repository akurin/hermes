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
| `OPENROUTER_PROXY` | Override the proxy used for all LLM API calls (OpenRouter and DeepSeek); defaults to the same Xray chain endpoint. Set `direct` to disable proxying |
| `NO_PROXY` | Comma-separated hosts that bypass the proxy; defaults to `127.0.0.1,localhost,::1`. Add `exa.ai,parallel.ai,keenable.ai` to send keyless web search direct |
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
    default: nvidia/nemotron-3.5-super-vl:free
    base_url: https://openrouter.ai/api/v1
  terminal:
    backend: local
```

Managed keys are leaf-level: pinning `model.*` does not freeze the rest of the
config. Hermes refuses `hermes config set` on a pinned key, and managed values
win over the PVC config and the environment. Everything else remains editable
on the volume and survives redeploys. Because the managed file is world-readable
by design, API keys stay in the `hermes-env` Secret rather than the managed
`.env` layer.

Verify the policy from inside the pod:

```bash
kubectl -n hermes exec deploy/hermes -c hermes -- hermes config
kubectl -n hermes exec deploy/hermes -c hermes -- hermes doctor
```

`hermes config` names the managed source and pinned keys; `hermes doctor`
reports the resolved managed directory.

### Models: OpenRouter default and DeepSeek fallback

```yaml
model:
  provider: openrouter
  default: nvidia/nemotron-3.5-super-vl:free
```

OpenRouter uses its built-in provider, so the default
`https://openrouter.ai/api/v1` base URL needs no configuration. The key lives
in the `hermes-env` Secret as `OPENROUTER_API_KEY`.

New sessions and gateway conversations start on the OpenRouter model. Switch to
DeepSeek for a session with:

```
/model deepseek:deepseek-flash
```

Or switch to any other OpenRouter model with
`/model openrouter:<model-id>`. To change the default, edit `managedConfig` in
`values.yaml` and redeploy. `model.provider`, `model.default`, and
`model.base_url` are pinned, so a stale base URL on the PVC cannot redirect
OpenRouter traffic. Do not put API keys in `values.yaml`.

### LLM traffic through the proxy

OpenRouter and DeepSeek are blocked from Russia, so the deployment proxies all
LLM API calls. Hermes reads `HTTPS_PROXY`/`HTTP_PROXY` (and `ALL_PROXY`) for its
OpenAI-compatible clients, including streams, auxiliary calls, and fallbacks.
The `10808` Xray chain endpoint is reused by default.

```bash
# In the hermes-env Secret
HTTPS_PROXY=socks5://192.168.1.100:10808
HTTP_PROXY=socks5://192.168.1.100:10808
NO_PROXY=127.0.0.1,localhost,::1
```

Telegram uses its own `TELEGRAM_PROXY` (which overrides `HTTPS_PROXY`).
`NO_PROXY` supports exact hosts, domain suffixes, CIDR ranges, `*.` wildcards,
and `host:port` entries. Hermes has no per-provider proxy: the env proxy applies
to every LLM client and to `requests`-based tools. To keep web search direct,
add its domains to `NO_PROXY`:

```bash
gh variable set NO_PROXY --env production --repo akurin/hermes \
  --body '127.0.0.1,localhost,::1,exa.ai,parallel.ai,keenable.ai,firecrawl.dev'
```

This also avoids the `Missing dependencies for SOCKS support.` error in
`requests`-based search backends, provided those hosts are reachable directly
from x13. If they are blocked, use an HTTP CONNECT proxy instead of SOCKS, or
build an image with PySocks. Set `OPENROUTER_PROXY=direct` in the GitHub
environment variables to disable the LLM proxy entirely.

A `socks5://` proxy requires the image's `httpx[socks]` support; if the pod
reports a SOCKS transport error, point the proxy at an HTTP CONNECT port on
Xray instead.
