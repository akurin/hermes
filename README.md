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
- The chart copies `initialConfig` to `/opt/data/config.yaml` **only when that
  file does not yet exist**. Subsequent Hermes config changes persist there.
  Changing `initialConfig` in Helm values does not replace a live config file.
- Terminal and code tools run inside the Hermes pod using the `local` backend.
  The pod has no host mounts, Docker socket, host networking, or Kubernetes
  API token. Tools can still access files and environment variables inside
  their own pod, including the gateway credentials.
- The official image starts as root for s6 initialization, then runs the
  supervised Hermes service as UID 10000. Its startup hook fixes ownership of
  the data directory and seeded config file for that UID.

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
| `OPENROUTER_PROXY` | Override the proxy used for OpenRouter/LLM API calls; defaults to the same Xray chain endpoint. Set `direct` to disable proxying |
| `HERMES_IMAGE_TAG` | Official image tag; defaults to `v0.23.0` |

The workflow recreates the `hermes-env` Kubernetes Secret before Helm installs
the chart: it deletes the old Secret and creates it from the workflow inputs, so
keys removed from GitHub do not linger. No secret values are placed in chart
values. A push to
`master` runs chart checks and deploys after they pass. `workflow_dispatch`
runs the same checks before deploying. Secrets are scoped to this repository's
production environment; Penny's GitHub environment does not share them.
Telegram and OpenRouter traffic use x13's Xray `10808` chain endpoint because
their direct routes are blocked at home. Hermes applies `OPENROUTER_PROXY` as
`HTTPS_PROXY`/`HTTP_PROXY` for LLM calls and sets
`NO_PROXY=127.0.0.1,localhost,::1,api.deepseek.com`, so DeepSeek keeps the
direct route.

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

To change the model later, edit `/opt/data/config.yaml` through Hermes' config
commands in the pod, or reinitialize a new volume with changed `initialConfig`.
The initial config sets OpenRouter's free
`nvidia/nemotron-3.5-super-vl:free` model as the default and keeps the terminal
backend local to the pod.

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
`/model openrouter:<model-id>`. To change the default, edit
`model.default` in a new volume's `initialConfig` or in the live
`/opt/data/config.yaml`, then restart the deployment. Do not put API keys in
`values.yaml`.

### OpenRouter through the proxy

OpenRouter is blocked from Russia, so the deployment proxies LLM API calls.
Hermes reads `HTTPS_PROXY`/`HTTP_PROXY` (and `ALL_PROXY`) for its OpenAI-compatible
clients, including streams, auxiliary calls, and fallbacks. The `10808` Xray
chain endpoint is reused by default.

```bash
# In the hermes-env Secret
HTTPS_PROXY=socks5://192.168.1.100:10808
HTTP_PROXY=socks5://192.168.1.100:10808
NO_PROXY=127.0.0.1,localhost,::1,api.deepseek.com
```

Telegram uses its own `TELEGRAM_PROXY` (which overrides `HTTPS_PROXY`), and
`NO_PROXY` keeps DeepSeek on the direct route. `NO_PROXY` supports exact hosts,
domain suffixes, CIDR ranges, `*.` wildcards, and `host:port` entries. Set
`OPENROUTER_PROXY=direct` in the GitHub environment variables to disable the
LLM proxy.

A `socks5://` proxy requires the image's `httpx[socks]` support; if the pod
reports a SOCKS transport error, point the proxy at an HTTP CONNECT port on
Xray instead.
