# Sandbox Factory Spec

A tool to create secure, isolated VM environments locally for running AI coding agents autonomously in YOLO mode.

## Goals

1. **Full isolation** - Agents run in true VMs with kernel-level separation from host
2. **Network control** - Granular allowlist of domains and ports
3. **Disposability** - VMs are ephemeral; destruction loses nothing important
4. **Safe autonomy** - Run `claude --dangerously-skip-permissions` without risk
5. **Familiar workflow** - Personalize dotfiles, install vim, git, etc as desired
6. **Multi-agent** - Support multiple concurrent agent VMs

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│  macOS Host                                                          │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Lima                                                            │  │
│  │                    ┌─────────┐    ┌─────────┐                   │  │
│  │                    │  VM 1   │    │  VM 2   │                   │  │
│  │                    │ repo-a  │    │ repo-b  │                   │  │
│  │                    │         │    │         │                   │  │
│  │                    │  vim    │    │  vim    │                   │  │
│  │                    │  claude │    │  claude │                   │  │
│  │                    │  git    │    │  git    │                   │  │
│  │                    └────┬────┘    └────┬────┘                   │  │
│  │                         │              │                        │  │
│  │                    ┌────┴──────────────┴────┐                   │  │
│  │                    │    NAT + Filtering     │                   │  │
│  │                    └───────────┬────────────┘                   │  │
│  └────────────────────────────────┼────────────────────────────────┘  │
│                                   │                                   │
└───────────────────────────────────┼───────────────────────────────────┘
                                    │
                          ┌─────────┴─────────┐
                          │    Internet       │
                          │ (allowed domains  │
                          │  only)            │
                          └───────────────────┘
```

## Components

### 1. Lima VM Configuration

**File:** `lima/agent-sandbox.yaml`

```yaml
vmType: vz                      # Apple Virtualization framework (fast)
cpus: 4                         # Configurable per-VM
memory: 8GiB                    # Configurable per-VM
disk: 50GiB                     # Configurable per-VM

images:
  - location: "https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-arm64.img"
    arch: "aarch64"
  - location: "https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-amd64.img"
    arch: "x86_64"

# CRITICAL: No host mounts for isolation
mounts: []

ssh:
  localPort: 0                  # Auto-assign port (avoids conflicts)
  forwardAgent: false           # Don't expose host SSH agent

containerd:
  system: false                 # We'll use Docker instead
  user: false

provision:
  - mode: system
    script: |
      #!/bin/bash
      set -eux
      # Base provisioning - see provision.sh for full script
```

### 2. Network Filtering

**Approach:** iptables port restrictions

**Allowed:**
- Outbound ports 80 (HTTP), 443 (HTTPS), 22 (SSH), 53 (DNS)

**Blocked:**
- All other outbound ports
- All inbound connections

**Implementation:** iptables rules in `/etc/rc.local` inside VM

### 3. Agent Sessionizer Script

**File:** `scripts/sandbox-factory`

**Workflow:**
```
User triggers hotkey (Alt+Shift+P)
         │
         ▼
┌─────────────────────────┐
│  fzf: Select repo or    │
│  running VM             │
└───────────┬─────────────┘
            │
    ┌───────┴───────┐
    │               │
    ▼               ▼
[New repo]    [Running VM]
    │               │
    ▼               │
Create new VM       │
Clone repo          │
    │               │
    ▼               ▼
Connect to VM via SSH
```

**Interface:**
```bash
# Start new agent session
sandbox-factory                    # Interactive fzf selection
sandbox-factory owner/repo         # Direct repo specification

# List running agent VMs
sandbox-factory --list

# Stop/destroy a VM
sandbox-factory --stop owner/repo
sandbox-factory --destroy owner/repo
```

### 4. Base Image Provisioning

**File:** `lima/provision.sh`

**Installed tools:**
- Docker
- Node.js (via nvm)
- Git, vim, zsh
- lazygit
- gh CLI
- Claude Code
- Playwright dependencies
- Dotfiles (subset for VM environment)

**VM-specific dotfiles:**
- `.zshrc` (simplified)
- `.gitconfig` (without personal email - uses repo config)
- `.claude/settings.json` (permissive for sandbox)

### 5. Configuration Files

**Directory structure:**
```
sandbox/
├── SPEC.md                     # This file
├── config/
│   └── repos.txt               # List of repos for fzf
├── lima/
│   └── agent-sandbox.yaml      # Lima VM template (includes provisioning)
└── scripts/
    └── sandbox-factory         # Main CLI entry point
