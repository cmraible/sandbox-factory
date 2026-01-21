# Sandbox Factory

Create secure, isolated VMs for running AI coding agents autonomously. VMs have no host filesystem access and restricted network (allowlisted domains only).

## Dependencies

```bash
brew install lima fzf
```

## Setup

1. Add repos to `config/repos.txt` (one per line, e.g., `owner/repo`)

2. Set GitHub token (required for private repos and pushing):
   ```bash
   export GITHUB_TOKEN=your_token
   ```

## Usage

```bash
# Interactive selection
./scripts/sandbox-factory

# Create VM for specific repo
./scripts/sandbox-factory owner/repo

# List all VMs
./scripts/sandbox-factory --list

# Shell into a VM
./scripts/sandbox-factory --shell <vm-name>

# Stop a VM (preserves state)
./scripts/sandbox-factory --stop <vm-name>

# Destroy a VM (irreversible)
./scripts/sandbox-factory --destroy <vm-name>
```

## How It Works

- Creates Lima VMs running Ubuntu 24.04 with Docker, Node.js, Claude Code, and common dev tools
- Clones the selected repo into `~/workspace/repo`
- Network restricted to ports 80/443/22/53 (HTTP, HTTPS, SSH, DNS)
- No host filesystem mounts for full isolation

See [SPEC.md](SPEC.md) for full architecture details.
