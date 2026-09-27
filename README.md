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

Install from the Linux package repository:

```sh
curl -fsSL https://yousef-muc.github.io/napctl/install.sh | sh
```

Debian and Ubuntu:

```sh
sudo install -d -m 0755 /usr/share/keyrings
curl -fsSL https://yousef-muc.github.io/napctl/apt/gpg/napctl.gpg | sudo tee /usr/share/keyrings/napctl-archive-keyring.gpg >/dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/napctl-archive-keyring.gpg] https://yousef-muc.github.io/napctl/apt stable main" | sudo tee /etc/apt/sources.list.d/napctl.list >/dev/null
sudo apt update
sudo apt install napctl
```

Fedora, RHEL, CentOS, and compatible distributions:

```sh
sudo curl -fsSL https://yousef-muc.github.io/napctl/rpm/napctl.repo -o /etc/yum.repos.d/napctl.repo
sudo dnf install napctl
```

The APT repository metadata is signed. The RPM repository metadata and RPM
packages are signed with the published Nap Linux repository key.

## Updates

After installing from Homebrew:

```sh
brew update
brew upgrade napctl
```

After installing from the APT repository:

```sh
sudo apt update
sudo apt upgrade napctl
```

After installing from the DNF/YUM repository:

```sh
sudo dnf upgrade napctl
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
