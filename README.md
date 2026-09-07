# Home Network

Documentation for the network I run at home: a self-built OPNsense
firewall replacing the ISP-provided router, two access points, and a
virtualization host running a separate Active Directory lab.

Built and cut over in 2026. This document covers the design, the
reasoning behind it, and what went wrong along the way.

<!-- Insert diagrams/topology.png here once exported from Visio -->

---

## Why replace the ISP router

The service came with a Calix GigaSpire gateway. It works, but it is
ISP-managed hardware with remote provisioning — the provider can push
configuration to it, and I do not control what it reports upstream.
Replacing it with hardware I own means I control the firewall rules,
the DNS resolution path, and the update cycle.

The tradeoff is real and worth stating: the household now depends on
a device I maintain. An ISP router that breaks is the ISP's problem.
This one is mine.

---

## Hardware

| Device | Role | Why this |
|---|---|---|
| Dell Precision 3440 SFF (i5-10500, 16 GB) | Firewall and router — OPNsense | Certified refurbished at roughly the price of a purpose-built appliance, with far more CPU. Has AES-NI, which matters for VPN throughput and was a hard requirement |
| FENVI dual Intel i226-V 2.5GbE PCIe card | WAN and LAN interfaces | Intel NICs only. Realtek NICs are a known source of throughput and stability problems on BSD-based firewalls |
| Reyee AX6000 | Primary access point, AP mode | Already owned. Demoted from routing to wireless-only |
| Calix GigaSpire | Secondary access point | Repurposed rather than discarded — extends coverage at zero cost |
| Custom workstation (i9-12900K, 16 GB, Zorin OS) | Daily driver and KVM virtualization host | Also hosts the isolated Active Directory lab |
| Brightspeed fiber ONT | Service handoff | Provider equipment |

### Hardware selection criteria

Three non-negotiables when screening candidates:

- **Intel NICs only** (i226 / i225 / i210 / i211)
- **AES-NI present** for cryptographic offload
- **Two or more Ethernet ports**

Several otherwise-attractive options were rejected against these: a
mini PC with a single Ethernet port, a closed appliance with no
OPNsense support, and a low-power box lacking AES-NI with Realtek
NICs.

---

## Network design

**Addressing:** `192.168.1.0/24`, currently flat.

| Host | Address | Notes |
|---|---|---|
| OPNsense LAN | `192.168.1.1` | Gateway, DNS, DHCP |
| Calix (as AP) | `192.168.1.3` | Static; its DHCP server disabled |
| Clients | DHCP pool | Managed by OPNsense |

**WAN:** Brightspeed fiber. Connected on plain DHCP — no VLAN tag or
PPPoE required, which was the best case of three possible
configurations tested in order. Public addressing is deliberately not
documented here.

### Why the network is flat, and what comes next

Everything currently sits on one subnet. This is a known gap rather
than an oversight.

The intended next step is separating untrusted devices — smart TVs,
IoT hardware, anything that phones home — from machines holding
personal data. That work is blocked on confirming whether the Reyee's
secondary SSID supports VLAN tagging in AP mode. Without tagging at
the access point, a separate wireless segment cannot be carried back
to the firewall over the single uplink.

Documenting the blocker rather than the aspiration seemed more useful
than describing a design I have not implemented.

---

## Services

### DNS — the part I spent most effort on

- **Recursive resolution** via Unbound, not forwarding. The firewall
  resolves from the root servers itself, so no third-party resolver
  sees a log of every lookup from this house.
- **DNSSEC validation** enabled.
- **Network-wide blocklists** for ads and trackers, which covers
  devices that cannot run a browser extension — smart TVs, phones,
  consoles.
- **Forced DNS**: a destination NAT rule redirects all port 53
  traffic to the local resolver. Without this, any device with a
  hardcoded DNS server simply bypasses the blocklists and the
  privacy benefit. This is the rule that makes the rest of it
  actually apply.

Verified from a phone on Wi-Fi rather than from the firewall itself:
a known tracker domain returns `0.0.0.0` while ordinary domains
resolve to real addresses. Testing from a client is the only test
that proves the whole path works.

### DHCP

Served by OPNsense for the whole subnet. Static reservations for
always-on devices, which makes writing firewall rules against them
straightforward.

### Firewall posture

Default deny inbound from WAN. Outbound from LAN is permitted
broadly, with specific exceptions:

- The ISP gateway is blocked from outbound connections, so it cannot
  phone home to the provider while still functioning as an access
  point.
- The management interface is bound to LAN only. SSH is disabled.
  HSTS enabled.

Specific rules and source addresses are deliberately omitted — see
the note at the end.

### VPN

A ProtonVPN client runs on the workstation, not on the firewall. This
means VPN protection applies to that one machine's traffic rather
than the whole network. That is the current design and is a
deliberate choice, not an incomplete one — routing all household
traffic through a commercial VPN would break streaming services and
add a dependency the household would notice when it failed.

---

## Performance

Wired throughput measured at roughly 909 Mbps down / 915 Mbps up on a
gigabit service, with a bufferbloat grade of A (+11 ms down, +1 ms up
under load). No traffic shaping needed — FQ-CoDel was evaluated and
found unnecessary given those figures.

Wi-Fi graded C, with a +125 ms upload latency spike under load. Traced
to 2.4 GHz channel overlap between the two access points, which is a
consequence of adding the second AP for coverage. Unresolved; noted as
an open item.

---

## Incident: total loss of connectivity during cutover

**Symptom.** After cutting the network over to the new firewall, the
workstation had no connectivity. Other devices on the network were
fine.

**Hypotheses.** Firewall rule blocking the host; DHCP not issuing a
lease; cabling to the wrong interface; NIC driver problem.

**Evidence.** Other clients on the same segment were working, which
ruled out the firewall and DHCP as general failures. The problem was
specific to one machine. Interface listing on that machine showed an
unexpected interface, `pvpnksintrf1`, still present.

**Root cause.** The ProtonVPN kill switch. It creates a virtual
interface that blocks all traffic outside the tunnel — by design. When
the network changed underneath it, the tunnel failed and the kill
switch did exactly what it exists to do, which presented as a total
network failure on that host.

**Fix.** Disconnected the VPN client, which removed the interface.

**Prevention.** Disconnect VPN clients before network changes. More
generally: when one host fails and its peers do not, the cause is on
that host, not on the network. I spent time checking firewall rules
that could not have been responsible.

---

## What I would do differently

- **Segment first, not later.** Retrofitting VLANs onto a working flat
  network means a second disruptive change. Designing the segments
  before cutover would have cost nothing extra at the time.
- **Reconsider the second AP.** It solved a coverage problem and
  created a 2.4 GHz interference problem. A single better-placed AP,
  or manual channel assignment, may have been the better answer.
- **The ISP gateway is still ISP-managed.** Even blocked outbound and
  demoted to an access point, it remains hardware the provider can
  reach and reconfigure. A cheap dedicated AP would remove that
  entirely.

## Open items

- IoT segmentation, blocked on Reyee VLAN support in AP mode
- 2.4 GHz channel overlap between the two access points
- WireGuard for remote access — planned, not built
- Suricata IDS/IPS — hardware is capable, not yet enabled

---

## What is deliberately not documented here

Public IP addressing, WAN interface details, firewall rules with real
source addresses, credentials, VPN keys, and SSIDs are omitted. The
design reasoning is the useful part; the specifics would only be
useful to someone attacking this network.

All screenshots have been checked for the same before committing.
