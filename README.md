# Home SIEM Lab — Wazuh Log Analysis & Detection

## Overview
A home SIEM lab built using Wazuh, deployed across two physical Linux
machines to simulate a monitored endpoint and a security operations
dashboard. Built to demonstrate log analysis, detection engineering,
security hardening assessment, and incident response — using only
real hardware, no virtual machines.

## Architecture

- **Manager / Dashboard host:** Linux Mint (HP Pavilion dv6), running
  Wazuh manager, indexer, and dashboard via Docker Compose (single-node
  deployment, Wazuh v4.9.2)
- **Monitored endpoint / attack host:** Kali Linux (Dell, i5 4th gen),
  running the Wazuh agent — used both as the monitored machine and as
  the source of simulated attack traffic
- **Network:** both machines on the same LAN, manager assigned a static IP (192.168.1.96)

<!-- Add architecture diagram here: two laptops, arrows showing agent → manager log flow -->

## Setup Steps
1. Set static IP on manager host, confirmed connectivity between both machines
2. Installed Docker, deployed Wazuh single-node stack via `wazuh-docker` (v4.9.2 branch)
3. Verified OpenSearch indexer cluster health (`green`, 9 active shards)
4. Confirmed dashboard accessible over HTTPS on port 443
5. Installed Wazuh agent on Kali host, pinned to matching version 4.9.2
6. Verified agent connection in the dashboard (agent status: Active)
7. Ran baseline Security Configuration Assessment (SCA) scan
8. Applied SSH hardening fixes, confirmed 100% SCA pass on rescan
9. Simulated SSH brute-force attack with Hydra, confirmed detection in Wazuh

## Findings

### 1. Baseline SCA Scan — SSH Hardening (Before)
Wazuh's built-in SCA module flagged the following SSH hardening gaps on
the Kali host out of the box:

| Check | Result |
|---|---|
| Port should not be 22 | Failed |
| Protocol should be set to 2 | Failed |
| Root account should not be able to login | Failed |
| No public key authentication | Failed |
| Password authentication | Failed |
| Empty passwords should be disabled | Failed |
| Rhost or shost should not be used | Failed |
| Grace time should be limited | Failed |
| Wrong maximum number of authentication attempts | Failed |

*(screenshot: SCA results — all failed)*

### 2. SSH Hardening Applied (After)
Modified `/etc/ssh/sshd_config` to fix all flagged checks:
- Changed default port from 22 to a non-standard port
- Set `PermitRootLogin no`
- Set `PasswordAuthentication no` (later re-enabled temporarily for attack testing)
- Set `PermitEmptyPasswords no`
- Set `LoginGraceTime 30`
- Set `Protocol 2`

**Result: 100% SCA score — 16 passed, 0 failed**

*(screenshot: SCA dashboard showing 100% score)*

### 3. Simulated SSH Brute-Force Attack
**Attack:** Used Hydra to simulate 8 repeated SSH login attempts with
wrong passwords against the monitored Kali host (password auth
temporarily re-enabled for the test):

```bash
for i in 1 2 3 4 5 6 7 8; do
  echo "wrongpass$i" > /tmp/pw.txt
  hydra -l treesups -P /tmp/pw.txt -t 1 ssh://localhost:3221
  sleep 1
done
```

**Detection result:** Wazuh detected all 26 authentication failure events
(8 Hydra attempts + related PAM events), visible as a spike in the
Threat Hunting dashboard at the exact time of the attack.

| Metric | Value |
|---|---|
| Total alerts generated | 26 |
| Authentication failures detected | 26 |
| Authentication successes | 0 (attack failed) |
| Detecting agent | kali |

*(screenshot: Threat Hunting dashboard showing 26 auth failure spike)*

Alerts also confirmed directly in the manager's alerts log:
```
Failed password for treesups from 127.0.0.1 port 42620 ssh2
Failed password for treesups from 127.0.0.1 port 42648 ssh2
... (8 total)
```

## What I Learned

- **Agent-manager version mismatch:** Wazuh agent must be the same version
  or older than the manager. Fixed by pinning the agent install to
  `wazuh-agent=4.9.2-1` to match the Docker stack version.

- **PAM module lockout:** Adding a `pam_tally2.so` line to
  `/etc/pam.d/common-auth` during SSH hardening locked out all logins
  entirely — `pam_tally2` is deprecated and removed in modern Kali/Debian,
  and `auth required` means PAM fails the entire auth chain if the module
  doesn't exist. Recovered via a Kali live USB, chrooting into the locked
  system, and removing the bad PAM line. Key lesson: always verify module
  availability before adding `auth required` PAM entries.

- **Live USB recovery:** Used Rufus to create a bootable Kali live USB,
  identified the correct disk by partition size (not device letter — drive
  letters can change between boots), mounted the root filesystem, and
  chrooted in to fix configs without reinstalling.

- **Docker container startup order:** Wazuh dashboard reports "not ready"
  immediately after `docker compose up` because it starts before OpenSearch
  finishes initializing. The fix is to wait for the indexer health check
  to return `green` before expecting the dashboard to respond.

- **Detection vs hardening are complementary:** With `PasswordAuthentication no`,
  Hydra couldn't even attempt password guessing — the brute-force was blocked
  at the protocol level before Wazuh ever saw a failed login. Temporarily
  re-enabled for testing to demonstrate the detection capability separately
  from the hardening, then re-disabled after.

## Tools Used
- Wazuh 4.9.2 (manager, indexer, dashboard, agent)
- Docker / Docker Compose
- Kali Linux — agent host, Hydra brute-force simulation
- Linux Mint — manager host
- Hydra — SSH brute-force simulation
- OpenSSH / sshd_config — hardening target

---

## Session Notes Log

### 2026-09-17
- Ran Hydra brute-force loop (8 attempts) against SSH on Kali host
- Confirmed all 8 failed attempts in `/var/log/auth.log`
- Confirmed 26 alerts generated in Wazuh manager alerts.log
- Threat Hunting dashboard showed spike of 26 auth failures at attack time
- Project core complete: setup → hardening → attack → detection all documented

### 2026-09-13
- Applied SSH hardening changes to `/etc/ssh/sshd_config`
- Triggered a full system lockout due to deprecated `pam_tally2.so` in PAM config
- Recovered via Kali live USB + chroot — removed bad PAM line, reset password
- Re-ran SCA scan: 100% score, 16 passed, 0 failed
- Re-enabled SSH service after recovery (`systemctl enable/start ssh`)

### 2026-09-06
- Fixed agent registration by matching agent version to manager (4.9.2)
- Agent connected successfully, confirmed in dashboard Agents view
- Reviewed first SCA scan results (SSH hardening failures)

### 2026-09-03
- Deployed Wazuh single-node stack via Docker on Mint laptop
- Hit dashboard "not ready" issue — resolved by restarting dashboard container
  after confirming indexer health was green
- Dashboard accessible over HTTPS, logged in successfully
