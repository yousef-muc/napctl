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

Verify from the client:

```sh
napctl --node gpu-server doctor
napctl --node gpu-server host inspect
napctl --node gpu-server service ls
napctl --node gpu-server
```

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
