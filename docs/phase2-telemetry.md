# Phase 2 — Telemetry and Visibility

**Status:** Linux telemetry complete and verified. Windows endpoint pending.
**SIEM:** Wazuh 4.14 all-in-one on `mgmt-wazuh-01` (10.10.10.20)

---

## 1. Telemetry architecture

```
  lnx-user-01 ──┐
  mgmt-jump-01 ─┤── agent (1514/tcp) ──> wazuh-manager
  win-client-01 ┘                             │
                                              │ alerts.json
                                              ▼
                                          filebeat
                                              │
                                              ▼
                                       wazuh-indexer (9200)
                                              │
                                              ▼
                                      wazuh-dashboard (443)
```

Every component runs on a single host. On constrained hardware this is the correct trade-off — a distributed deployment would consume the RAM the endpoints need, and the ingest volume of a lab never approaches the point where separation matters.

---

## 2. Agents onboarded

| ID | Host | Address | Type | Collection |
|---|---|---|---|---|
| 000 | `mgmt-wazuh-01` | 10.10.10.20 | Manager self-monitoring | Local |
| 001 | `mgmt-jump-01` | 10.10.10.10 | Debian 13 LXC | journald |
| 002 | `lnx-user-01` | 10.10.10.40 | Debian 13 VM | journald · auditd · FIM |
| — | `win-client-01` | — | Windows | Planned — Sysmon, Security, PowerShell |

### Why the container is not sufficient

`mgmt-jump-01` is an unprivileged LXC and shares the host kernel. auditd requires kernel audit subsystem access the container does not have, so syscall-level telemetry is impossible there. It proves the ingest path works; it cannot satisfy the Linux telemetry requirements. `lnx-user-01` exists as a full VM specifically for that reason.

This distinction matters operationally: containers and VMs are not interchangeable endpoints from a monitoring standpoint, and assuming otherwise produces a blind spot that looks like coverage.

---

## 3. Linux log sources

| Source | Mechanism | Provides |
|---|---|---|
| journald | `<log_format>journald</log_format>` | SSH authentication, sudo, systemd, service events |
| auditd | `/var/log/audit/audit.log`, `<log_format>audit</log_format>` | Syscall-level execution, file access, privilege use |
| FIM | syscheck, realtime on `/etc` and SSH key directories | File creation, modification, deletion |

Debian 13 ships journald-only, with no rsyslog and therefore no `/var/log/auth.log`. Collecting from journald directly is the correct approach rather than installing rsyslog to recreate a file that the distribution deliberately dropped. Wazuh's stock decoders parse journald output without modification.

---

## 4. Audit ruleset

`/etc/audit/rules.d/cyber-range.rules`

```
## Identity and authorization files
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/gshadow -p wa -k identity
-w /etc/sudoers -p wa -k identity
-w /etc/sudoers.d/ -p wa -k identity

## Privilege escalation binaries
-w /usr/bin/sudo -p x -k sudo_exec
-w /usr/bin/su -p x -k su_exec

## Commands run as root by a normal user
-a always,exit -F arch=b64 -S execve -F euid=0 -F auid>=1000 -F auid!=unset -k privileged_exec

## SSH configuration
-w /etc/ssh/sshd_config -p wa -k sshd_config
```

The `auid>=1000` condition is what makes `privileged_exec` useful: it matches commands run as root *by a logged-in user*, and excludes the constant background noise of system daemons already running as root. Testing this rule requires running commands as an unprivileged user via `sudo` — from a root shell the audit user ID is unset and the rule correctly does not match.

---

## 5. Verification

Telemetry was verified against deliberately generated events rather than assumed working.

| Action | Path | Result |
|---|---|---|
| `sudo cat /etc/shadow` | auditd — `sudo_exec`, `identity`, `privileged_exec` | Detected |
| `sudo touch /etc/fim-test-file` | syscheck FIM, realtime | Detected |
| `sudo nano /etc/ssh/sshd_config` | auditd — `sshd_config` | Detected |
| SSH with non-existent user | journald — rule 5710 | Detected |

All four surfaced in the dashboard with correct agent attribution.

### Why verification is not optional

Two separate failures during this phase produced **no error of any kind** while silently collecting nothing:

1. An agent enrolled, reported `Active`, and shipped zero events — the configured log path did not exist on the distribution.
2. Alerts were generated and stored correctly but were invisible in the dashboard because the time range excluded them.

Neither would have been discovered by checking agent status. Only generating known events and confirming their arrival distinguishes a working pipeline from one that merely appears healthy.

---

## 6. Detection ideas

Five detections derived from what this environment can currently observe. Rules 5710 and 2502 are stock and already firing; the remainder are the development queue.

### D1 — SSH brute force against a lab host

**Rules:** 5710 → 2502 (correlated)
**MITRE:** T1110.001 Password Guessing
**Data source:** journald
**Status:** Verified

Repeated authentication failures against a single host. The stock correlation chain escalates from individual failures to a brute-force determination. Baseline noise in an isolated lab is effectively zero, which makes this high-fidelity here and considerably noisier in a real environment — worth noting when reasoning about tuning.

### D2 — Privileged command execution by an interactive user

**Rule:** custom, on auditd key `privileged_exec`
**MITRE:** T1548 Abuse Elevation Control Mechanism
**Data source:** auditd
**Status:** Data flowing, rule to be written

Any `execve` running with `euid=0` where `auid>=1000`. Catches privilege escalation and post-compromise activity regardless of the binary used. Requires an allowlist for routine administration, which is precisely the tuning exercise worth documenting.

### D3 — Modification of identity or authorization files

**Rule:** custom, on auditd key `identity`
**MITRE:** T1098 Account Manipulation · T1136 Create Account
**Data source:** auditd
**Status:** Data flowing, rule to be written

Writes to `/etc/passwd`, `/etc/shadow`, `/etc/sudoers`, or `/etc/sudoers.d/`. Legitimate changes are rare and deliberate on a stable host, making unexpected ones inherently suspicious.

### D4 — SSH configuration change

**Rule:** custom, on auditd key `sshd_config`
**MITRE:** T1098.004 SSH Authorized Keys · T1556 Modify Authentication Process
**Data source:** auditd
**Status:** Data flowing, rule to be written

Modification of `sshd_config` or the monitored `.ssh` directories. A common persistence mechanism, and one that FIM and auditd detect from different angles — file content versus the syscall that changed it.

### D5 — Agent stopped reporting

**Rule:** stock 503 / 504 (agent disconnected)
**MITRE:** T1562.001 Impair Defenses — Disable or Modify Tools
**Data source:** manager
**Status:** To be enabled with alerting

Detection of telemetry loss itself. Directly motivated by this phase's experience: an endpoint that stops reporting looks identical to an endpoint where nothing is happening. Given that two silent collection failures occurred here, monitoring the monitoring is not a theoretical concern.

---

## 7. Deliverables

- [x] Wazuh deployed and reachable
- [x] Linux endpoints enrolled
- [x] journald, auditd, and FIM collection configured
- [x] Telemetry verified against generated test events
- [x] First five detection ideas documented
- [ ] Windows endpoint with Sysmon
- [ ] Custom rules written for D2–D4
- [ ] Dashboard screenshots
- [ ] Agent disconnection alerting enabled

---

## 8. Next

1. Build `win-client-01`, deploy Sysmon with a curated configuration, enroll the agent
2. Write custom rules for D2 through D4 and tune out routine administration
3. Enable agent disconnection alerting
4. Capture dashboard evidence
