# Making OPNsense the Gateway for My Lab Network (Dead Ends Included)

**Date:** October 28, 2026
**Time spent:** about 2 hours
**Outcome:** Working. OPNsense now routes between my attacker network and my lab network, which sets up Suricata to see attack traffic and ship alerts to Wazuh.

---

## Why I was doing this

The goal is to monitor all the traffic in my lab: run Suricata on OPNsense and send its alerts into Wazuh.

The problem was that Suricata only sees traffic that actually passes **through** OPNsense, and in my setup almost nothing did:

- Wazuh lived on `labnet`, a Proxmox SDN VXLAN network (`192.168.100.0/24`), and my Proxmox node was acting as its gateway.
- OPNsense's LAN was a separate internal bridge (`vmbr1`) that only Kali was on.

So Kali attacking a lab VM would never cross OPNsense, and OPNsense had no path to Wazuh to send logs anyway. The fix was to make OPNsense the gateway for **both** networks.

## The layout

| Network | Proxmox bridge | OPNsense interface | Subnet | What's on it |
|---|---|---|---|---|
| WAN | vmbr0 | WAN (vtnet0) | DHCP from home router (10.0.0.x) | — |
| Attacker | vmbr1 | LAN (vtnet1) | 192.168.10.0/24 | Kali |
| Lab | labnet (VXLAN) | LABNET (vtnet2) | 192.168.100.0/24 | Wazuh, Metasploitable, DVWA |

Hardware: two HP EliteDesk 705 G5 Minis running Proxmox.
- **pve (node 1):** Wazuh
- **pve02 (node 2):** OPNsense and Kali

That split across two nodes ends up mattering a lot later.

---

## Part 1: Just getting into the OPNsense web UI

OPNsense's LAN is on `vmbr1`, an internal-only bridge with no physical port, so my laptop can't reach the GUI. The easy path was to use Kali, which is on the same bridge, and browse to OPNsense from there.

**Dead end #1:** Kali got `192.168.100.17` from DHCP. That's my *labnet* range, not OPNsense's default `192.168.1.1`. At first it looked like Kali was on the wrong network, but its only NIC was on `vmbr1`. It turned out I'd set OPNsense's LAN to `192.168.100.x` during install, the **same subnet** as labnet. That works fine while the two networks never touch, but a router can't route between two networks with the same address range. So I moved OPNsense's LAN to `192.168.10.1/24` through the setup wizard.

**Small gotcha:** after changing the LAN IP, I tried `sudo dhclient -r eth0` on Kali to get a new address and got "command not found." Newer Kali uses NetworkManager instead:

```bash
sudo nmcli device disconnect eth0 && sudo nmcli device connect eth0
```

## Part 2: Taking the gateway away from the Proxmox node

The plan was to clear the gateway from the labnet subnet in Proxmox SDN. But the gateway field was already empty and SNAT was already off. So where was `192.168.100.1` coming from?

```bash
root@pve:~# ip a show labnet
    inet 192.168.100.1/24 scope global labnet
```

It was live on the node, but `grep -rn "192.168.100" /etc/network/` found nothing. It had been added by hand at some point, along with a NAT rule:

```bash
root@pve:~# iptables -t nat -S | grep 192.168.100
-A POSTROUTING -s 192.168.100.0/24 -o vmbr0 -j MASQUERADE
```

These were leftovers from an earlier fix that got Wazuh online. I removed both:

```bash
ip addr del 192.168.100.1/24 dev labnet
iptables -t nat -D POSTROUTING -s 192.168.100.0/24 -o vmbr0 -j MASQUERADE
```

**Lesson:** manual `ip` and `iptables` changes don't show up in config files. If something on the network doesn't match the config, check the live state.

## Part 3: Adding the labnet NIC to OPNsense

I added a third network device (`net2`) to the OPNsense VM on the `labnet` bridge.

**Dead end #2:** OPNsense only showed `vtnet0` and `vtnet1`. A reboot wasn't enough. It took a full **Shutdown → Start** for the VM to pick up the new hardware. After that, `ifconfig -l` in the console showed `vtnet2`.

**Dead end #3:** the web UI still only showed WAN and LAN. That's normal: the GUI only lists *assigned* interfaces. I added `vtnet2` under **Interfaces → Assignments**, then configured it:

