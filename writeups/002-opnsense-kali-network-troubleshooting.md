# Getting OPNsense and Kali Talking to Each Other (Harder Than I Expected)

**Date:** July 2026
**Category:** Networking / Troubleshooting

## Summary
Documenting how I spent way longer than I'd like to admit getting my Kali VM 
to actually reach OPNsense's LAN interface — turned out to be a bridge 
mismatch that IP addresses alone didn't reveal.

## The Setup
Two-node Proxmox homelab. OPNsense running as a VM with two NICs — one for 
WAN (internet-facing, on vmbr0), one for LAN (internal lab network, on 
vmbr1). Kali also sitting on vmbr1, so it should've been able to reach 
OPNsense's LAN interface directly.

It didn't.

## The Problem
Kept getting `Destination host unreachable` pinging OPNsense's LAN IP 
(192.168.1.1) from Kali. Both sides *looked* correctly configured — OPNsense 
showed LAN at 192.168.1.1/24, Kali eventually had an IP in the same subnet — 
but nothing could talk to anything.

## What I Tried (In Order, Including the Dead Ends)
- Checked `ip a` on Kali — no IP at all on eth0 at first, not even loopback 
  weirdness, just nothing
- Ran `dhclient eth0` — command not found. Turns out newer Kali doesn't ship 
  with `isc-dhcp-client` by default, it's all NetworkManager now
- Switched to `nmcli device connect eth0` instead — got "Activation failed"
- Checked `nmcli device status` — eth0 showed disconnected, then later 
  "connecting" and just hung there
- Manually assigned a static IP with `sudo ip addr add 192.168.1.50/24 dev 
  eth0` just to rule out DHCP as the whole problem — still got 
  "Destination host unreachable" even with a valid IP in the right subnet
- At this point I was fairly sure it wasn't a DHCP issue at all — DHCP not 
  working was a symptom, not the actual problem
- Also hit a separate issue along the way where OPNsense kept booting back 
  into the FreeBSD installer instead of the actual installed system — turned 
  out the boot order had the CD/DVD drive prioritized above the disk, so it 
  kept re-launching the installer on every boot. Fixed by reordering boot 
  priority in Proxmox (Options → Boot Order) to put the disk first.

## Root Cause
Even though OPNsense's console showed the LAN interface with the "right" IP 
(192.168.1.1/24), I'd assumed which physical NIC (net0 vs net1 in Proxmox) 
corresponded to LAN vs WAN — and I'd assumed wrong at some point during all 
the bridge swapping I was doing to troubleshoot. The NIC I *thought* was LAN 
was actually sitting on vmbr0, not vmbr1 — meaning it was never actually on 
the same network segment as Kali at all, regardless of what IP address was 
configured on it.

IP addresses don't mean anything if the underlying Layer 2 path (the bridge) 
is wrong. Kali and OPNsense could both claim to be on 192.168.1.0/24 all day 
and still never reach each other if they're not physically on the same 
virtual switch.

## How I Actually Confirmed It
Stopped guessing based on net0/net1 order and instead matched MAC addresses:
- Pulled the LAN interface's MAC from OPNsense's console (under "Assign 
  Interfaces")
- Checked each Network Device in Proxmox's Hardware tab for OPNsense and 
  compared MACs
- Found the actual LAN NIC (confirmed by matching MAC) was set to vmbr0 
  instead of vmbr1

## Resolution
Changed the correct NIC's bridge to vmbr1, did a full Stop/Start on the VM 
(not just a reboot — bridge changes need a clean restart to take effect), 
and confirmed via the console that LAN was still showing the right IP. 
Pinged from Kali again — finally got replies.

## Lessons Learned
- Don't assume net0 = first interface = WAN (or LAN, or whatever) just 
  because of add order. Verify by MAC address against what the OS itself 
  reports, every time.
- A correct IP address is not proof of a correct network path. If two 
  devices "should" be able to reach each other and can't, check the bridge/
  switch layer before spending more time on IP or DHCP config.
- Newer Kali doesn't have `dhclient` by default — it's `nmcli` / 
  NetworkManager territory now.
- Boot order matters more than I expected — leaving an install ISO attached 
  and prioritized can silently send you back into an installer instead of 
  booting your actual system, which looks like a completely different 
  problem if you're not paying attention.
