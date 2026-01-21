# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Sandbox Factory creates secure, isolated Lima VMs on macOS for running AI coding agents (like Claude Code) with `--dangerously-skip-permissions`. VMs have no host filesystem access and restricted network (allowlisted domains only, ports 80/443/22/53).

## Architecture

```
macOS Host
  └─ Lima (VM hypervisor)
      ├─ VM 1 (agent-owner-repo-suffix)
      │  └─ vim, claude, git, docker
      └─ NAT + Network Filtering → Internet (allowlisted domains only)
```

**Key components:**
- `scripts/sandbox-factory` - Main CLI (bash) for VM lifecycle management
- `lima/agent-sandbox.yaml` - Lima VM template with inline provisioning scripts
- `config/` - repos.txt, network-allowlist.txt, resources.yaml

## Commands

```bash
# Interactive mode - select from running VMs or configured repos
sandbox-factory

# Create VM for specific repo
sandbox-factory owner/repo

# Management
sandbox-factory --list              # List all agent VMs
sandbox-factory --shell <vm-name>   # Shell into VM
sandbox-factory --stop <vm-name>    # Stop VM (preserves state)
sandbox-factory --destroy <vm-name> # Destroy VM (irreversible)
```

**Required:** `GITHUB_TOKEN` env var, `limactl` and `fzf` installed.

## Development Notes

- VM naming: `agent-{owner}-{repo}-{random-suffix}`
- All provisioning logic lives inline in `lima/agent-sandbox.yaml` - test changes carefully
- GitHub token injected into VMs via `~/.zshenv`
- VMs run Ubuntu 24.04 with Docker, Node.js (nvm), Claude Code, Playwright pre-installed
- No host filesystem mounts (critical for isolation)

## Project Status

- **Phase 1 (MVP):** Complete - VM creation, iptables port filtering
- **Phase 2:** In progress - DNS-based domain filtering not yet implemented
- **Phase 3:** Future - 1Password integration, homelab support

See `SPEC.md` for full architecture specification.
