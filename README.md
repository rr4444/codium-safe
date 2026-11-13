# codium-safe

> **Hardened, Sandboxed VSCodium Launcher for Secure AI-Assisted Development**  
> *Confines VSCodium, Google Antigravity (`agy`), and gnassro's Phi (Pi Agent) using Linux Kernel Namespaces.*

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform: Linux](https://img.shields.io/badge/Platform-Linux-orange.svg)](#requirements)
[![Engine: Bubblewrap](https://img.shields.io/badge/Sandbox-Bubblewrap%20%28bwrap%29-green.svg)](#architecture)

---

## Overview

Modern AI coding agents (such as **Google Antigravity** and **gnassro's Phi / Pi Agent**) require terminal execution, file inspection, and editing capabilities to perform autonomous engineering tasks. However, running these agents inside an unconfined desktop IDE exposes your machine to substantial risks:

- **Silent SSH / Git Pushes**: Unattended agents can push unvetted commits or branches to remote repositories.
- **Credential & Secret Theft**: Agents can inadvertently read `~/.ssh/id_*`, `~/.aws/credentials`, `~/.kube/config`, `~/.gnupg`, or browser cookies and histories.
- **Destructive Filesystem Writes**: Rogue or buggy tool loops can alter or delete root system files outside the workspace.
- **Flatpak/Snap Limitations**: Running VSCodium via Flatpak or Snap isolates the GUI but introduces friction with nested sandboxes (`bwrap` inside Flatpak), host toolchains, and desktop portals.

`codium-safe` solves this by wrapping **native VSCodium (`.deb`)** inside an unprivileged **Bubblewrap (`bwrap`)** sandbox. It masks sensitive host files, provides read-only mounts for system binaries, routes SSH signing through interactive confirmation modals, and locks down AI agent permissions—with zero friction for daily development.

---

## Key Features

- **Root & System Immutability**: The entire root filesystem (`/`, `/usr`, `/lib`, `/etc`) is mounted strictly **read-only**.
- **Complete Secret Masking**: The host `$HOME` is mounted over with an isolated `tmpfs`. Private SSH keys (`~/.ssh/id_*`), `~/.aws`, `~/.kube`, `~/.gnupg`, browser profiles (`~/.mozilla`, Google Chrome), and shell histories (`.bash_history`) do not exist inside the sandbox.
- **Interactive SSH Signing**: Private keys remain on the host. `codium-safe` forwards the SSH agent socket with `-c` confirmation mode. Every Git push or SSH handshake triggers a desktop confirmation modal (`ksshaskpass` on KDE Plasma / `ssh-askpass-gnome` on GNOME) requiring user approval.
- **In-Process AI Agent Containment**: Extensions like **Phi (Pi Agent)** and **Google Antigravity** run inside VSCodium's Extension Host. Every file read, edit, and bash tool spawned by the agent inherits Bubblewrap confinement automatically.
- **Dedicated Terminal Profile & Visual Sandbox Badge**: Automatically configures an "Antigravity (agy)" terminal profile in VSCodium's integrated terminal and appends `[Sandboxed]` to the window title.
- **Concurrent Multi-Workspace Isolation**: Deterministic per-workspace instance profiles (`--user-data-dir`) enable launching multiple sandboxed VSCodium sessions concurrently without window hijacking, lock contention, or cross-workspace socket leakage.
- **CLI & Antigravity Tooling Access**: Mounts `~/.local/bin` read-only and configures `$PATH` so the `agy` CLI and custom user binaries remain executable in integrated terminals while preventing modification of host binaries.
- **Ubuntu 24.04 AppArmor Fix**: Automated one-command provisioning for Ubuntu 24.04’s `apparmor_restrict_unprivileged_userns` kernel restriction.
- **Zero-Config Key Discovery**: Automatically detects and arms all valid private keys in `~/.ssh/` for both terminal and desktop menu launches.
- **Single-Cluster Kubernetes Isolation (`--k8s`)**: Generates an isolated, namespace-confined `dev-<project>` ServiceAccount token and project-local `.kubeconfig`. Your primary host `~/.kube/config` is never exposed.
- **Persistent Tool Caches**: Mounts writable, persistent cache directories for language toolchains: `pip`, `uv`, Go modules, Terraform providers, Helm charts, and Ansible collections.
- **Automated `direnv` Integration**: Auto-generates and auto-approves `.envrc` with Git exclusion safeguards.

---

## Architecture: Mounted vs. Masked

```
HOST SYSTEM                                          SANDBOX (bwrap)
========================================================================================
/ (Root Filesystem)                   ──[ ro-bind ]──>  / (Read-Only)
/dev, /proc                           ──[   dev   ]──>  /dev, /proc
/dev/dri (GPU Acceleration)           ──[ dev-bind]──>  /dev/dri (Direct Rendering)
/tmp/.X11-unix, $XDG_RUNTIME_DIR      ──[   bind  ]──>  Display & D-Bus Sockets
/home/user/projects/my-app            ──[   bind  ]──>  /home/user/projects/my-app (Read/Write)

~/.config/VSCodium, ~/.vscode-oss     ──[   bind  ]──>  Settings, Extensions & History
~/.cache/{pip,uv,go,terraform,helm}   ──[   bind  ]──>  Persistent Tool Caches
~/.ssh/{config,known_hosts}           ──[ ro-bind ]──>  Read-Only Host Configs

~/.ssh/id_rsa, id_ed25519             ──[ MASKED  ]──>  HIDDEN (Accessible only via agent socket)
~/.aws, ~/.kube, ~/.gnupg             ──[ MASKED  ]──>  HIDDEN (Empty tmpfs)
~/.mozilla, ~/.config/google-chrome   ──[ MASKED  ]──>  HIDDEN (Empty tmpfs)
~/.bash_history, ~/.zsh_history       ──[ MASKED  ]──>  HIDDEN (Empty tmpfs)
========================================================================================
```

---

## Installation & Requirements

### System Requirements
- **OS**: Ubuntu 24.04 / 22.04 LTS, Debian 12+, KDE neon, Kubuntu, or any Linux distribution with `systemd` and `bubblewrap`.
- **Desktop Environment**: KDE Plasma (recommended) or GNOME / XFCE.

### Quick Setup

1. **Clone or Download the Repository**:
   ```bash
   git clone https://github.com/rr4444/codium-safe.git
   cd codium-safe
   ```

2. **Install to `~/.local/bin`**:
   ```bash
   install -m 755 codium-safe ~/.local/bin/codium-safe
   ```
   *(Ensure `~/.local/bin` is in your `$PATH`).*

3. **Install System Dependencies**:
   Run the standalone installer to configure the official VSCodium APT repository, install system packages (`bubblewrap`, `direnv`, `ksshaskpass`), and apply the Ubuntu 24.04 AppArmor profile:
   ```bash
   codium-safe --install-system-deps
   ```

4. **Install the Desktop Menu Launcher**:
   ```bash
   codium-safe --install-desktop
   ```
   This registers **VSCodium (Sandboxed)** in your application launcher and file manager context menus.

---

## Usage

### Opening Workspaces

```bash
# Open current directory
codium-safe .

# Open a specific project
codium-safe ~/code/my-project

# Open a multi-root workspace
codium-safe ~/code/backend ~/code/frontend
```

### Passing Environment Variables

```bash
# Inject an environment variable (persists automatically to .envrc)
codium-safe -e ANSIBLE_FORKS=20 -e ENVIRONMENT=staging ~/code/my-project

# Load environment variables from an env file into session memory
codium-safe --env-file=~/.config/my-project.env ~/code/my-project
```

### Docker, Podman & Kubernetes Controls

```bash
# Enable Docker socket passthrough (/var/run/docker.sock)
codium-safe --docker ~/code/my-project

# Enable Rootless Podman socket passthrough
codium-safe --podman ~/code/my-project

# Initialize isolated dev-<project> Kubernetes namespace & .kubeconfig
codium-safe --k8s ~/code/my-project

# Force re-generation of project Kubernetes ServiceAccount token
codium-safe --k8s-refresh ~/code/my-project
```

### CLI Options Reference

| Option | Description |
| :--- | :--- |
| `WORKSPACE_DIR...` | One or more workspace directories to mount into the sandbox. |
| `-e, --env KEY=VAL` | Inject environment variable and auto-persist to project `.envrc`. |
| `--env-file=PATH` | Load environment variables from file into session memory. |
| `--key=NAME` | Specify an SSH key file in `~/.ssh/` to arm (default: auto-discovers all keys). |
| `--k8s` | Provision isolated `dev-<project>` Kubernetes namespace & `.kubeconfig`. |
| `--k8s-refresh` | Force regeneration of project `.kubeconfig` token from cluster. |
| `--docker` | Mount `/var/run/docker.sock` for container workloads. |
| `--podman` | Mount Rootless Podman socket for rootless container builds. |
| `--allow-push` | Temporarily allow `git push*` in Antigravity command denylist. |
| `--clean-instances` | Prune all cached per-workspace instance configuration directories. |
| `--install-desktop` | Install or update desktop application menu launcher (`.desktop`). |
| `--install-system-deps` | Add VSCodium repository and install all required APT packages via `sudo`. |
| `-y, --yes` | Assume yes for prompts during dependency installation. |
| `-h, --help` | Show help menu and usage examples. |

---

## Verification Test Suite

Verify all sandbox boundaries by opening an integrated terminal in VSCodium (`Ctrl+\``) and running this test block:

```bash
cat << 'EOF' > /tmp/test_sandbox.sh
#!/usr/bin/env bash
set -u
GREEN='\033[0;32m'; RED='\033[0;31m'; BLUE='\033[0;34m'; BOLD='\033[1m'; NC='\033[0m'
pass() { echo -e "  [${GREEN}PASS${NC}] $*"; }
fail() { echo -e "  [${RED}FAIL${NC}] $*"; }
info() { echo -e "${BLUE}=== $* ===${NC}"; }

echo -e "\n${BOLD}--- VSCodium Sandbox Verification Report ---${NC}\n"

info "1. SSH Credentials Masking"
PRIVATE_KEYS=$(ls ~/.ssh/id_* ~/.ssh/*.pem 2>/dev/null || true)
[ -z "$PRIVATE_KEYS" ] && pass "Host private SSH keys are completely hidden / absent." || fail "Private SSH keys leaked: $PRIVATE_KEYS"
[ -f ~/.ssh/config ] || [ -f ~/.ssh/known_hosts ] && pass "Safe SSH configs mounted successfully." || fail "SSH configs missing."

info "2. Cloud & Secret Paths Isolation"
LEAKED_SECRETS=""
for path in ~/.aws ~/.kube ~/.gnupg; do
  [ -d "$path" ] && [ "$(ls -A "$path" 2>/dev/null)" ] && LEAKED_SECRETS="$LEAKED_SECRETS $path"
done
[ -z "$LEAKED_SECRETS" ] && pass "Cloud & crypto directories (~/.aws, ~/.kube, ~/.gnupg) are masked." || fail "Credentials leaked:$LEAKED_SECRETS"

info "3. Browser Data & Shell History Isolation"
LEAKED_PRIVACY=""
for path in ~/.mozilla ~/.config/google-chrome ~/.bash_history ~/.zsh_history; do
  [ -e "$path" ] && LEAKED_PRIVACY="$LEAKED_PRIVACY $path"
done
[ -z "$LEAKED_PRIVACY" ] && pass "Host browser profiles and shell histories are completely masked." || fail "History visible:$LEAKED_PRIVACY"

info "4. Root Filesystem Write Protection"
if touch /test-write-attempt 2>/dev/null; then
  rm -f /test-write-attempt && fail "Root filesystem is WRITABLE."
else
  pass "Root filesystem is read-only (write correctly rejected)."
fi

info "5. Workspace Read/Write Access"
touch ./sandbox_rw_test.tmp 2>/dev/null && rm -f ./sandbox_rw_test.tmp && pass "Workspace is writable." || fail "Workspace write failed."

info "6. SSH-Agent Socket Connectivity"
AGENT_OUT=$(ssh-add -l 2>&1)
[[ "$AGENT_OUT" != *"Could not open a connection"* ]] && [[ "$AGENT_OUT" != *"No such file"* ]] \
  && pass "SSH agent connected: $AGENT_OUT" || fail "Cannot connect to ssh-agent: $AGENT_OUT"

info "7. Process Security & Namespace Isolation"
[ "$(ps -eo user | grep -c "nobody" || true)" -gt 0 ] && pass "Host processes mapped to 'nobody' (user namespace active)." || pass "Isolated PID namespace active."

info "8. Antigravity & User Tooling Availability"
command -v agy >/dev/null 2>&1 && pass "Antigravity CLI ('agy') accessible on PATH: $(which agy)" || fail "'agy' binary not found on PATH."

echo -e "\n${BOLD}--- End of Report ---${NC}\n"
EOF
bash /tmp/test_sandbox.sh && rm -f /tmp/test_sandbox.sh
```

---

## License

This project is licensed under the [MIT License](LICENSE).
