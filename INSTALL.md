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

Configure the signed APT repository:

```sh
sudo install -d -m 0755 /usr/share/keyrings
curl -fsSL https://yousef-muc.github.io/napctl/apt/gpg/napctl.gpg | sudo tee /usr/share/keyrings/napctl-archive-keyring.gpg >/dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/napctl-archive-keyring.gpg] https://yousef-muc.github.io/napctl/apt stable main" | sudo tee /etc/apt/sources.list.d/napctl.list >/dev/null
sudo apt update
sudo apt install napctl
```

Or use the one-line installer:

```sh
curl -fsSL https://yousef-muc.github.io/napctl/install.sh | sh
```

## RPM-Based Linux

Configure the DNF/YUM repository:

```sh
sudo curl -fsSL https://yousef-muc.github.io/napctl/rpm/napctl.repo -o /etc/yum.repos.d/napctl.repo
sudo dnf install napctl
```

Or use the one-line installer:

```sh
curl -fsSL https://yousef-muc.github.io/napctl/install.sh | sh
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
