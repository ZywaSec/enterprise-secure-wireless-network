# Network Architecture

This document describes the physical and logical design of the **Secure Enterprise Wireless Network**, built and validated in Cisco Packet Tracer as an academic project at ENSA Tetouan (Cybersecurity & Embedded Systems, 2025-2026).

> The full technical report (in French) is available in [`../reports/`](../reports/).

## 1. Design goals

- Reproduce a realistic enterprise network with a clear hierarchy (edge, core/distribution, access).
- Isolate four departments (Administration, IT, Human Resources, Guest) on separate VLANs.
- Centralize shared services (DHCP, DNS, HTTP) and make them reachable only where policy allows.
- Provide secure wireless access with one SSID per department.
- Enforce least privilege with extended ACLs and validate the result with systematic tests.

## 2. Topology overview

![Network topology](../assets/network-topology.png)

The network follows a three-tier hierarchical model:

| Tier | Device | Role |
|------|--------|------|
| Edge | Cisco 2911 router (`R1`) | Gateway toward the external side, static routing back to every VLAN |
| Core / Distribution | Cisco 3560-24PS Layer 3 switch (`SW-CORE`) | Inter-VLAN routing (SVIs), trunk aggregation, DHCP relay, ACL enforcement, STP root bridge |
| Access | 4 x Cisco 2960 switches (`SW-ACC-1` to `SW-ACC-4`) | Department switching, VLAN assignment |
| Wireless | 4 access points (`AP-ADMIN`, `AP-IT`, `AP-RH`, `AP-GUEST`) | One SSID and one VLAN per department |
| Servers | `Server-DHCP`, `Server-DNS`, `Server-WEB` | Centralized services in VLAN 20 |

## 3. Device inventory and links

| Link | From | To | Type | Addressing |
|------|------|----|------|------------|
| Uplink | `R1` Gi0/0 | `SW-CORE` Gi0/1 | Routed point-to-point | 10.0.0.0/30 (R1 = .1, SW-CORE = .2) |
| Trunk | `SW-CORE` Fa0/1 | `SW-ACC-1` | 802.1Q trunk | VLANs 10, 20, 30, 40, 99 |
| Trunk | `SW-CORE` Fa0/2 | `SW-ACC-2` | 802.1Q trunk | VLANs 10, 20, 30, 40, 99 |
| Trunk | `SW-CORE` Fa0/3 | `SW-ACC-3` | 802.1Q trunk | VLANs 10, 20, 30, 40, 99 |
| Trunk | `SW-CORE` Fa0/4 | `SW-ACC-4` | 802.1Q trunk | VLANs 10, 20, 30, 40, 99 |
| Servers | `SW-CORE` access ports | DHCP / DNS / WEB servers | Access ports | VLAN 20 |
| Wireless | `SW-ACC-x` Fa0/2 | Department access point | Access port | VLAN 10 / 20 / 30 / 40 |

Detailed addressing is in [`vlan-and-ip-addressing.md`](vlan-and-ip-addressing.md).

## 4. Design rationale

**Dedicated access points instead of Wi-Fi routers.** In enterprise networks, wireless access is delivered by specialized access points, while routing, DHCP, DNS and security are handled by dedicated network devices. Separating these responsibilities makes the design more scalable, more secure and easier to manage. Each access point is wired to its own access switch on a dedicated VLAN, so wireless traffic is isolated from the other departments from the first hop.

**Layer 3 switch for inter-VLAN routing.** SVIs on `SW-CORE` act as default gateways, so traffic between VLANs is routed at the core at switching speed, and ACLs can be applied directly on the VLAN interfaces.

**Centralized services in VLAN 20.** One DHCP server serves all VLANs through `ip helper-address` relays; DNS and the intranet web server are hosted next to it. This keeps management simple and gives a single place to apply policy.

**Rapid PVST+ with `SW-CORE` as root.** The core switch is configured with priority 4096 so it is elected root bridge in every VLAN. Redundant paths stay blocked until needed, which prevents loops and speeds up convergence after a failure.

## 5. Traffic flows

| Flow | Path | Policy |
|------|------|--------|
| Client DHCP request (any VLAN) | Client -> access switch -> `SW-CORE` SVI -> relay -> `Server-DHCP` | Allowed |
| Admin / IT / HR -> intranet (`http://intranet.corp`) | Client -> `SW-CORE` -> `Server-WEB` (VLAN 20) | Allowed (HR via `permit ip any any`) |
| HR -> IT servers | Client -> `SW-CORE` -> VLAN 20 | Allowed |
| HR -> Administration | Client -> `SW-CORE` | **Denied** by `BLOCK_RH_ADMIN` |
| Guest -> any internal VLAN | Client -> `SW-CORE` | **Denied** by `BLOCK_GUEST` |
| Guest -> external networks | Client -> `SW-CORE` -> `R1` | Allowed (not explicitly denied) |

## 6. Scope and limitations

- The project is a **simulation** in Cisco Packet Tracer; behaviour and performance figures come from the simulator, not from physical equipment.
- Wireless security uses WPA2-PSK (see [`security-implementation.md`](security-implementation.md) for the production hardening roadmap).
- CDP reports native VLAN mismatch warnings on some links; they are documented as a follow-up item.
- The Packet Tracer file is in [`../packet-tracer/`](../packet-tracer/); sanitized CLI configurations are in [`../packet-tracer/configs-redacted/`](../packet-tracer/configs-redacted/).
