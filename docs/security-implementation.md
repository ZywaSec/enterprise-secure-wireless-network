# Security Implementation

This document explains how security is built into the network, what each control does, and which gaps remain. The principle applied throughout is **least privilege**: each department gets only the access it needs.

## 1. Security layers

| Layer | Control | Where | Purpose |
|-------|---------|-------|---------|
| Segmentation | VLANs 10, 20, 30, 40, 99 | All switches | Isolate departments at Layer 2 |
| Access control | Named extended ACLs | `SW-CORE` SVIs (inbound) | Restrict inter-VLAN traffic at Layer 3 |
| Wireless | WPA2-PSK with AES, one SSID per department | Access points | Authenticate and encrypt wireless clients |
| Device access | `enable secret`, `service password-encryption` | All switches and `R1` | Protect privileged mode and stored passwords |
| Management | Dedicated VLAN 99 | Switch SVIs | Keep device management on its own network |
| Loop prevention | Rapid PVST+, `SW-CORE` root (priority 4096) | All switches | Prevent switching loops and broadcast storms |

Secrets are not published: see [`../packet-tracer/configs-redacted/`](../packet-tracer/configs-redacted/).

## 2. Access control lists

Both ACLs are **named extended ACLs applied inbound** on the SVI of the source VLAN.

### `BLOCK_GUEST` (VLAN 40)

Goal: guest users cannot reach any internal network and can only use external resources.

| Seq | Action | Source | Destination | Meaning |
|-----|--------|--------|-------------|---------|
| 10 | deny ip | 192.168.40.0/24 | 192.168.10.0/24 | Guest -> Administration |
| 20 | deny ip | 192.168.40.0/24 | 192.168.20.0/24 | Guest -> IT and servers |
| 30 | deny ip | 192.168.40.0/24 | 192.168.30.0/24 | Guest -> HR |
| 40 | deny ip | 192.168.40.0/24 | 192.168.99.0/24 | Guest -> Management |
| 50 | permit ip | any | any | Everything else (external side) |

Applied with `interface vlan 40` / `ip access-group BLOCK_GUEST in`.

### `BLOCK_RH_ADMIN` (VLAN 30)

Goal: HR cannot reach the Administration network, but keeps access to other authorized resources such as the IT servers.

| Seq | Action | Source | Destination | Meaning |
|-----|--------|--------|-------------|---------|
| 10 | deny ip | 192.168.30.0/24 | 192.168.10.0/24 | HR -> Administration |
| 20 | permit ip | any | any | Everything else |

Applied with `interface vlan 30` / `ip access-group BLOCK_RH_ADMIN in`.

Full commands: [`../packet-tracer/configs-redacted/acl.txt`](../packet-tracer/configs-redacted/acl.txt).

## 3. Policy matrix

Reading guide: the row is the VLAN that **starts** the traffic. "Blocked (reply denied)" means the request reaches the destination but the destination's reply is dropped by an ACL, so the session cannot complete (ACLs are stateless).

| From \ To | Administration | IT / servers | HR | Guest | Management | External |
|-----------|----------------|--------------|----|-------|------------|----------|
| **Administration** | - | Allowed | Blocked (reply denied) | Blocked (reply denied) | Allowed | Allowed |
| **IT** | Allowed | - | Allowed | Blocked (reply denied) | Allowed | Allowed |
| **HR** | **Denied** | Allowed | - | Blocked (reply denied) | Allowed | Allowed |
| **Guest** | **Denied** | **Denied** | **Denied** | - | **Denied** | Allowed |

This matrix is derived from the ACL logic. The cases verified by tests are listed in [`test-plan-and-results.md`](test-plan-and-results.md).

## 4. Why ACLs are stateless here

A standard or extended ACL does not track connections. When `BLOCK_RH_ADMIN` denies `192.168.30.0/24 -> 192.168.10.0/24`, it also drops the **replies** HR hosts send back to Administration hosts. The lab shows this effect: a ping started from an Administration PC toward an HR PC times out after the ACL is applied. If session-aware filtering is needed, use reflexive ACLs, `established` matching for TCP, or a stateful firewall.

## 5. Wireless security

- One SSID per department, each mapped to its own VLAN through the access port of its access point.
- WPA2-PSK with AES encryption on all four SSIDs.
- The guest SSID lands in VLAN 40, where `BLOCK_GUEST` removes access to every internal network.
- Pre-shared keys are not published. For real use, choose long, unique passphrases and rotate them.

Details: [`../packet-tracer/configs-redacted/wireless-security.txt`](../packet-tracer/configs-redacted/wireless-security.txt).

## 6. Known gaps and observations

| Topic | Observation | Suggested action |
|-------|-------------|------------------|
| Management VLAN | Only Guest is denied to VLAN 99; Administration, IT and HR are not restricted by an ACL. | Add an ACL limiting access to 192.168.99.0/24 to administrators. |
| Native VLAN mismatch | CDP logs `NATIVE_VLAN_MISMATCH` between `SW-CORE` and access switches. | Set the same native VLAN on both ends of each trunk and verify which physical ports are cabled together. |
| Wireless authentication | WPA2-PSK uses one shared key per SSID. | Move to WPA2-Enterprise (802.1X) with a RADIUS server. |
| Layer 2 attacks | DHCP Snooping and Dynamic ARP Inspection are not enabled. | Enable both on the access switches (trusted port = uplink). |
| Guest name resolution | The guest policy denies VLAN 20, which hosts the internal DNS server. | Provide a public or dedicated DNS resolver for guests. |
| Device credentials | Lab credentials are simple and password-based. | Use strong secrets and SSH-only remote access in production. |
| Simulation scope | Results come from Cisco Packet Tracer. | Re-validate on real equipment before deployment. |

## 7. Production hardening roadmap

1. Resolve the native VLAN mismatch warnings.
2. Enable DHCP Snooping and Dynamic ARP Inspection on the access switches.
3. Add a wireless LAN controller (WLC) with a RADIUS server for WPA2-Enterprise and seamless roaming.
4. Restrict the management VLAN and enable SSH-only administration.
5. Add port security and BPDU protection on access ports.
6. Introduce a stateful firewall at the edge for inspection and logging.
