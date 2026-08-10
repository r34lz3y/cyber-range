# Troubleshooting Log

Every issue encountered building this environment, with the reasoning that led to each fix. Recorded because diagnosis is the transferable skill — a list of commands that worked demonstrates nothing.

I am learning this field, not practicing it professionally, and I use Claude (Anthropic's AI assistant) to help reason through these problems. Several entries below describe incorrect assumptions — mine and the assistant's — that cost time before the real cause surfaced. Those are left in deliberately. See [About this project](../README.md#about-this-project--read-this-first).

Addresses shown use the [RFC 5737](https://datatracker.ietf.org/doc/html/rfc5737) documentation range in place of real upstream values.

---

## 1 — Host unreachable after installation

**Symptom.** Fresh Proxmox install, connected to the network, no monitor attached. Port scans for the web UI on 8006 returned nothing across the assumed subnet.

**Investigation.** Scanning for the specific service port was the right instinct — 8006 uniquely identifies a Proxmox host. The scans themselves were sound; the target range was wrong. The upstream network was a `/22`, and initial scans covered only part of it.

**Root cause.** Incorrect assumption about the upstream subnet mask.

**Resolution.** Read the actual address from the console. Confirmed with `ifconfig`, which reported netmask `0xfffffc00` — a `/22`, not the `/24` assumed.

**Takeaway.** Verify the network you are scanning before concluding a host is absent. A negative result only means "not in the range I searched."

---

## 2 — Package signatures rejected as not yet valid

**Symptom.**

```
Sub-process /usr/bin/sqv returned an error code (1), error message is:
Verifying signature: Not live until 2026-08-09T20:52:39Z
Error: The repository ... is not signed.
```

**Investigation.** The signatures were not invalid — they were dated *ahead* of the host's clock. That inverts the usual reading of a signature failure: the repository was fine, the verifier's sense of time was not.

**Root cause.** A cascade of three failures.

1. The host was installed with no network, so the installer wrote its fallback nameserver into `/etc/resolv.conf`.
2. With no working DNS, `chrony` could never resolve an NTP pool server, so the clock drifted.
3. A clock behind real time makes validly issued signatures appear to be from the future.

**Resolution.**

```bash
cat > /etc/resolv.conf <<'EOF'
nameserver 192.0.2.1
nameserver 1.1.1.1
EOF

timedatectl set-ntp true
systemctl restart chrony
```

**Takeaway.** The error message named none of the actual causes. When a failure makes no sense on its own terms, look one layer down — an apt signature problem was in fact a DNS problem two steps removed.

---

## 3 — VM boot loop: "No bootable device"

**Symptom.** A newly created VM with an installer ISO attached looped through SeaBIOS reporting `Boot failed: not a bootable disk`.

**Investigation.** The message indicated the CD-ROM was not in the boot order at all, rather than a bad ISO. Inspecting `qm config` confirmed `boot: order=scsi0` — the CD-ROM was attached but never made the boot list.

**Root cause.** Shell parsing:

```bash
qm create 100 ... --boot order=scsi0;ide2 --onboot 1
```

Bash split the line at the semicolon. `qm create` received only `order=scsi0`; the remainder ran as a separate command and failed silently in a long scrollback.

**Resolution.**

```bash
qm set 100 --boot 'order=ide2;scsi0'
```

**Takeaway.** Shell metacharacters inside tool arguments must be quoted. A command that partially succeeds is more dangerous than one that fails outright — the VM was created, so nothing looked broken.

---

## 4 — Correct password rejected after installation

**Symptom.** The root password set during installation was refused at the console.

**Root cause.** The boot order still preferred the ISO, so the live installer environment was booting rather than the installed system. The live image uses its own fixed credentials; the password from the install had never been in play.

**Resolution.** Set boot order to disk and detach the ISO.

**Takeaway.** "Wrong password" was accurate — it was simply a different system than assumed. Confirm which OS instance is actually running before troubleshooting credentials.

---

## 5 — Locked out by the firewall's own brute-force protection

**Symptom.** The management web UI became unreachable from one specific workstation. Rules were verified correct. `tcpdump` showed SYN packets arriving at the firewall with no response.

**Investigation.** `pfctl -vvsr` prints per-rule counters:

```
059 block drop in log quick proto tcp from <sshlockout:12> to (self) port = https
  [ Evaluations: 248   Packets: 248 ]

084 pass in quick on vtnet0 ... port = https
  [ Evaluations: 903   Packets: 0 ]
```

The pass rule was being evaluated 903 times and passing nothing. A rule with evaluations but zero packets is being shadowed by an earlier match.

**Root cause.** Earlier failed logins added the workstation to the `sshlockout` table. That block rule carries `quick`, which terminates evaluation on match — so the pass rule below it was never reached.

**Resolution.**

```bash
pfctl -t sshlockout -T flush
```

**Takeaway.** Rule counters answer "which rule is actually matching" directly. Reading a ruleset top to bottom and reasoning about it is slower and less reliable. Also worth noting: the lockout was the firewall working correctly, and it is invisible from the web interface.

---

## 6 — Traffic passed the firewall but no response ever returned

**Symptom.** After clearing the lockout, connections still timed out. `tcpdump` confirmed SYN packets arriving. `tcpdump -ni pflog0` logged **no block** for that traffic. The service was confirmed listening via `sockstat`.

**Investigation.** Three facts that together excluded most explanations:

| Observation | Excludes |
|---|---|
| SYN packets arrive at the interface | Routing, bridging, wrong address |
| `pflog0` records no block | Firewall rules |
| `sockstat` shows the service bound to `*:443` | Service down |

A timeout rather than a reset means no response was generated *and* no rejection was sent. That points at the return path.

**Root cause.** The pass rule read:

```
pass in quick on vtnet0 reply-to (vtnet0 192.0.2.1) inet proto tcp from ...
```

OPNsense attaches `reply-to <gateway>` to WAN rules automatically, on the assumption that WAN faces a distinct upstream network. Here the administrative workstation shared a subnet with the WAN interface, so responses were handed to the upstream router instead of being delivered directly to a host on the same segment — and were discarded as asymmetric.

**Resolution.** Firewall → Settings → Advanced → **Disable reply-to**.

**Takeaway.** Distinguish "dropped" from "never answered." A silent timeout with no logged block is a return-path problem, and no amount of inbound rule inspection will find it. This is also a direct consequence of managing a firewall from its own WAN subnet — an unconventional topology the software does not expect.

---

## 7 — `pfctl -e` restoring a stale ruleset

**Symptom.** After using `pfctl -d` to regain access and correcting the configuration, `pfctl -e` locked the session out again.

**Root cause.** `pfctl -e` re-enables the ruleset that was loaded at the moment of disabling. Configuration changes made in the interim are not compiled in.

**Resolution.**

```bash
configctl filter reload
```

**Takeaway.** Use the platform's own configuration path rather than the underlying tool. `pfctl` operates on the running state; `configctl` rebuilds from saved configuration.

---

## 8 — Segment isolation not actually in place

**Symptom.** None. Everything appeared correct.

**Investigation.** Isolation was tested from a host inside the lab segment rather than assumed:

```
=== Should FAIL: lab to upstream router ===
2 packets transmitted, 2 received, 0% packet loss     <-- reachable

=== Should FAIL: lab to hypervisor management ===
8006/tcp open                                          <-- reachable
```

**Root cause.** Block rules existed in the design document but had never been committed to the firewall. Layer 2 isolation — bridges with no physical port — was in place, but insufficient on its own, because the gateway routes between all three segments by design. Additionally, OPNsense appends new rules *below* the existing allow rule, where they would have had no effect even once created.

**Resolution.** Created block rules on the lab and DMZ interfaces and positioned them above the permissive egress rules. Re-tested:

```
=== Should FAIL: lab to upstream router ===
2 packets transmitted, 0 received, 100% packet loss

=== Should FAIL: lab to hypervisor management ===
8006/tcp filtered
```

**Takeaway.** A control that has not been tested is an assumption. This one was documented, diagrammed, and absent. `filtered` rather than `closed` also matters — it confirms silent dropping rather than active refusal, which is the correct behaviour for default-deny.

---

## 9 — Agent enrolled and healthy, collecting nothing

**Symptom.** The SIEM agent registered, reported `Active`, and produced no events. No errors on either side.

**Investigation.**

```bash
grep sshd /var/log/auth.log
# grep: /var/log/auth.log: No such file or directory
```

**Root cause.** Debian 13 minimal images ship journald-only. `rsyslog` is not installed, so `/var/log/auth.log` never exists. The agent's default file-based collection referenced a path that was never created — and reported no error, because a missing file is not a failure condition for a log reader that expects rotation.

**Resolution.** Collect from journald directly:

```xml
<localfile>
  <location>journald</location>
  <log_format>journald</log_format>
</localfile>
```

The stock sshd decoder parses journald output without modification.

**Takeaway.** The most dangerous monitoring failure is the silent one. An agent reporting healthy while collecting nothing is worse than an agent that is visibly down, because dashboards look normal. Only verification against deliberately generated test events reveals it.

---

## 10 — Alerts generated but not visible in the dashboard

**Symptom.** Detections confirmed present in `alerts.json` on the manager, absent from the dashboard.

**Investigation.**

```bash
grep -c '5710' /var/ossec/logs/alerts/alerts.json
# 34
```

Detection was working. The gap was between storage and presentation.

**Root cause.** The dashboard's time range was set to a window that did not include the events.

**Takeaway.** Absence of data in a view is not absence of detection. Verify against the source of truth — the manager's alert log — before troubleshooting rules, decoders, or agents. This is a common and expensive false alarm in SIEM work, and the habit of checking the raw log first saves hours.

---

## Diagnostic techniques worth reusing

| Technique | Answers |
|---|---|
| `tcpdump -ni <iface> port <n>` | Do packets arrive at all? Separates upstream problems from local ones |
| `tcpdump -ni pflog0` | Which rule blocked this packet? pf logs the rule number directly |
| `pfctl -vvsr` | Per-rule counters. Evaluations without packets means the rule is shadowed |
| Timeout vs. connection reset | Silent drop versus active refusal — different root causes entirely |
| Checking `alerts.json` before the dashboard | Distinguishes detection failure from presentation failure |
| Generating known-good test events | The only way to prove a collection path works end to end |
