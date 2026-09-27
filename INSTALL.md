# Installation

This repository contains public release assets for `napctl`.

The source repository is private. Public releases contain prebuilt binaries, Linux packages, checksums, and installation documentation.

## Homebrew

```sh
brew tap yousef-muc/tap
brew install napctl
```

This installs both `napctl` and `napd`.

For client-only use:

```sh
napctl --help
```

For local Docker workloads on macOS:

```sh
brew services start napctl
```

## Debian and Ubuntu

Download the matching `.deb` package from the release page:

```sh
sudo apt install ./napctl_VERSION_linux_amd64.deb
```

On ARM64 Linux:

```sh
sudo apt install ./napctl_VERSION_linux_arm64.deb
```

## RPM-Based Linux

Download the matching `.rpm` package from the release page:

```sh
sudo dnf install ./napctl_VERSION_linux_amd64.rpm
```

On ARM64 Linux:

```sh
sudo dnf install ./napctl_VERSION_linux_arm64.rpm
```

## Agent Hosts

Run `napd` only on machines that should manage Docker workloads.

```sh
sudo usermod -aG docker nap
sudo usermod -aG nap "$USER"
sudo systemctl enable --now napd
```

After changing group membership, start a new login shell.

## Client-Only Hosts

Client-only hosts do not need to start `napd`. They can use `napctl` to connect to remote agents over HTTPS with bearer-token authentication.

## Verify a Release

Download `checksums.txt` and the package or archive you want to verify, then run:

```sh
shasum -a 256 -c checksums.txt
```
