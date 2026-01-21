# Network Filtering Implementation Spec

## Goal
Implement host-level network filtering for Lima VMs that the agent cannot bypass.

## Architecture Overview
```
macOS Host (agent cannot access)
├── pf firewall (redirects VM traffic to proxy)
├── mitmproxy (filters by domain allowlist)
├── socket_vmnet (creates bridge interface for VM)
└── Lima VM (agent runs here, isolated)
```

## Prerequisites (Manual, One-Time)

### 1. Install socket_vmnet
```bash
# Secure installation (not Homebrew - Lima docs recommend against it)
git clone https://github.com/lima-vm/socket_vmnet.git
cd socket_vmnet
sudo make PREFIX=/opt/socket_vmnet install
```

### 2. Install mitmproxy
```bash
brew install mitmproxy
```

### 3. Configure sudoers for socket_vmnet
```bash
limactl sudoers | sudo tee /etc/sudoers.d/lima
```

---

## Implementation

### Step 1: Create allowlist config
**File:** `config/allowlist.txt`

Contains domains the VM can access:
```
.github.com
.githubusercontent.com
.npmjs.org
.npmjs.com
.yarnpkg.com
.pypi.org
.pythonhosted.org
.docker.io
.docker.com
.anthropic.com
.googleapis.com
.ubuntu.com
.cloudfront.net
```

### Step 2: Create pf rules template
**File:** `config/pf-sandbox.conf.template`

Note: `BRIDGE_INTERFACE` is replaced dynamically (usually `bridge100` but can vary).

```
# Redirect HTTP/HTTPS from VM bridge to mitmproxy
rdr pass on $BRIDGE_INTERFACE inet proto tcp from any to any port {80, 443} -> 127.0.0.1 port 8080

# Allow DNS
pass on $BRIDGE_INTERFACE inet proto udp from any to any port 53
pass on $BRIDGE_INTERFACE inet proto tcp from any to any port 53

# Block everything else from VM
block drop on $BRIDGE_INTERFACE inet proto tcp from any to any
block drop on $BRIDGE_INTERFACE inet proto udp from any to any
```

The CLI will detect the bridge interface using `ifconfig | grep -o 'bridge[0-9]*'` after socket_vmnet starts.

### Step 3: Create proxy startup script
**File:** `scripts/start-proxy`

```bash
#!/usr/bin/env bash
# Starts mitmproxy with domain allowlist filtering
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ALLOWLIST="$SCRIPT_DIR/../config/allowlist.txt"

# Convert allowlist to mitmproxy regex
ALLOW_REGEX=$(grep -v '^#' "$ALLOWLIST" | grep -v '^$' | sed 's/\./\\./g' | sed 's/^/^(.*\\.)?/' | sed 's/$/$/' | paste -sd'|' -)

exec mitmproxy --mode transparent --listen-port 8080 --allow-hosts "$ALLOW_REGEX" --showhost
```

### Step 4: Create LaunchDaemon for pf rules
**File:** `config/com.sandbox-factory.pf.plist`

LaunchDaemon to load pf rules at boot and when sandbox starts.

### Step 5: Update Lima template
**File:** `lima/agent-sandbox.yaml`

Changes:
- Remove `networks: [vzNAT: true]`
- Add socket_vmnet shared network config
- Remove the iptables rules (no longer needed)
- Keep the rest of provisioning as-is

```yaml
networks:
  - lima: shared
```

### Step 6: Update CLI
**File:** `scripts/sandbox-factory`

Add new commands/functions:
- `setup_network_filter()` - loads pf rules, starts proxy
- `teardown_network_filter()` - unloads pf rules, stops proxy
- `--setup` command for one-time host setup
- Integrate filter start/stop with VM lifecycle

New workflow:
1. `sandbox-factory --setup` (one-time): installs LaunchDaemon, configures sudoers
2. `sandbox-factory owner/repo`: starts proxy, loads pf rules, then starts VM
3. When all VMs stop: unloads pf rules, stops proxy

---

## Files to Create
| File | Purpose |
|------|---------|
| `config/allowlist.txt` | Domain allowlist |
| `config/pf-sandbox.conf.template` | pf firewall rules template |
| `config/com.sandbox-factory.pf.plist` | LaunchDaemon for persistence |
| `scripts/start-proxy` | mitmproxy launcher with allowlist |

## Files to Modify
| File | Changes |
|------|---------|
| `lima/agent-sandbox.yaml` | Switch to socket_vmnet, remove iptables |
| `scripts/sandbox-factory` | Add network filter management |
| `CLAUDE.md` | Update documentation |

---

## Verification

1. **Test pf rules load correctly:**
   ```bash
   sudo pfctl -f config/pf-sandbox.conf
   sudo pfctl -sr  # should show rules
   ```

2. **Test proxy starts:**
   ```bash
   ./scripts/start-proxy  # should see mitmproxy UI
   ```

3. **Test VM creation with new network:**
   ```bash
   sandbox-factory test/repo
   ```

4. **Test filtering inside VM:**
   ```bash
   # Should work (in allowlist)
   curl https://github.com

   # Should fail (not in allowlist)
   curl https://example.com

   # Agent cannot bypass
   unset HTTP_PROXY HTTPS_PROXY
   curl https://example.com  # still fails
   ```

5. **Test non-HTTP blocked:**
   ```bash
   # Inside VM - should fail (port 8080 not allowed)
   nc -zv somehost.com 8080
   ```
