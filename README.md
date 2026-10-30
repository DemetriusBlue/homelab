# Homelab SOC

A two-node Proxmox lab with an OPNsense gateway, Suricata IDS, and Wazuh SIEM, built to practice what a SOC analyst does: generate attack traffic, detect it, and investigate the alerts end to end.

![Homelab network diagram](network/homelab-network-diagram.svg)

## How it works

All lab traffic is routed through **OPNsense (`fw01`)**. Kali sits on its own attacker network, and the targets and Wazuh sit on a separate lab network, so every attack has to cross the firewall, where **Suricata** inspects it. Suricata alerts and firewall logs are forwarded over syslog to **Wazuh**, which decodes them into searchable alerts.

```
Kali (attacker LAN) ──► OPNsense + Suricata ──► targets (lab network)
                              │
                              └── syslog ──► Wazuh SIEM ──► alerts & investigation
```

Full details are in [network/network.md](network/network.md): address plan, host inventory, firewall policy, and design decisions.

## Stack

| Layer | Tools |
|---|---|
| Virtualization | Proxmox VE (2 nodes), SDN VXLAN |
| Firewall / routing | OPNsense |
| Network detection | Suricata (ET Open rules) |
| SIEM / HIDS | Wazuh (manager, indexer, dashboard) |
| Attacker | Kali Linux |
| Remote access | Tailscale |
| Hardware | 2× HP EliteDesk 705 G5 Mini (Ryzen 5 Pro 3400G, 16 GB RAM, 256 GB NVMe) |

## Skills demonstrated

- **Network segmentation:** separate attacker and lab networks routed through a single inspection point
- **IDS deployment:** Suricata on OPNsense, interface selection, ruleset management, and validating detection with a custom test trigger
- **Log pipeline engineering:** syslog forwarding from OPNsense to Wazuh, plus a custom Wazuh decoder for Suricata's JSON alerts
- **Detection validation:** using `wazuh-logtest` to trace exactly where an event failed to decode or match
- **Packet-level troubleshooting:** hop-by-hop `tcpdump` across VMs, Linux bridges, a VXLAN tunnel, and the firewall
- **Firewall operations:** pf rule verification with `pfctl`, outbound NAT, and anti-spoofing behavior
- **Linux administration:** systemd-networkd, Proxmox SDN, and service config debugging

## Writeups

Each writeup covers what I set out to do, what broke, how I found it, and what I learned, dead ends included.

| Writeup | Summary |
|---|---|
| [Making OPNsense the Gateway for My Lab Network](writeups/opnsense-labnet-gateway.md) | Moving the lab network's gateway from a Proxmox node to OPNsense, then tracing a "nothing pings" problem hop by hop across a VXLAN tunnel to a firewall rule that was saved but never applied. |
| [Getting Suricata Alerts from OPNsense into Wazuh](writeups/suricata-wazuh-pipeline.md) | Building the IDS-to-SIEM pipeline and fixing eight separate breaks, ending with a custom Wazuh decoder for Suricata JSON behind a syslog header. |

## Repository layout

```
homelab-soc/
├── network/            # Network design, diagram, address plan
├── writeups/           # Build and troubleshooting writeups
├── detection-rules/    # Custom Wazuh decoders and rules
└── automation/         # Scripts and tools for the lab
```

## Roadmap

- [x] Two-node Proxmox cluster with SDN VXLAN lab network
- [x] OPNsense as gateway for attacker and lab networks
- [x] Suricata IDS on the inside interfaces
- [x] Suricata and firewall logs flowing into Wazuh
- [x] Tune noisy alerts (starting with Tailscale STUN traffic)
- [ ] Deploy Metasploitable and DVWA as targets
- [ ] Run attack scenarios (nmap, Metasploit, web attacks) and write up the investigations
- [ ] Replace the temporary allow-all lab firewall rule with specific rules
- [ ] Python port scanner integrated with the lab
- [ ] Active Directory lab for Windows detection practice

## About me

I'm Demetrius Blue, a Computer Information Systems student at Saint Xavier University (graduating May 2027) working toward a SOC Analyst role. CompTIA Security+ certified.

[LinkedIn](https://linkedin.com/in/demetriusblue) · [GitHub](https://github.com/DemetriusBlue)
