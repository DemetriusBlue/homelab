# Wazuh Agents Disconnected After Moving the Labnet Gateway

**Date:** November 12, 2026
**Components:** Proxmox VE (pve, pve02), Proxmox SDN (labnet VXLAN), Wazuh 4.14.6, OPNsense
**Time to fix:** ~30 minutes

## What happened

I logged into the Wazuh dashboard and both of my Proxmox hosts, pve and pve02, showed up as **Disconnected**. Everything else in the lab looked fine.

The timing gave me a hint. A few weeks earlier I'd reworked my lab network: Wazuh (192.168.100.20) lives on a Proxmox SDN VNet called `labnet` (192.168.100.0/24), and I removed the pve node's manual `192.168.100.1` gateway and NAT so OPNsense could take over as the labnet gateway. I didn't connect that change to the agents at the time.

## Troubleshooting

### 1. Is the agent even running?

First thing I checked was the agent itself on pve:

```
root@pve:~# systemctl status wazuh-agent
● wazuh-agent.service - Wazuh agent
     Active: active (running)
```

Running, all the sub-processes (agentd, syscheckd, logcollector, modulesd) up. So the agent wasn't the problem.

### 2. Is it pointed at the right manager?

```
root@pve:~# grep -A3 "<server>" /var/ossec/etc/ossec.conf
    <server>
      <address>192.168.100.20</address>
      <port>1514</port>
      <protocol>tcp</protocol>
```

Correct IP, correct port. Config was fine too.

### 3. Can the host reach the manager?

```
root@pve:~# nc -zv 192.168.100.20 1514
192.168.100.20: inverse host lookup failed: Unknown host
```

That first line threw me for a second, but it's just `nc` failing a reverse DNS lookup, which is normal on a lab network with no DNS. The real tell was that the command **hung** with no response. Reran it with `-n` to skip DNS:

```
root@pve:~# nc -nzv 192.168.100.20 1514
```

Still hung. A hang (instead of "connection refused") means packets are going somewhere and never coming back, so this was a network path problem, not a Wazuh problem.

### 4. Where is the traffic actually going?

```
root@pve:~# ip route get 192.168.100.20
192.168.100.20 via 10.0.0.1 dev vmbr0 src 10.0.0.53
```

There it is. The host was sending Wazuh traffic out `vmbr0` to my **home router (10.0.0.1)**, which has no idea labnet exists, so the packets just got dropped.

## Root cause

When I removed `192.168.100.1` from the pve host so OPNsense could be the labnet gateway, I also removed the host's only connection to 192.168.100.0/24. With no address on labnet and no route to it, the host fell back to its default route through the home router.

The VMs on labnet were fine because they use OPNsense as their gateway. The hosts themselves were the ones cut off, and the Wazuh agents on them went dark.

## Fix

Gave each Proxmox host its own address on labnet, **without a gateway**, so OPNsense stays the router for the VMs.

**Test it first (temporary):**

```
ip addr add 192.168.100.2/24 dev labnet
ip route get 192.168.100.20      # now shows dev labnet
nc -nzv 192.168.100.20 1514      # open
systemctl restart wazuh-agent
```

pve showed as Active in the dashboard within a minute. Did the same on pve02 with `192.168.100.3`.

**Make it permanent:**

The temporary address disappears on reboot, so I added this to the bottom of `/etc/network/interfaces` on each node (`.3` on pve02):

```
iface labnet inet static
    address 192.168.100.2/24
```

Then applied it with `ifreload -a` and verified with `ip a show labnet`.

I put it in `/etc/network/interfaces` instead of the SDN-generated file in `/etc/network/interfaces.d/` because Proxmox overwrites that file every time you apply SDN changes.

## Lessons learned

- **A hang is different from a refusal.** "Connection refused" means you reached the box and nothing's listening. A hang means your packets aren't making it there, so check routing before anything else.
- **`ip route get` is the fastest way to answer "where is this traffic going?"** One command showed the exact problem.
- **Network changes have side effects outside the thing you're changing.** I tested that the VMs could still talk to each other after the gateway change, but not that the hosts could still reach Wazuh. Next time I'll check the SIEM dashboard after any network change.
- **Monitoring the monitor matters.** The hosts were offline in Wazuh for a while before I noticed. Adding an alert for disconnected agents is on the list.

## What's next

This fix puts both hypervisors directly on labnet, the same network as some intentionally vulnerable VMs. That's fine for now, but the cleaner long-term setup is to route host traffic to labnet through OPNsense and only allow ports 1514/1515 to the Wazuh manager. That keeps the hypervisors off the vulnerable network entirely.
