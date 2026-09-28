# Agent Installation Guide

This repository is the public binary distribution for Nap. It does not contain
the private source code. Coding agents should install and configure the released
package from Homebrew, APT, or DNF.

## Goal

Help a user install Nap in one of these roles:

- Client-only workstation: install `napctl` and configure a remote node.
- Docker/GPU host: install `napctl`, run `napd`, configure services, and expose
  the Nap gateway.
- Single-machine setup: install the same package and run both `napctl` and
  `napd` locally.

The package name is always `napctl`. It installs both binaries:

- `napctl`: CLI and terminal UI
- `napd`: host agent, control API, gateway, and Docker orchestrator

## Rules For Agents

- Do not clone or request access to the private source repository.
- Do not store bearer tokens, Hugging Face tokens, API keys, or credentials in
  Git.
- Do not run `docker compose down` for normal Nap switching. Nap expects
  existing containers and manages them with `docker start` and `docker stop`.
- Do not enable `napd` on a client-only workstation unless the user explicitly
  wants that machine to manage local Docker workloads.
- Prefer package-manager installs over manual binary downloads.
- Verify every installation with `napctl version`, `napd --version`, `doctor`,
  `host inspect`, and `service ls`.
- If the user wants a client that works without repeated `export` commands, use
  `napctl node token set <node> --stdin` to store the token in that user's local
  Nap config.
- Use HTTPS or a trusted network boundary for remote control in production.

## Information To Collect First

Ask or infer these values before changing configuration:

- Is this machine a client, an agent host, or both?
- Operating system and architecture.
- Docker container names that Nap should manage.
- Internal target URLs for each container, for example
  `http://127.0.0.1:8092`.
- Public paths or hostnames users already call.
- Resource requirements per service: RAM, GPU count, vendor, minimum VRAM.
- Whether a reverse proxy such as Caddy, Nginx, Apache, or Traefik is already in
  front of the services.
- For remote clients: server IP or DNS name and remote-control token env var.

## macOS Client Install

Use this for a Mac that only controls remote hosts:

```sh
brew tap yousef-muc/tap
brew install napctl
napctl version
napd --version
napctl config path
```

Do not start the service on client-only Macs:

```sh
brew services stop napctl || true
```

Configure a remote node:

```sh
export NAP_REMOTE_TOKEN='replace-with-server-token'
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

Optionally store the token directly in the local client config so future
`napctl` runs do not require an `export` first:

```sh
printf '%s' 'replace-with-server-token' | napctl node token set gpu-server --stdin
```

Verify:

```sh
napctl --node gpu-server doctor
napctl --node gpu-server host inspect
napctl --node gpu-server service ls
napctl --node gpu-server
```

## macOS Local Docker Host

Only use this when the Mac itself should manage local Docker containers:

```sh
brew tap yousef-muc/tap
brew install napctl
brew services start napctl
napctl doctor
napctl host inspect
```

## Ubuntu And Debian Agent Host

Install:

```sh
curl -fsSL https://yousef-muc.github.io/napctl/install.sh | sh
napctl version
napd --version
```

Manual APT setup if needed:

```sh
sudo install -d -m 0755 /usr/share/keyrings
curl -fsSL https://yousef-muc.github.io/napctl/apt/gpg/napctl.gpg | sudo tee /usr/share/keyrings/napctl-archive-keyring.gpg >/dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/napctl-archive-keyring.gpg] https://yousef-muc.github.io/napctl/apt stable main" | sudo tee /etc/apt/sources.list.d/napctl.list >/dev/null
sudo apt update
sudo apt install napctl
```

Enable the host agent:

```sh
sudo usermod -aG docker nap
sudo usermod -aG nap "$USER"
sudo systemctl enable --now napd
sudo systemctl restart napd
```

Verify host access:

```sh
id nap
sudo -u nap docker ps
sudo -u nap nvidia-smi || true
sudo napctl --config /etc/nap/nap.yaml config validate
sudo napctl --config /etc/nap/nap.yaml --socket /run/nap/napd.sock doctor
sudo napctl --config /etc/nap/nap.yaml --socket /run/nap/napd.sock host inspect
```

## Fedora, RHEL, CentOS, And Compatible Hosts

```sh
sudo curl -fsSL https://yousef-muc.github.io/napctl/rpm/napctl.repo -o /etc/yum.repos.d/napctl.repo
sudo dnf install napctl
sudo usermod -aG docker nap
sudo systemctl enable --now napd
```

## Prepare Existing Docker Containers

Nap starts and stops existing containers. Create them once with Docker Compose,
then stop them:

```sh
cd /path/to/service-compose
docker compose up -d
docker stop CONTAINER_NAME
docker ps -a --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

Avoid `docker compose down` during normal operation.

## Agent Host Configuration

Edit `/etc/nap/nap.yaml`:

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
  example:
    host: example.nap.local
    container: example
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

Apply:

```sh
sudo napctl --config /etc/nap/nap.yaml config validate
sudo systemctl restart napd
sudo napctl --config /etc/nap/nap.yaml --socket /run/nap/napd.sock service ls
```

## Remote Control On Agent Host

Generate a token on the server:

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

Show the configured token again on an agent host such as `blackwell` or
`vision`:

```sh
sudo grep '^NAP_REMOTE_TOKEN=' /etc/nap/napd.env
sudo sed -n 's/^NAP_REMOTE_TOKEN=//p' /etc/nap/napd.env
```

If no token is stored there, generate a new one, update `/etc/nap/napd.env`, and
restart `napd`.

Restrict port `8788` to trusted clients with firewall rules when possible.

## Client Token Storage

Agents cannot permanently set environment variables in the user's parent shell.
Use one of these supported client-side approaches.

Session-only token:

```sh
export NAP_REMOTE_TOKEN='replace-with-server-token'
napctl --node gpu-server doctor
```

Persistent per-node token in the local Nap config:

```sh
printf '%s' 'replace-with-server-token' | napctl node token set gpu-server --stdin
napctl --node gpu-server doctor
napctl
```

Direct flag form, only when the user accepts shell-history exposure:

```sh
napctl node token set gpu-server --token 'replace-with-server-token'
```

Remove the stored direct token:

```sh
napctl node token unset gpu-server
```

With multiple nodes configured, plain `napctl` opens the multi-node TUI. Switch
nodes with `n` or `Tab`, and `p` for the previous node. Commands act on the
currently selected node.

## Reverse Proxy Through Nap Gateway

Nap gateway routes by `Host` header. If Caddy is in front, route public paths to
Nap gateway and set the internal service host:

```caddyfile
http://SERVER_IP {
  handle_path /app/* {
    reverse_proxy 127.0.0.1:8787 {
      header_up Host example.nap.local
    }
  }
}
```

Validate and reload:

```sh
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

## Operational Verification

Run from a configured client:

```sh
napctl --node gpu-server doctor
napctl --node gpu-server host inspect
napctl --node gpu-server service plan example
napctl --node gpu-server service ls
napctl --node gpu-server
```

Run on the server:

```sh
journalctl -u napd -f
docker ps -a
nvidia-smi
```

## Resource Behavior

Nap protects active HTTP requests. If a service has no active request and no
relevant GPU work, another incoming request may preempt it by stopping its
container. Nap observes CPU, RAM, GPU utilization, VRAM, and GPU processes
attributed to Docker containers.

Linux may keep file cache after a container stops. Treat `Memory Available` as
the scheduling signal, not raw `used` memory.

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
