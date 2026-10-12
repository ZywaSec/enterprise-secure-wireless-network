# Test Plan and Results

The validation follows a three-step method: **baseline connectivity first**, **service availability second**, **security enforcement last**. Each step is checked before the next one, which isolates the cause of any failure.

## 1. Test environment

| Item | Value |
|------|-------|
| Tool | Cisco Packet Tracer (Realtime and Simulation modes) |
| Client tools | PC Command Prompt (`ping`, `ipconfig`), Web Browser |
| Evidence | [`connectivity-tests.png`](../assets/connectivity-tests.png) and the figures in the report |

## 2. Phase 1: baseline connectivity (before ACLs)

Goal: prove that routing and DHCP work before any restriction exists.

| ID | Test | Source | Destination | Expected | Observed | Result |
|----|------|--------|-------------|----------|----------|--------|
| T01 | DHCP lease in every VLAN | Clients in VLAN 10/20/30/40 | `Server-DHCP` via relay | Valid address in the VLAN subnet | Addresses assigned correctly | PASS |
| T02 | Inter-VLAN ping Admin -> HR | Admin PC (`PC0`) | 192.168.30.10 | 0% loss | 4/4 replies, avg 37 ms, max 62 ms | PASS |
| T03 | Inter-VLAN ping Admin -> Guest | Admin PC (`PC0`) | 192.168.40.10 | 0% loss | 4/4 replies, avg 22 ms, max 34 ms | PASS |
| T04 | Inter-VLAN ping Admin -> IT server | Admin PC (`PC0`) | 192.168.20.10 | 0% loss | 4/4 replies, avg 24 ms, max 41 ms | PASS |

Round-trip times stayed below 65 ms in all baseline tests.

## 3. Phase 2: service availability

| ID | Test | Source | Destination | Expected | Observed | Result |
|----|------|--------|-------------|----------|----------|--------|
| T05 | Name resolution | Internal client | `intranet.corp` | Resolves to 192.168.20.30 | Portal reachable by name | PASS |
| T06 | Intranet portal | Admin, IT, HR clients | `http://intranet.corp` | Page loads | "Secure Enterprise Wireless Network" page displayed | PASS |

## 4. Phase 3: security enforcement (after ACLs)

| ID | Test | Source | Destination | Expected | Observed | Result |
|----|------|--------|-------------|----------|----------|--------|
| T07 | Guest -> internal servers | Guest VLAN 40 | Internal servers | Blocked | 100% packet loss (request timed out) | PASS |
| T08 | Guest -> internal hosts (Simulation mode) | `PC3` (Guest) | `PC0`, `PC1`, `PC2` | Blocked | ICMP PDUs reported **Failed** | PASS |
| T09 | HR -> Administration | HR VLAN 30 | 192.168.10.x | Blocked | 100% packet loss | PASS |
| T10 | HR -> IT servers | HR VLAN 30 | 192.168.20.x | Allowed | 0% packet loss | PASS |
| T11 | Console check: restricted vs permitted path | `PC0` | 192.168.30.10 and 192.168.20.12 | Restricted path fails, IT path works | 100% loss to 192.168.30.10; 0% loss to 192.168.20.12 (avg 29 ms) | PASS |
| T12 | Guest -> intranet portal | Guest VLAN 40 | `http://intranet.corp` | Blocked | HTTP connection failed | PASS |

### Reading T11

The Administration PC pings an HR host. The request reaches HR, but `BLOCK_RH_ADMIN` denies the HR reply on its way to the Administration network, so the ping times out. This is the stateless behaviour explained in [`security-implementation.md`](security-implementation.md#4-why-acls-are-stateless-here). The ping to the IT host (192.168.20.12) is unaffected, which confirms the ACL only removes the targeted traffic.

## 5. Summary

| Phase | Tests | Passed |
|-------|-------|--------|
| Baseline connectivity | 4 | 4 |
| Service availability | 2 | 2 |
| Security enforcement | 6 | 6 |
| **Total** | **12** | **12** |

## 6. How to reproduce

1. Open [`../packet-tracer/enterprise-network.pkt`](../packet-tracer/) in Cisco Packet Tracer.
2. In each client, open **Desktop > Command Prompt** and run `ipconfig` to confirm the DHCP lease.
3. Run the pings of phases 1 and 4 from the source hosts listed above.
4. Open **Desktop > Web Browser**, enter `http://intranet.corp` from an internal client and from the guest client.
5. For T08, switch to **Simulation mode**, add ICMP PDUs from `PC3` to `PC0`, `PC1` and `PC2`, then press **Capture / Forward** and check the **Last Status** column.
6. On `SW-CORE`, run `show access-lists` and compare with [`../packet-tracer/configs-redacted/acl.txt`](../packet-tracer/configs-redacted/acl.txt).

To compare against the baseline, remove the `ip access-group` lines, re-run the pings, then restore them.

## 7. Limits of the validation

- All results come from a simulator, with simulated latency.
- The tests use ICMP and HTTP; other protocols are not covered.
- Figures for T02 to T04, T08 and T11 come from the report screenshots; the other results come from the report text.
