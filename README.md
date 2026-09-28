# napctl

`napctl` is the public binary distribution for Nap, a local and edge compute
orchestrator for Docker workloads.

Nap starts existing Docker containers on demand, waits for health checks,
proxies HTTP requests, observes host resources, and stops idle services after a
configured timeout.

## Included Binaries

The package installs both binaries:

- `napctl`: CLI and terminal UI client
- `napd`: host agent, control API, gateway, and Docker orchestrator

Install the same package everywhere. Run `napd` only on machines that should
manage Docker workloads. Client-only machines can use `napctl` without starting
the daemon.

## Coding Agent Installation

If you want ChatGPT Codex, OpenClaude, OpenCode, Pi, or another coding agent to
install and configure Nap for you, point it to [AGENTS.md](AGENTS.md). That file
contains the package-manager install commands, role-specific setup rules,
remote-node configuration, reverse-proxy guidance, and verification checklist an
agent should follow.

## macOS Install

```sh
brew tap yousef-muc/tap
brew install napctl
```

Verify:

```sh
napctl version
napd --version
napctl config path
```

Client-only Macs do not need to start the service. If this Mac should manage
local Docker workloads:

```sh
brew services start napctl
```

Update:

```sh
brew update
brew upgrade napctl
```

## Ubuntu And Debian Install

One-line install:

```sh
curl -fsSL https://yousef-muc.github.io/napctl/install.sh | sh
```

Manual APT repository setup:

```sh
sudo install -d -m 0755 /usr/share/keyrings
curl -fsSL https://yousef-muc.github.io/napctl/apt/gpg/napctl.gpg | sudo tee /usr/share/keyrings/napctl-archive-keyring.gpg >/dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/napctl-archive-keyring.gpg] https://yousef-muc.github.io/napctl/apt stable main" | sudo tee /etc/apt/sources.list.d/napctl.list >/dev/null
sudo apt update
sudo apt install napctl
```

Verify:

```sh
napctl version
napd --version
sudo systemctl status napd --no-pager
sudo napctl --config /etc/nap/nap.yaml config validate
sudo napctl --config /etc/nap/nap.yaml --socket /run/nap/napd.sock doctor
sudo napctl --config /etc/nap/nap.yaml --socket /run/nap/napd.sock host inspect
sudo napctl --config /etc/nap/nap.yaml --socket /run/nap/napd.sock service ls
```

Update:

```sh
sudo apt update
sudo apt upgrade napctl
```

## Fedora, RHEL, CentOS, And Compatible Install

```sh
sudo curl -fsSL https://yousef-muc.github.io/napctl/rpm/napctl.repo -o /etc/yum.repos.d/napctl.repo
sudo dnf install napctl
```

Update:

```sh
sudo dnf upgrade napctl
```

The APT repository metadata is signed. The RPM repository metadata and RPM
packages are signed with the published Nap Linux repository key:

```text
https://yousef-muc.github.io/napctl/rpm/RPM-GPG-KEY-napctl
```

## Enable A Linux Agent Host

Run these only on machines that should manage Docker workloads:

```sh
sudo usermod -aG docker nap
sudo usermod -aG nap "$USER"
sudo systemctl enable --now napd
```

After changing group membership, restart `napd` and start a new login shell:

```sh
sudo systemctl restart napd
```

Check Docker and GPU visibility:

```sh
id nap
sudo -u nap docker ps
sudo -u nap nvidia-smi
```

## Configure Services

Nap currently manages existing Docker containers with:

```text
docker start
docker stop
docker inspect
docker logs
```

Create containers once with Docker or Docker Compose, then leave them stopped for
Nap to wake on demand. Avoid `docker compose down` for normal switching, because
it can remove the container Nap needs to start.

Example `/etc/nap/nap.yaml`:

```yaml
agent:
  socket: /run/nap/napd.sock

nodes:
  local:
    type: local
    default: true
    socket: /run/nap/napd.sock

gateway:
  listen: 0.0.0.0:8787

services:
  app:
    host: app.nap.local
    container: app
    target: http://127.0.0.1:8080
    healthcheck:
      url: http://127.0.0.1:8080/
      timeout: 120s
      interval: 2s
    resources:
      memory: 8GiB
      gpus:
        count: 1
        vendor: nvidia
        min_vram: 8GiB
    idle: 5m
    stop_timeout: 30s
```

Apply config changes:

```sh
sudo napctl --config /etc/nap/nap.yaml config validate
sudo systemctl restart napd
sudo napctl --config /etc/nap/nap.yaml --socket /run/nap/napd.sock service ls
```

If `service ls` is empty, `napd` is running but no services are configured yet.

## Configuration And Environment Files

`napctl` is the client. It reads a `nap.yaml` and optional shell environment
variables. It does not have a separate global `.env` file.

Common client paths:

- macOS user config: `$HOME/Library/Application Support/nap/nap.yaml`
- Linux user config: `$HOME/.config/nap/nap.yaml`
- Resolved path: `napctl config path`
- Override path: `NAP_CONFIG=/path/to/nap.yaml napctl ...`

