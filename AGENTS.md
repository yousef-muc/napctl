# Coding Agent Guide

This file is for ChatGPT Codex, OpenClaude, OpenCode, Pi, and other coding or
operations agents that install and configure Nap from the public `napctl`
distribution repository.

## Primary Reference

Before changing anything, read [README.md](README.md). The README is the
canonical product and installation reference for this repository. Use this file
as an agent-focused operating checklist, not as a replacement for the README.

If this guide and the README ever disagree, prefer the README and then make a
minimal documentation update so both files match.

## Repository Scope

This repository is the public binary distribution for Nap. It intentionally does
not contain the private source code.

The package name is always `napctl`. It installs both binaries:

- `napctl`: CLI and terminal UI client.
- `napd`: host agent, control API, gateway, and Docker workload orchestrator.

Install the released package from Homebrew, APT, or DNF. Do not clone or request
access to the private source repository.

## Agent Safety Rules

- Do not store bearer tokens, Hugging Face tokens, API keys, or credentials in
  Git, shell history, screenshots, logs, or public documentation.
- Do not run `docker compose down` for normal Nap switching. Nap expects
  existing containers and manages them with `docker start` and `docker stop`.
- Do not enable `napd` on a client-only workstation unless the user explicitly
  wants that machine to manage local Docker workloads.
- Do not expose the remote control API on an untrusted network without a bearer
  token and a firewall, VPN, private network, or TLS/reverse-proxy boundary.
- Prefer package-manager installs over manual binary downloads.
- After every install or config change, verify with the commands in the
  verification checklist below.
- For multi-node client configs, never set `default: true` on more than one
  node.

## Decide The Machine Role

Ask or infer the role before making changes:

- Client-only workstation: install `napctl`, configure remote nodes, do not run
  local `napd`.
- Agent host: install `napctl`, run `napd`, configure Docker services, expose
  the Nap gateway, and optionally expose remote control.
- Single-machine setup: install one package and run both `napctl` and `napd` on
  the same host.

Also collect:

- Operating system and architecture.
- Docker container names Nap should manage.
- Internal target URL for each container, for example `http://127.0.0.1:8092`.
- Public hostnames or paths users already call.
- RAM, GPU count, GPU vendor, and minimum VRAM requirements per service.
- Whether Caddy, nginx, Apache, Traefik, or another reverse proxy is already in
  front of the services.
- Remote server IP or DNS name and desired token environment variable names.
- Which single node should be the default in a multi-node client config.

## Install By Platform

Follow the README for full details. Use this section as the shortest safe
execution checklist.

### macOS Client

Use this when the Mac controls remote Linux hosts:

```sh
brew tap yousef-muc/tap
brew install napctl
napctl version
napd --version
napctl config path
brew services stop napctl || true
```

Configure remote nodes in:

```text
$HOME/Library/Application Support/nap/nap.yaml
```

### macOS Local Docker Host

Use this only when the Mac itself should manage local Docker containers:

```sh
brew tap yousef-muc/tap
brew install napctl
brew services start napctl
napctl doctor
napctl host inspect
```

### Ubuntu And Debian Agent Host

Use the one-line installer when possible:

```sh
curl -fsSL https://yousef-muc.github.io/napctl/install.sh | sh
napctl version
napd --version
```

Manual APT setup:

```sh
sudo install -d -m 0755 /usr/share/keyrings
curl -fsSL https://yousef-muc.github.io/napctl/apt/gpg/napctl.gpg | sudo tee /usr/share/keyrings/napctl-archive-keyring.gpg >/dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/napctl-archive-keyring.gpg] https://yousef-muc.github.io/napctl/apt stable main" | sudo tee /etc/apt/sources.list.d/napctl.list >/dev/null
sudo apt update
sudo apt install napctl
```

Enable the agent only on machines that should manage Docker containers:

```sh
sudo usermod -aG docker nap
sudo usermod -aG nap "$USER"
sudo systemctl enable --now napd
sudo systemctl restart napd
```

Verify boot state:

```sh
systemctl is-active napd
systemctl is-enabled napd
```

Expected:

```text
active
enabled
```

If the service is disabled, run:

```sh
sudo systemctl enable --now napd
```

### Fedora, RHEL, CentOS, And Compatible Hosts

```sh
sudo curl -fsSL https://yousef-muc.github.io/napctl/rpm/napctl.repo -o /etc/yum.repos.d/napctl.repo
sudo dnf install napctl
sudo usermod -aG docker nap
sudo usermod -aG nap "$USER"
sudo systemctl enable --now napd
```

Verify:

```sh
napctl version
napd --version
systemctl is-active napd
systemctl is-enabled napd
```

## Configure Docker Containers For Nap

Nap manages existing Docker containers. Create them once with Docker Compose or
Docker, then stop them:

```sh
cd /path/to/service-compose
docker compose up -d
docker stop CONTAINER_NAME
docker ps -a --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

Avoid `docker compose down` during normal Nap operation. `down` can remove the
container that Nap needs to start later.

## Configure napd On Agent Hosts

Edit:

```text
/etc/nap/nap.yaml
```

Minimal example:

```yaml
agent:
  socket: /run/nap/napd.sock
  remote:
    listen: ""
    token_env: NAP_REMOTE_TOKEN

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
    # Optional Host header presented to the upstream application.
    # upstream_host: app.example.com
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
    idle_shutdown: true
    stop_timeout: 30s