- Static IPv4 `192.168.100.1/24`
- MTU `1450` (VXLAN adds overhead)
- No upstream gateway
- Block private networks and Block bogons **unchecked** (it's an internal interface)

Then I added a firewall rule under **Firewall → Rules → LABNET**: pass, any protocol, from LABNET net to any.

## Part 4: Nothing works

From Wazuh:

```bash
ping 192.168.100.1   # nothing
ping 1.1.1.1         # nothing
ping google.com      # nothing
```

Not even the gateway responded. This is where the real troubleshooting started. I worked through it hop by hop with `tcpdump`.

### Hop 1: Is Wazuh's ARP cache stale?

The node used to own `.1`, so Wazuh might still have had the old MAC cached. I flushed it with `sudo ip neigh flush all`. No change.

### Hop 2: Is the VXLAN tunnel working?

This was my main suspect. Before tonight, the gateway lived on the **same node** as Wazuh, so labnet traffic **never had to cross between nodes**. Now it had to travel from node 1 to node 2 to reach OPNsense, for the first time.

The zone's peer list was `10.0.0.53,10.0.0.129`. Both checked out against `ip a show vmbr0` on each node, and the MAC for `10.0.0.129` in node 1's ARP table matched pve02's actual MAC.

Then I checked whether traffic was leaving node 1:

```bash
root@pve:~# tcpdump -ni vmbr0 udp port 4789
IP 10.0.0.53.43115 > 10.0.0.129.4789: VXLAN, flags [I] (0x08), vni 100
IP 192.168.100.20.43820 > 192.73.242.28.443: Flags [S] ...
```

Yes, it was leaving, wrapped in VXLAN and headed to node 2. I also checked that it arrived on node 2:

```bash
root@pve02:~# tcpdump -ni vmbr0 udp port 4789 -c 20
IP 10.0.0.53.54478 > 10.0.0.129.4789: VXLAN, flags [I] (0x08), vni 100
```

It did. (My first capture on node 2 showed nothing, which sent me chasing a DHCP/IP-change theory for a bit. The second run showed packets fine, so I probably just botched the first one.)

Forwarding entries were correct in both directions:

```bash
root@pve:~#   bridge fdb show dev vxlan_labnet | grep 00:00:00:00:00:00
00:00:00:00:00:00 dst 10.0.0.129 self permanent

root@pve02:~# bridge fdb show dev vxlan_labnet | grep 00:00:00:00:00:00
00:00:00:00:00:00 dst 10.0.0.53 self permanent
```

**The tunnel was fine.** My main suspect was wrong.

> Side note: the captures were full of SYNs to random IPs on 443 and pings to dozens of addresses. That's Tailscale on Wazuh probing its relay (DERP) servers to find the fastest path. It's normal, not malware.

### Hop 3: Does traffic reach OPNsense?

On node 2's labnet bridge, I saw only requests and no replies:

```bash
root@pve02:~# tcpdump -eni labnet icmp and host 192.168.100.1 -c 3
bc:24:11:51:c3:ea > bc:24:11:f6:e3:9f ... 192.168.100.20 > 192.168.100.1: ICMP echo request
```

The destination MAC `bc:24:11:f6:e3:9f` belonged to OPNsense's new NIC:

```bash
root@pve02:~# bridge fdb show br labnet | grep -i f6:e3:9f
bc:24:11:f6:e3:9f dev tap200i2 master labnet
```

And the pings were going into OPNsense's tap interface:

```bash
root@pve02:~# tcpdump -eni tap200i2 icmp -c 3
192.168.100.20 > 192.168.100.1: ICMP echo request
```

So Proxmox, the tunnel, and the bridge were all doing their jobs. **The problem was inside OPNsense.**

### Hop 4: What is OPNsense doing with them?

**Dead end #4:** the firewall's Live View log showed **nothing** from `192.168.100.20`, no blocks and no passes. I took that to mean "not the firewall." That was wrong. The OPNsense GUI packet capture also showed nothing, but I had filters on, so that result didn't prove anything.

The console was more honest. Directly on the interface:

```bash
tcpdump -eni vtnet2 icmp -c 5
```

There were echo requests and no replies. OPNsense was receiving the pings and ignoring them. So I checked which rules were actually **loaded**:

```bash
pfctl -sr | grep -i vtnet2
block drop in log on ! vtnet2 inet from 192.168.100.0/24 to any
```

That's just the normal anti-spoofing rule. The `!` means *not* on vtnet2, so it blocks labnet addresses showing up on the wrong interface.

The important part was what was **missing**. There was no `pass` rule on vtnet2. My rule existed in the GUI but never made it into the active ruleset, so Wazuh's pings were falling through to the default deny.

## The fix

I went back to **Firewall → Rules → LABNET**, made sure the rule was enabled, and clicked **Apply changes**. Then I checked again:

```bash
pfctl -sr | grep -i vtnet2
```

The `pass` line showed up, and everything worked:

```bash
# from Wazuh
ping -c 3 192.168.100.1   # ✓
ping -c 3 1.1.1.1         # ✓
ping -c 3 google.com      # ✓

# from Kali
ping -c 3 192.168.100.20  # ✓
```

---

## Root cause

A firewall rule that was **saved but never applied**. Everything else (the leftover manual gateway, the subnet overlap, the new NIC not showing up) was a real issue I had to fix along the way, but the final "nothing pings" problem was one missed Apply.

## What I learned

- **Check the live state, not just the config.** The node had an IP and a NAT rule that existed nowhere in `/etc/network/`. `ip a` and `iptables -S` showed the truth.
- **In OPNsense, Save ≠ Apply.** And the GUI showing a rule doesn't mean it's loaded. `pfctl -sr` shows what pf is actually enforcing.
- **No log entries doesn't mean "not the firewall."** I trusted an empty Live View and lost time on it.
- **Troubleshoot hop by hop.** Running `tcpdump` at each point (VM → bridge → node NIC → tunnel → other node → bridge → tap → firewall NIC) turned "nothing works" into "packets reach vtnet2 and die there." Each capture ruled out a whole layer.
- **Moving a service can expose problems that were always there.** The VXLAN tunnel between nodes had never actually carried labnet traffic before tonight. Luckily it was fine, but it was the obvious thing to suspect.
- **Two networks can't share a subnet once you route between them.** My OPNsense LAN and labnet were both `192.168.100.0/24` and nothing broke, until the plan required them to talk.
- **Shutdown → Start, not Reboot, when adding VM hardware.**

## What's next

1. Enable Suricata (ET Open + abuse.ch rules) on the LAN and LABNET interfaces.
2. Forward Suricata alerts and firewall logs from OPNsense to Wazuh over syslog.
3. Test it end to end: run an nmap scan from Kali at the lab and watch the alerts show up in Wazuh.
4. Tighten the temporary allow-all rule on LABNET.
5. Set DHCP reservations (or static IPs) for both Proxmox nodes so the VXLAN peer list never goes stale.
