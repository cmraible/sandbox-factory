# Agent Sandbox Spec

A secure, isolated VM environment for running AI coding agents autonomously.

## Goals

1. **Full isolation** - Agents run in true VMs with kernel-level separation from host
2. **Network control** - Granular allowlist of domains and ports
3. **Disposability** - VMs are ephemeral; destruction loses nothing important
4. **Safe autonomy** - Run `claude --dangerously-skip-permissions` without risk
5. **Familiar workflow** - Integrate with existing tmux-sessionizer pattern
6. **Multi-agent** - Support 1-10 concurrent agent VMs

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  macOS Host                                                         │
│                                                                     │
│  ┌─────────────────┐    ┌─────────────────┐                        │
│  │ tmux session    │    │ tmux session    │                        │
│  │ (repo-a)        │    │ (repo-b)        │                        │
│  │                 │    │                 │                        │
│  │  SSH ──────────────────────┐           │                        │
│  └─────────────────┘    │     │           │                        │
│                         │     │           │                        │
│  ┌──────────────────────┼─────┼───────────┼──────────────────────┐ │
│  │ Lima               ┌─┴─────┴─┐    ┌────┴────┐                 │ │
│  │                    │  VM 1   │    │  VM 2   │                 │ │
│  │                    │ repo-a  │    │ repo-b  │                 │ │
│  │                    │         │    │         │                 │ │
│  │                    │ ┌─────┐ │    │ ┌─────┐ │                 │ │
│  │                    │ │tmux │ │    │ │tmux │ │                 │ │
│  │                    │ │ vim │ │    │ │ vim │ │                 │ │
│  │                    │ │claude│ │   │ │claude│ │                │ │
│  │                    │ │ git │ │    │ │ git │ │                 │ │
│  │                    │ └─────┘ │    │ └─────┘ │                 │ │
│  │                    └────┬────┘    └────┬────┘                 │ │
│  │                         │              │                      │ │
│  │                    ┌────┴──────────────┴────┐                 │ │
│  │                    │    NAT + Filtering     │                 │ │
│  │                    └───────────┬────────────┘                 │ │
│  └────────────────────────────────┼──────────────────────────────┘ │
│                                   │                                 │
└───────────────────────────────────┼─────────────────────────────────┘
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

**Approach:** DNS-based filtering + iptables port restrictions

**Allowed Domains** (configurable in `config/network-allowlist.txt`):
```
# GitHub
github.com
api.github.com
raw.githubusercontent.com
objects.githubusercontent.com

# npm
registry.npmjs.org

# Docker Hub
hub.docker.com
registry-1.docker.io
auth.docker.io
production.cloudflare.docker.com

# Anthropic
api.anthropic.com
claude.ai

# Ubuntu packages
archive.ubuntu.com
security.ubuntu.com

# Node.js
nodejs.org
deb.nodesource.com
```

**Blocked:**
- All other domains (DNS returns NXDOMAIN)
- All ports except 80, 443, 22 outbound
- All inbound connections

**Implementation:**
1. Run `dnsmasq` inside VM as DNS server
2. Configure dnsmasq to only resolve allowlisted domains
3. Use iptables to:
   - Force all DNS through local dnsmasq
   - Block outbound except 80/443/22
   - Block all inbound

### 3. Agent Sessionizer Script

**File:** `scripts/agent-sessionizer`

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
Setup tmux          │
    │               │
    ▼               ▼
Switch to local tmux session
(SSH into VM tmux)
```

**Interface:**
```bash
# Start new agent session
agent-sessionizer                    # Interactive fzf selection
agent-sessionizer owner/repo         # Direct repo specification

# List running agent VMs
agent-sessionizer --list

# Stop/destroy a VM
agent-sessionizer --stop owner/repo
agent-sessionizer --destroy owner/repo
```

### 4. Base Image Provisioning

**File:** `lima/provision.sh`

**Installed tools:**
- Docker
- Node.js (via nvm)
- Git, vim, tmux, zsh
- lazygit
- gh CLI
- Claude Code
- Playwright dependencies
- Dotfiles (subset for VM environment)

**VM-specific dotfiles:**
- `.zshrc` (simplified)
- `.tmux.conf`
- `.gitconfig` (without personal email - uses repo config)
- `.claude/settings.json` (permissive for sandbox)
- `ready-tmux` (modified for agent workflow)

### 5. Configuration Files

**Directory structure:**
```
sandbox/
├── SPEC.md                     # This file
├── config/
│   ├── repos.txt               # List of repos for fzf
│   ├── network-allowlist.txt   # Allowed domains (for Phase 2)
│   └── resources.yaml          # CPU/memory/disk defaults
├── lima/
│   └── agent-sandbox.yaml      # Lima VM template (includes provisioning)
└── scripts/
    └── agent-sessionizer       # Main CLI entry point
