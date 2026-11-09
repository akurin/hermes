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
| `LM_API_KEY` | *Optional.* LM Studio server token. If unset, the workflow stores a placeholder so keyless LM Studio servers still work |

Optional environment variables:

| Variable | Purpose |
| --- | --- |
| `TELEGRAM_ALLOWED_USERS` | Comma-separated numeric Telegram user IDs authorized to use Hermes. Required; not sensitive, so it lives as a variable |
| `TELEGRAM_PROXY` | Override the default `socks5://192.168.1.100:10808` Xray chain endpoint if it changes |
| `OPENROUTER_PROXY` | *Fallback only.* Used when `splitProxy.enabled: false`; the sidecar overrides it. Defaults to the Xray chain endpoint; `direct` disables proxying |
| `NO_PROXY` | *Fallback only.* Exclusions for the env-based proxy; the sidecar bypasses loopback, private, and link-local ranges. Supports domain suffixes, CIDRs, `*.` wildcards, and `host:port` |
| `LMSTUDIO_API_URL` | Set to enable the `lmstudio-lan` provider. The workflow injects it into `managedConfig`; no LAN address is stored in the chart. Unset means no provider |
| `LMSTUDIO_MODEL` | Optional chat model ID for `lmstudio-lan` (e.g. `google/gemma-4-31b-qat`). Injected as `default_model`, guaranteeing a picker entry when live discovery is cold |
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
  terminal:
    backend: local

# Seeded only when /opt/data/config.yaml does not exist.
seedConfig:
  model:
    provider: custom:lmstudio-lan
    default: google/gemma-4-31b-qat
```

Managed keys are leaf-level. Only `terminal.backend` and the LM Studio provider
definition injected by the workflow are managed; `model.provider`,
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

### Models: LM Studio default, OpenRouter available

A fresh volume seeds `/opt/data/config.yaml` with:

```yaml
model:
  provider: custom:lmstudio-lan
  default: google/gemma-4-31b-qat
```

Change the default from chat:

```
/model google/gemma-4-31b-qat --provider lmstudio-lan --global
```

`--global` writes `model.provider`, `model.default`, and the resolved
`model.base_url` to `/opt/data/config.yaml`; `/new` then starts on LM Studio.
Without `--global`, the switch is session-scoped.

Switch back to OpenRouter the same way:

```
/model deepseek/deepseek-v4.1-flash-1017 --provider openrouter --global
```

After a `--global` switch, verify and clear a stale base URL if present:

```bash
kubectl -n hermes exec deploy/hermes -c hermes -- hermes config get model
kubectl -n hermes exec deploy/hermes -c hermes -- hermes config unset model.base_url
```

`OPENROUTER_API_KEY` stays in the `hermes-env` Secret. Do not put API keys in
`values.yaml`.

### Local models via LM Studio

The chart contains no LM Studio address. When the `LMSTUDIO_API_URL` GitHub
environment variable is set, the deploy workflow injects the
`providers.lmstudio-lan` entry with that URL; the token comes from the optional
`LM_API_KEY` secret. Unset the variable and redeploy to remove the provider.

All private and link-local networks (`10.0.0.0/8`, `172.16.0.0/12`,
`192.168.0.0/16`, `169.254.0.0/16`) bypass the sidecar, so LAN requests connect
directly.

`seedConfig` makes LM Studio the initial default. Switch per session with:

```
/model --provider lmstudio-lan <model-id>
```

or persist it from chat:

```
/model <model-id> --provider lmstudio-lan --global
```

Change the URL by updating the `LMSTUDIO_API_URL` variable and redeploying — no
PVC edit needed. `LMSTUDIO_MODEL` adds a declared model so the picker has an
entry even when the live catalog probe is cold.

Requirements:

- LM Studio Developer tab → **Serve on Network** (not localhost-only).
- The Hermes pod must stay pinned to x13; the cloud worker has no route to the
  home LAN.
- Load a model before switching. Tool calling needs LM Studio 0.3.6+ and a
  model with native tool support; a ~64k context length is recommended.

Verify from the pod:

```bash
curl --noproxy '*' -sS http://192.168.1.101:1234/v1/models
```

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
