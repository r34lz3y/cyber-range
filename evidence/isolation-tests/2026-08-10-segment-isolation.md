# Segment Isolation Verification — 2026-08-10

**Objective.** Verify that the lab segment cannot reach the upstream network or the hypervisor management interface, while retaining internet egress required for patching.

**Test host.** `mgmt-jump-01` (10.10.10.10), lab segment, `vmbr1`.

**Method.** Executed from inside the lab segment rather than inferred from configuration.

```bash
echo "=== Should SUCCEED: lab to internet ==="
curl -s -o /dev/null -w '%{http_code}\n' https://deb.debian.org

echo "=== Should SUCCEED: lab to gateway ==="
ping -c2 10.10.10.1

echo "=== Should FAIL: lab to upstream router ==="
ping -c2 -W2 192.0.2.1

echo "=== Should FAIL: lab to hypervisor management ==="
nmap -Pn -p 8006 --host-timeout 10s 192.0.2.10
```

---

## Run 1 — initial state · 03:57 UTC · FAILED

```
=== Should SUCCEED: lab to internet ===
200

=== Should SUCCEED: lab to gateway ===
2 packets transmitted, 2 received, 0% packet loss

=== Should FAIL: lab to upstream router ===
64 bytes from 192.0.2.1: icmp_seq=1 ttl=63 time=1.14 ms
2 packets transmitted, 2 received, 0% packet loss

=== Should FAIL: lab to hypervisor management ===
PORT     STATE SERVICE
8006/tcp open  wpl-analytics
```

| Test | Expected | Actual | Result |
|---|---|---|---|
| Lab → internet | Allow | HTTP 200 | Pass |
| Lab → lab gateway | Allow | 0% loss | Pass |
| Lab → upstream router | Block | 0% loss | **FAIL** |
| Lab → hypervisor mgmt | Block | `open` | **FAIL** |

**Finding.** The lab segment could reach both the upstream network and the hypervisor's management interface. Bridges without physical ports provided layer 2 isolation, but the gateway routes between all segments by design, so layer 3 policy was required and had never been committed.

**Contributing factor.** New rules are appended below existing ones. Evaluation is top-down, first match wins — a block rule created without repositioning would have had no effect.

---

## Remediation

| Interface | Action | Source | Destination | Position |
|---|---|---|---|---|
| LAN | Block | LAN net | 192.0.2.0/24 | Above default allow |
| OPT1 | Block | OPT1 net | 192.0.2.0/24 | Above default allow |
| OPT1 | Block | OPT1 net | LAN net | Above default allow |

---

## Run 2 — after remediation · 04:06 UTC · PASSED

```
=== Should SUCCEED: lab to internet ===
200

=== Should SUCCEED: lab to gateway ===
2 packets transmitted, 2 received, 0% packet loss

=== Should FAIL: lab to upstream router ===
2 packets transmitted, 0 received, 100% packet loss

=== Should FAIL: lab to hypervisor management ===
PORT     STATE    SERVICE
8006/tcp filtered wpl-analytics
```

| Test | Expected | Actual | Result |
|---|---|---|---|
| Lab → internet | Allow | HTTP 200 | Pass |
| Lab → lab gateway | Allow | 0% loss | Pass |
| Lab → upstream router | Block | 100% loss | Pass |
| Lab → hypervisor mgmt | Block | `filtered` | Pass |

---

## Notes

`filtered` rather than `closed` confirms the firewall is dropping silently rather than the destination refusing the connection — the correct behaviour for a default-deny posture, and a meaningful distinction when reading scan output.

Administrative access from the workstation was unaffected. The block rules govern connections *originating* in the lab; inbound management sessions match existing state.

**Re-test required after:** any firewall rule change, gateway upgrade, or addition of a new segment.