```

### 6. Secrets Management

**Secrets needed:**
- GitHub PAT (scoped: repo read/write, no delete, no admin)
- Anthropic credentials (TBD - see open questions)

**Injection method:**
- Passed as environment variables to VM
- Never baked into base image

```bash
# Example: inject secrets at VM start
export GITHUB_TOKEN=your_token
limactl start --name=agent-foo ./lima/agent-sandbox.yaml
limactl shell agent-foo -- bash -c "echo 'export GITHUB_TOKEN=${GITHUB_TOKEN}' >> ~/.zshenv"
```

## Workflow Details

### Creating a New Agent Session

```bash
# 1. User triggers sandbox-factory
$ sandbox-factory

# 2. fzf shows repos from config/repos.txt
> anthropics/claude-code
  your-org/some-repo
  your-org/another-repo

# 3. User selects repo, script runs:
#    a. Generate unique VM name: agent-anthropics-claude-code-a1b2c3
#    b. Start Lima VM from template
#    c. Inject secrets
#    d. Clone repo

# 4. User connects to VM via: limactl shell <vm-name>
```

### Switching Between Agent Sessions

```bash
# Same hotkey (Alt+Shift+P) shows running VMs at top of fzf
$ sandbox-factory

> [RUNNING] anthropics/claude-code     # Switch to existing
  [RUNNING] your-org/some-repo         # Switch to existing
  ─────────────────────────────────
  another-org/new-repo                  # Start new
```

### Destroying an Agent Session

```bash
# Manual destruction
$ sandbox-factory --destroy anthropics/claude-code
```

## Security Model

### Trust Boundaries

```
┌─────────────────────────────────────────────────────────┐
│ TRUSTED: macOS Host                                     │
│ - Your files, credentials, network access               │
│ - Lima daemon                                           │
│                                                         │
├─────────────────────────────────────────────────────────┤
│ UNTRUSTED: Agent VM                                     │
│ - Can run arbitrary code                                │
│ - Has scoped GitHub token (can't push to main)          │
│ - Has Anthropic credentials (TBD)                       │
│ - Limited network access                                │
│ - No access to host filesystem                          │
│ - Disposable - destruction loses nothing                │
└─────────────────────────────────────────────────────────┘
```

### What the Agent CAN Do

- Read/write code in cloned repo
- Create branches and push to remote
- Open pull requests
- Install packages (npm, apt, etc.)
- Run Docker containers inside VM
- Run development servers
- Run tests
- Run Playwright browser automation
- Make HTTP requests to allowed domains
- Use full Claude Code capabilities

### What the Agent CANNOT Do

- Access host filesystem
- Access host network services
- Push to main/master branch (GitHub protection)
- Access domains not in allowlist
- Make connections on non-allowed ports
- Access other VMs
- Persist state after VM destruction
- Access your real credentials (only scoped tokens)

### Blast Radius

If an agent is compromised or goes rogue:
- **Worst case:** Pushes garbage to a feature branch
- **Recovery:** Delete branch, revoke PAT, destroy VM
- **Cannot:** Access your files, credentials, or other systems

## Resource Limits

**Per-VM defaults** (hardcoded in `lima/agent-sandbox.yaml`):
- CPUs: 4
- Memory: 8GiB
- Disk: 50GiB

**System-wide:**
- No GPU passthrough (prevents crypto mining)

## Getting Started

### Prerequisites

```bash
# Install Lima
brew install lima

# Verify installation
limactl --version
```

### Quick Start

```bash
# 1. Add repos to config
echo "your-org/your-repo" >> config/repos.txt

# 2. Set GitHub token
export GITHUB_TOKEN=your_scoped_pat

# 3. Run sandbox-factory
./scripts/sandbox-factory
```

### Adding to PATH

Add to your shell config:
```bash
export PATH="$HOME/Developer/sandbox/scripts:$PATH"
```

Or create a symlink:
```bash
ln -s ~/Developer/sandbox/scripts/sandbox-factory ~/.local/bin/
```

---

*Last updated: 2026-01-20*