```

### 6. Secrets Management

**Secrets needed:**
- GitHub PAT (scoped: repo read/write, no delete, no admin)
- Anthropic credentials (TBD - see open questions)

**Injection method:**
- Secrets stored in 1Password
- Injected at VM start via `op run` or `op read`
- Passed as environment variables to VM
- Never baked into base image

```bash
# Example: inject secrets at VM start
export GITHUB_TOKEN=$(op read "op://Private/GitHub PAT Agent/token")
limactl start --name=agent-foo ./lima/agent-sandbox.yaml
limactl shell agent-foo -- bash -c "echo 'export GITHUB_TOKEN=${GITHUB_TOKEN}' >> ~/.zshenv"
```

## Workflow Details

### Creating a New Agent Session

```bash
# 1. User triggers agent-sessionizer
$ agent-sessionizer

# 2. fzf shows repos from config/repos.txt
> anthropics/claude-code
  your-org/some-repo
  your-org/another-repo

# 3. User selects repo, script runs:
#    a. Generate unique VM name: agent-anthropics-claude-code-a1b2c3
#    b. Start Lima VM from template
#    c. Inject secrets
#    d. Clone repo
#    e. Run ready-tmux inside VM
#    f. Create local tmux session that SSHs to VM

# 4. User is now in tmux session connected to VM
#    Windows: vim | server | claude | git | scratch
```

### Switching Between Agent Sessions

```bash
# Same hotkey (Alt+Shift+P) shows running VMs at top of fzf
$ agent-sessionizer

> [RUNNING] anthropics/claude-code     # Switch to existing
  [RUNNING] your-org/some-repo         # Switch to existing
  ─────────────────────────────────
  another-org/new-repo                  # Start new
```

### Destroying an Agent Session

```bash
# Manual destruction
$ agent-sessionizer --destroy anthropics/claude-code

# Or from within tmux: detach and destroy
# (keybinding TBD, e.g., prefix + D)
```

## Security Model

### Trust Boundaries

```
┌─────────────────────────────────────────────────────────┐
│ TRUSTED: macOS Host                                     │
│ - Your files, credentials, network access               │
│ - Lima daemon                                           │
│ - 1Password                                             │
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

**Per-VM defaults** (configurable in `config/resources.yaml`):

```yaml
defaults:
  cpus: 4
  memory: 8GiB
  disk: 50GiB

# Override per-repo if needed
overrides:
  large-org/monorepo:
    cpus: 8
    memory: 16GiB
    disk: 100GiB
```

**System-wide:**
- Max concurrent VMs: 10 (soft limit, configurable)
- No GPU passthrough (prevents crypto mining)

## Open Questions

### 1. Anthropic Authentication

**Problem:** User wants to use Claude Pro subscription, not API key.

**Options:**
| Option | Pros | Cons |
|--------|------|------|
| API key | Simple, revocable, scoped | Costs money, not Pro |
| Pre-auth in base image | Works offline | Creds may expire, image has secrets |
| Mount ~/.claude from host | Fresh auth always | Breaks isolation |
| Auth once per VM start | Secure | Requires browser, breaks unattended |
| Persistent creds volume | Survives VM destroy | Shared across VMs |

**Recommendation:** TBD - needs more investigation into Claude Pro OAuth flow and token lifetime.

### 2. Homelab Integration

**Current scope:** macOS (Lima works on Linux too)

**Future questions:**
- Same Lima setup on Linux homelab?
- Central VM management across machines?
- Shared config/secrets?

### 3. Agent Task Specification

**Current:** Manual - user starts agent, types instructions

**Future consideration:**
- Pass task description at VM creation?
- Integration with issue trackers (Linear)?
- Automated agent dispatch?

### 4. Monitoring & Observability

**Current:** SSH in and watch

**Future consideration:**
- Log aggregation from VMs?
- Alerts on agent errors?
- Usage/cost tracking?

## Implementation Phases

### Phase 1: MVP ✅
- [x] Lima VM config with isolation (`lima/agent-sandbox.yaml`)
- [x] Basic provisioning script (node, docker, git, claude) - embedded in yaml
- [x] Simple agent-sessionizer (create/list/destroy) (`scripts/agent-sessionizer`)
- [x] Manual secret injection (GITHUB_TOKEN, 1Password optional)
- [x] Basic network filtering (iptables ports only - 80, 443, 22)
- [x] Config files (repos.txt, network-allowlist.txt, resources.yaml)

### Phase 2: Polish
- [ ] DNS-based domain filtering (use network-allowlist.txt)
- [ ] Full dotfiles in VM
- [x] tmux integration (host session → VM session) - basic version done
- [ ] Config-driven resource limits (parse resources.yaml)
- [x] Repo list in config file (config/repos.txt)

### Phase 3: Advanced
- [ ] 1Password integration for secrets (partially done - optional in agent-sessionizer)
- [ ] Homelab support
- [x] Multiple concurrent VMs (supported via unique VM names)
- [ ] Monitoring/logging

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

# 2. Set GitHub token (or configure 1Password)
export GITHUB_TOKEN=your_scoped_pat

# 3. Run agent-sessionizer
./scripts/agent-sessionizer
```

### Adding to PATH

Add to your shell config:
```bash
export PATH="$HOME/Developer/sandbox/scripts:$PATH"
```

Or create a symlink:
```bash
ln -s ~/Developer/sandbox/scripts/agent-sessionizer ~/.local/bin/
```

---

*Last updated: 2026-01-20*
