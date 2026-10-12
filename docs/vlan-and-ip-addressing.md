# VLAN Design and IP Addressing

## 1. VLAN plan

| VLAN | Name | Department | Network | Default gateway (SVI on `SW-CORE`) |
|------|------|------------|---------|-------------------------------------|
| 10 | ADMIN | Administration | 192.168.10.0/24 | 192.168.10.1 |
| 20 | IT | IT (and shared servers) | 192.168.20.0/24 | 192.168.20.1 |
| 30 | RH | Human Resources | 192.168.30.0/24 | 192.168.30.1 |
| 40 | GUEST | Guest Wi-Fi | 192.168.40.0/24 | 192.168.40.1 |
| 99 | MANAGEMENT | Network device management | 192.168.99.0/24 | 192.168.99.1 |

Why segment with VLANs:

- **Security:** departmental traffic is isolated; unauthorized inter-VLAN access is blocked by ACLs.
- **Performance:** smaller broadcast domains reduce unnecessary traffic.
- **Broadcast control:** each VLAN is its own broadcast domain, which limits the impact of broadcast storms.
- **Administration:** each VLAN can be managed and monitored independently.

## 2. Infrastructure addressing

| Item | Address | Notes |
|------|---------|-------|
| `R1` Gi0/0 | 10.0.0.1/30 | Point-to-point link to the core |
| `SW-CORE` Gi0/1 | 10.0.0.2/30 | Routed port (`no switchport`) |
| `SW-ACC-1` VLAN 99 SVI | 192.168.99.11/24 | Management address, gateway 192.168.99.1 |
| Other access switches | 192.168.99.x/24 | Same pattern in VLAN 99 (see the `.pkt` file) |

## 3. Server addressing (all in VLAN 20)

| Server | IP address | Mask | Gateway | Service |
|--------|------------|------|---------|---------|
| `Server-DHCP` | 192.168.20.10 | 255.255.255.0 | 192.168.20.1 | DHCP for all VLANs |
| `Server-DNS` | 192.168.20.20 | 255.255.255.0 | 192.168.20.1 | Name resolution |
| `Server-WEB` | 192.168.20.30 | 255.255.255.0 | 192.168.20.1 | HTTP intranet portal |

Servers use static addresses; all clients receive their addresses by DHCP.

## 4. DHCP

- A single DHCP server (192.168.20.10) serves every VLAN.
- Each SVI outside VLAN 20 forwards client broadcasts with `ip helper-address 192.168.20.10`.
- The server defines one pool per VLAN (`VLAN10`, `VLAN20`, `VLAN30`, `VLAN40`), with the SVI as default gateway, 192.168.20.20 as DNS server and a maximum of 90 users per pool in the captured configuration.
- The report confirms that clients in each VLAN obtained correct addresses before the ACLs were applied.

## 5. DNS records

| Name | Type | Address |
|------|------|---------|
| `dhcp.corp` | A | 192.168.20.10 |
| `dns.corp` | A | 192.168.20.20 |
| `intranet.corp` | A | 192.168.20.30 |

## 6. Wireless networks mapped to VLANs

| Access point | SSID | Security | Encryption | VLAN |
|--------------|------|----------|------------|------|
| `AP-ADMIN` | `CORP_ADMIN` | WPA2-PSK | AES | 10 |
| `AP-IT` | `CORP_IT` | WPA2-PSK | AES | 20 |
| `AP-RH` | `CORP_RH` | WPA2-PSK | AES | 30 |
| `AP-GUEST` | `GUEST_WIFI` | WPA2-PSK | AES | 40 |

Pre-shared keys are intentionally not published in this repository.

## 7. Quick checks

On a client, `ipconfig` should show an address in the subnet of its VLAN, the `.1` gateway, and 192.168.20.20 as DNS server. On the switches:

```text
show vlan brief
show ip interface brief
show interfaces trunk
```

CLI details: [`../packet-tracer/configs-redacted/vlans.txt`](../packet-tracer/configs-redacted/vlans.txt) and [`inter-vlan-routing.txt`](../packet-tracer/configs-redacted/inter-vlan-routing.txt).
