# SOC Analyst Home Lab

## Overview
A self-hosted cybersecurity lab built to practice detection engineering, 
network segmentation, and incident response — hands-on preparation for a 
SOC Analyst role. Rather than just studying for certifications, this project 
focuses on building, breaking, and monitoring real systems.

## Status
🚧 In progress — actively being built. See [Current Progress](#current-progress) below.

## Goals
- Practice building and defending a segmented network
- Generate real attack traffic and validate detection coverage using a SIEM
- Map detections back to the MITRE ATT&CK framework
- Document troubleshooting and findings as I go, treating this like a 
  real-world engineering log

## Architecture
- **Hardware**: 2x HP EliteDesk 705 G5 Desk Mini (Ryzen 5 Pro 3400G, 16GB RAM, 256GB NVMe)
- **Hypervisor**: Proxmox VE (2 standalone nodes)
- **Planned network segmentation**: pfSense/OPNsense separating management, 
  detection, victim, and attacker networks
- **Planned detection stack**: Wazuh SIEM
- **Planned attack range**: Kali Linux, Metasploitable, DVWA, Windows AD lab

<img width="431" height="401" alt="Topology drawio" src="https://github.com/user-attachments/assets/88276f2c-e942-42da-8422-22dbe7f5b2b1" />