Client-only `nap.yaml` example:

```yaml
nodes:
  gpu-server:
    type: remote
    default: true
    url: http://SERVER_IP:8788
    token_env: NAP_REMOTE_TOKEN

  gpu-server-2:
    type: remote
    url: http://SECOND_SERVER_IP:8788
    token_env: NAP_REMOTE_TOKEN_2
```

Instead of using `token_env`, the token can be stored directly per node with
`napctl node token set`. Do this only in a local private client config.

`napd` is the host daemon. Linux packages use:

- `/etc/nap/nap.yaml`: main daemon config
- `/etc/nap/napd.env`: optional systemd environment file for secrets/overrides
- `/run/nap/napd.sock`: local Unix control socket
- `/usr/lib/systemd/system/napd.service`: packaged systemd unit

Typical `/etc/nap/napd.env`:

```sh
NAP_LOG_LEVEL=info
NAP_REMOTE_TOKEN=replace-with-generated-token
NAP_REMOTE_LISTEN=0.0.0.0:8788
NAP_REMOTE_TOKEN_ENV=NAP_REMOTE_TOKEN
```

Supported `napd` environment variables:

| Variable | Flag | Purpose |
| --- | --- | --- |
| `NAP_CONFIG` | `--config` | Path to `nap.yaml`. |
| `NAP_SOCKET` | `--socket` | Local control socket. |
| `NAP_GATEWAY_LISTEN` | `--gateway-listen` | HTTP gateway listen address. |
| `NAP_REMOTE_LISTEN` | `--remote-listen` | Remote-control API listen address. |
| `NAP_REMOTE_TOKEN` | `--remote-token` | Remote-control bearer token. |
| `NAP_REMOTE_TOKEN_ENV` | `--remote-token-env` | Env var name containing the remote token. |
| `NAP_REMOTE_TLS_CERT` | `--remote-tls-cert` | TLS certificate for remote control. |
| `NAP_REMOTE_TLS_KEY` | `--remote-tls-key` | TLS private key for remote control. |
| `NAP_LOG_LEVEL` | `--log-level` | `debug`, `info`, `warn`, or `error`. |

## Split Client And Agent Setup

Typical setup:

```text
MacBook or admin workstation
  napctl
    |
    | HTTP(S) + bearer token
    v
Docker/GPU server
  napd
    ├── remote control API
    ├── HTTP gateway
    └── Docker workloads
```

On the server, generate a token:

```sh
napctl token --env NAP_REMOTE_TOKEN
```

Create `/etc/nap/napd.env`:

```sh
sudo tee /etc/nap/napd.env >/dev/null <<'EOF'
NAP_LOG_LEVEL=info
NAP_REMOTE_TOKEN=replace-with-generated-token
NAP_REMOTE_LISTEN=0.0.0.0:8788
NAP_REMOTE_TOKEN_ENV=NAP_REMOTE_TOKEN
EOF
sudo chmod 0600 /etc/nap/napd.env
sudo systemctl restart napd
```

Show the configured token again on an agent host:

```sh
sudo grep '^NAP_REMOTE_TOKEN=' /etc/nap/napd.env
sudo sed -n 's/^NAP_REMOTE_TOKEN=//p' /etc/nap/napd.env
```

On the client machine:

```sh
export NAP_REMOTE_TOKEN='replace-with-generated-token'
mkdir -p "$HOME/Library/Application Support/nap"
cat > "$HOME/Library/Application Support/nap/nap.yaml" <<'EOF'
nodes:
  gpu-server:
    type: remote
    default: true
    url: http://SERVER_IP:8788
    token_env: NAP_REMOTE_TOKEN
EOF
```

Instead of exporting tokens before every shell session, store the token directly
for a configured node in the local client config:

```sh
printf '%s' 'replace-with-generated-token' | napctl node token set gpu-server --stdin
napctl
```

The direct form also exists, but may be captured in shell history:

```sh
napctl node token set gpu-server --token 'replace-with-generated-token'
```

Remove a direct token from the local config:

```sh
napctl node token unset gpu-server
```

Verify from the client:

```sh
napctl --node gpu-server doctor
napctl --node gpu-server host inspect
napctl --node gpu-server service ls
napctl --node gpu-server
```

With multiple configured nodes, plain `napctl` opens the multi-node TUI. Switch
nodes with `n` or `Tab`, and `p` for the previous node. Commands act on the
currently selected node.

For production, put HTTPS in front of the remote control API or use native TLS.

## Gateway Requests

Nap routes gateway traffic by `Host` header.

```sh
curl -v -H 'Host: app.nap.local' http://SERVER_IP:8787/
```

When the request arrives, `napd` checks resources, starts the configured
container if needed, waits for the healthcheck, then proxies the request to the
configured target.

## Verify Release Downloads

Every release includes `checksums.txt`.

```sh
shasum -a 256 -c checksums.txt
```

## License

Released under the MIT License. See [LICENSE](LICENSE).
