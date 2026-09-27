# napctl

`napctl` is the command-line and terminal UI distribution for Nap, a local and edge compute orchestrator for Docker workloads.

Nap starts configured containers on demand, waits for health checks, proxies HTTP requests, and stops idle services after a configurable timeout. It is designed for workstations, LAN GPU hosts, and remote Docker/GPU servers.

## What Is Included

The `napctl` package installs two binaries:

- `napctl`: CLI and terminal UI client
- `napd`: host agent, control API, gateway, and Docker orchestrator

Install the same package everywhere. Run `napd` only on machines that should manage Docker workloads. Client-only machines can use `napctl` without starting the daemon.

## macOS

Install with Homebrew:

```sh
brew tap yousef-muc/tap
brew install napctl
```

Check the installed binaries:

```sh
napctl version
napd --version
```

Start the local agent only if this Mac should manage local Docker workloads:

```sh
brew services start napctl
```

Default Homebrew configuration:

```text
$(brew --prefix)/etc/nap/nap.yaml
```

## Linux

Download the `.deb` or `.rpm` package from the latest GitHub release:

```sh
sudo apt install ./napctl_VERSION_linux_amd64.deb
```

or:

```sh
sudo dnf install ./napctl_VERSION_linux_amd64.rpm
```

The Linux package installs:

```text
/usr/bin/napctl
/usr/bin/napd
/etc/nap/nap.yaml
```

Enable the agent only on hosts that should manage Docker workloads:

```sh
sudo usermod -aG docker nap
sudo usermod -aG nap "$USER"
sudo systemctl enable --now napd
```

## Typical Split Setup

```text
MacBook or admin workstation
  napctl
    |
    | HTTPS + bearer token
    v
Docker/GPU server
  napd
    ├── control API
    ├── HTTP gateway
    └── Docker workloads
```

The gateway, scheduling, Docker orchestration, and GPU checks run on the host where `napd` is installed.

## Verify Downloads

Every release includes `checksums.txt`.

```sh
shasum -a 256 -c checksums.txt
```

## License

Released under the MIT License. See [LICENSE](LICENSE).