```

Keep `host` as the gateway routing identity. For browser applications that
validate Host against Origin, set optional `upstream_host` to the exact public
host opened by the browser, without a URL scheme. Omitted `upstream_host`
preserves the existing Host behavior.

`idle_shutdown` defaults to `true`. Set it to `false` only when the user wants
to disable time-based idle stopping. It does not make the service
non-preemptible when an inactive container holds resources required by another
workload.

Apply:

```sh
sudo napctl --config /etc/nap/nap.yaml config validate
sudo systemctl restart napd
sudo napctl --config /etc/nap/nap.yaml --socket /run/nap/napd.sock service ls
```

If `service ls` is empty, `napd` is running but no services are configured yet.

## Configure Remote Control

Remote control lets a workstation run `napctl --node gpu-server ...` against an
agent host. Store the server-side token in:

```text
/etc/nap/napd.env
```

Generate a token on the server:

```sh
napctl token --env NAP_REMOTE_TOKEN
```

Create or update `/etc/nap/napd.env`:

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

Show the token again only on the private agent host:

```sh
sudo sed -n 's/^NAP_REMOTE_TOKEN=//p' /etc/nap/napd.env
```

Restrict port `8788` to trusted clients with firewall rules, VPN, private
networking, or TLS where possible.

## Configure napctl Clients

`napctl` reads a local `nap.yaml` and optional environment variables. It does
not have a separate global `.env` file.

Common config paths:

- macOS: `$HOME/Library/Application Support/nap/nap.yaml`
- Linux: `$HOME/.config/nap/nap.yaml`
- Resolved path: `napctl config path`
- Override: `NAP_CONFIG=/path/to/nap.yaml napctl ...`

Single remote node:

```yaml
nodes:
  gpu-server:
    type: remote
    default: true
    url: http://SERVER_IP:8788
    token_env: NAP_REMOTE_TOKEN
```

Multiple remote nodes:

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

Validate before launching the TUI:

```sh
napctl config validate
napctl node ls
```

If the user wants a setup that works without repeated `export` commands, store
the token directly in the local private client config:

```sh
printf '%s' 'replace-with-server-token' | napctl node token set gpu-server --stdin
napctl --node gpu-server doctor
napctl
```

Use the direct flag only when the user accepts shell-history exposure:

```sh
napctl node token set gpu-server --token 'replace-with-server-token'
```

Remove a stored token:

```sh
napctl node token unset gpu-server
```

## Reverse Proxy Through The Nap Gateway

Public traffic should go to the Nap gateway, usually `127.0.0.1:8787`, not
directly to Docker container ports. The reverse proxy must set a `Host` header
that matches the service `host` in `/etc/nap/nap.yaml`.

Example Caddy host-based route:

```caddyfile
app.example.com {
  reverse_proxy 127.0.0.1:8787 {
    header_up Host app.nap.local
  }
}
```

Example Caddy path-based route:

```caddyfile
example.com {
  handle_path /app/* {
    reverse_proxy 127.0.0.1:8787 {
      header_up Host app.nap.local
    }
  }
}
```

For nginx and Apache examples, use the README as the source of truth. Validate
and reload the reverse proxy after editing its config.

## Operational Verification Checklist

Run on the agent host:

```sh
napctl version
napd --version
systemctl is-active napd
systemctl is-enabled napd
id nap
sudo -u nap docker ps
sudo -u nap nvidia-smi || true
sudo napctl --config /etc/nap/nap.yaml config validate
sudo napctl --config /etc/nap/nap.yaml --socket /run/nap/napd.sock doctor
sudo napctl --config /etc/nap/nap.yaml --socket /run/nap/napd.sock host inspect
sudo napctl --config /etc/nap/nap.yaml --socket /run/nap/napd.sock service ls
```

Run from a configured client:

```sh
napctl config validate
napctl node ls
napctl --node gpu-server doctor
napctl --node gpu-server host inspect
napctl --node gpu-server service plan app
napctl --node gpu-server service ls
napctl --node gpu-server
```

Watch server behavior:

```sh
journalctl -u napd -f
docker ps -a
nvidia-smi
```

## Resource Behavior

Nap protects active HTTP requests. If a service has no active request and no
relevant GPU work, another incoming request may preempt it by stopping its
container. Nap observes CPU, available RAM, GPU utilization, VRAM, and GPU
processes attributed to Docker containers.

Linux may keep file cache after a container stops. Treat `Memory Available` as
the scheduling signal, not raw `used` memory.

Multi-GPU hosts are one node with multiple detected GPUs. Services can request
GPU count, vendor, minimum VRAM, and explicit GPU indices when needed.

## Updates

macOS:

```sh
brew update
brew upgrade napctl
```

Ubuntu/Debian:

```sh
sudo apt update
sudo apt install --only-upgrade napctl
sudo systemctl restart napd
```

Fedora/RHEL-compatible:

```sh
sudo dnf upgrade napctl
sudo systemctl restart napd
```

After package updates, confirm agent hosts are still enabled:

```sh
systemctl is-active napd
systemctl is-enabled napd
```
