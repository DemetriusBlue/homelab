# SSH Brute Force vs. Wazuh: Why My Brute-Force Alerts Never Fired

**Dates:** November 12–13, 2026
**Attacker:** Kali Linux (192.168.10.102, LAN behind OPNsense)
**Target:** Metasploitable 3, Ubuntu 14.04 (192.168.100.30, labnet)
**Detection:** Wazuh 4.14.6 agent on the target, Suricata on OPNsense forwarding to Wazuh (192.168.100.20)
**Result:** Brute-force detection was silently broken. Found three root causes, fixed them, and got the high-severity alerts to fire.

## The goal

This was my first attack-and-detect lab. The plan was simple: run recon and an SSH brute force from Kali against Metasploitable, then see what Wazuh and Suricata caught.

Kali sits on a different subnet than Metasploitable, so every attack has to route through OPNsense. That means I get two layers of detection: Suricata watching the network, and the Wazuh agent watching the host's logs.

## Setup notes (and one security catch)

Getting the agent onto Metasploitable took some work. Ubuntu 14.04 couldn't resolve `packages.wazuh.com` (no DNS on labnet, on purpose), so I downloaded the `.deb` on the Proxmox host and copied it over with `scp`. That also keeps the vulnerable box off the internet.

While I was in the VM's hardware settings, I noticed it had a second NIC bridged to **vmbr0, my home network**. An intentionally vulnerable machine one `ifup` away from my real devices. I removed it. Only the labnet NIC stays.

## Baseline test

Before attacking anything, I made sure alerts actually flowed. A few failed SSH logins from Kali produced:

| What I did | What Wazuh flagged |
|---|---|
| `nc -zv` to port 22 | **5706** sshd insecure connection attempt (scan) |
| Login as `fakeuser` | **5710** login attempt with a non-existent user |
| Wrong password for `vagrant` | **5760** / **5503** sshd / PAM auth failed |
| Repeated failures | **2502** user missed the password more than once |

Pipeline confirmed. Snapshot taken.

## Attack 1: Recon

```
sudo nmap -sS -sV 192.168.100.30
```

Open services included Apache 2.4.7 (80), Samba (445), CUPS 1.7 (631), MySQL (3306), and Jetty 8.1.7 (8080). That's a to-do list for future labs.

**What got detected:**
- Suricata: **ET SCAN Possible Nmap User-Agent Observed**
- Wazuh host: **31101** web server 400 error code

Both of these fired because of `-sV`, which sends HTTP requests that literally announce themselves as Nmap. Nothing fired on the SYN scan itself. An attacker who skips version detection or changes the user agent probably slips past both. (I haven't run the plain `-sS` test yet to confirm that.)

## Attack 2: SSH brute force (small wordlist)

```
hydra -l vagrant -P pw.txt ssh://192.168.100.30 -t 4
```

Six passwords, the real one last. Hydra found it.

**What got detected:** a handful of 5760 / 5503 failures, a **5715** successful login, and **5501 / 5502** session opened and closed immediately. That open-then-close pattern is a nice tell: a person logs in and stays, while a tool connects, confirms, and leaves.

**What didn't fire:** **5763** (sshd brute force) and **40112** (multiple failures followed by a success). Suricata had nothing SSH-related either.

My first theory was the threshold. Those rules need roughly 8 failures from the same source in a short window, and five wrong guesses stays under that.

## Attack 3: SSH brute force (bigger wordlist)

Fourteen wrong passwords, then the real one. Hydra cracked it in about 27 seconds.

This time I got lots of 5760s, the 5715 success, and three **2502** alerts at level 10. But still no 5763 or 40112:

```
rule.id:(5763 or 40111 or 40112)   →   0 hits
```

Fourteen failures in under 30 seconds and no brute-force alert. The threshold theory was wrong. Something was broken.

## Root cause #1: rsyslog was hiding the attacker

I expanded one of the 5760 events. There was no `data.srcip` field at all. The raw log explained why:

```
sshd[3348]: message repeated 3 times: [ Failed password for vagrant from 192.168.10.102 port 59340 ssh2]
```

Ubuntu's rsyslog ships with `$RepeatedMsgReduction on`. When identical messages arrive back to back, it writes one "message repeated N times" line instead of N separate ones. That broke detection two ways:

1. **Wazuh couldn't extract the source IP.** The sshd decoder expects a normal `Failed password ...` line. The wrapper doesn't match, so `srcip` never gets pulled out, and rules that correlate "same source IP" can't link the failures together.
2. **The attack looked smaller than it was.** Three failures became one log line.

A default OS logging setting quietly disabled brute-force detection for this host.

**Fix**, in `/etc/rsyslog.conf`:
```
$RepeatedMsgReduction off
```
Then `sudo service rsyslog restart`.

## Then SSH stopped working

When I went to rerun Hydra to verify the fix, it started timing out every time. This is where most of the time went.

### Dead end: a self-inflicted burst

At one point I typed `-t 40` instead of `-t 4`, so Hydra hit sshd with 15 parallel connections. That briefly made SSH unreachable. It was real, but it wasn't the main problem: the timeouts kept happening at `-t 4` long after.

### Dead end: UseDNS

Earlier, an SSH login had hung before the password prompt. Old sshd does a reverse DNS lookup on every client, and there's no DNS on labnet, so I added `UseDNS no` to `sshd_config`. It seemed to work.

During cleanup I ran `grep UseDNS /etc/ssh/sshd_config` and got nothing back. The line had never actually saved. So that "fix" didn't fix anything, and the hang was something else. I added the line properly anyway, but it goes in this writeup as a false lead.

### Root cause #2: the kernel kept panicking

The first time Metasploitable crashed was during the agent install. The trace mentioned the ATA driver, so I blamed the empty IDE CD-ROM drive and removed it.

The next day it crashed again:

```
RIP  [<ffffffff8104ed58>] kvm_unlock_kick+0xa8/0x100
Kernel panic - not syncing: Fatal exception in interrupt
```

`kvm_unlock_kick` was in the first crash trace too. I'd misread it. The real cause is a known bug in the 3.13 kernel's **KVM paravirtual spinlocks**, which shows up when the VM has more than one vCPU. Removing the CD-ROM just changed how long it took to crash.

**Fix:** dropped the VM to **1 core**. With a single CPU there's nothing for the paravirt spinlock code to kick, so the buggy path never runs. A crashing target also explains some of the flaky behavior I'd been chasing.

### Root cause #3: VXLAN MTU

With the kernel stable, plain `ssh` from Kali still hung. `ssh -v` showed exactly where:

```
debug1: SSH2_MSG_KEXINIT sent
```

The connection opened and banners were exchanged, but the server's key-exchange reply never arrived. Small packets worked; the first big one died.

A ping with "don't fragment" set confirmed it:

```
ping -M do -s 1472 -c 3 192.168.100.30
From 192.168.10.1 icmp_seq=1 Frag needed and DF set (mtu = 1450)
```

Labnet is a Proxmox SDN **VXLAN** network. VXLAN wraps every packet in about 50 bytes of extra headers, so the real MTU is 1450, not 1500. Metasploitable's NIC still thought it was 1500 and told Kali it could accept full-size packets. Modern OpenSSH sends a large key-exchange packet, and it got dropped on the way.

To prove it, I forced a smaller handshake:

```
ssh -o KexAlgorithms=curve25519-sha256@libssh.org vagrant@192.168.100.30
```

That connected instantly. Plain `ssh` hung. Same host, same network, only the packet size changed.

**Fix:** set the labnet interface to MTU 1450.

```
sudo ip link set eth2 mtu 1450
```

Making it permanent took one more round. After a reboot the MTU was back to 1500, and `cat /etc/network/interfaces` showed why: there were **two `auto eth2` blocks**, and the one without `mtu` was the one being applied. There was also a leftover `eth0` block for the NIC I'd removed. I rewrote the file cleanly with one eth2 block, `mtu 1450`, and a `post-up ip link set eth2 mtu 1450` line as a backup, then rebooted to confirm it stuck.

## Verification

With all three fixes in, same attack, same 15-password list:

| Time | Rule | Level | Description |
|---|---|---|---|
| 19:37:16 | **5763** | 10 | sshd: brute force trying to get access to the system |
| 19:37:18 | **40112** | 12 | Multiple authentication failures followed by a success |

40112 is the one that matters most. It means an attacker guessed their way in, and it fired two seconds after the brute-force alert. The day before, the same attack produced neither.

I took a new snapshot of the fixed state and deleted the old one, so I can't accidentally roll back to the broken version.

## Smaller things I noticed

- **Clock mismatch.** Kali's timezone was an hour off, so Hydra's timestamps didn't line up with Wazuh. Fixed with `timedatectl set-timezone America/Chicago`. Metasploitable logs in UTC, which Wazuh handles, but correlating events across three different clocks by hand was confusing. Synced time matters.
- **SSDP noise.** Suricata fired "ET DOS Possible SSDP Amplification Scan" about once a minute, unrelated to my attacks. Candidate for a scoped suppression rule.
- **Everything from Suricata is level 3.** In a real queue, the Nmap alert would be buried.
- **The agent registered as `ubuntu`** instead of `metasploitable`, because the install that set the name crashed partway through.

## Lessons learned

- **"No alert" doesn't mean "no attack."** The brute force was happening the whole time. The detection was broken by a logging default nobody would think to check.
- **Check the fields, not just the alerts.** A missing `data.srcip` explained everything. Expanding one raw event was worth more than staring at the alert list.
- **Verify your fixes stuck.** UseDNS never saved. The MTU didn't survive a reboot. Both looked fixed until I checked.
- **Reread the crash trace.** I blamed the wrong component the first time because I latched onto one line instead of the instruction pointer.
- **Overlay networks need smaller MTUs.** Anything running over VXLAN should be set to 1450, or large packets fail in ways that look random.

## What's next

- Set MTU 1450 on the other labnet NICs (Wazuh, OPNsense's labnet interface)
- Run a plain `nmap -sS` (no `-sV`) to confirm whether a SYN scan goes undetected
- Write a custom Wazuh rule that alerts on a success after 3+ failures from the same IP, to catch slower, low-volume guessing
- Raise the level on Suricata ET SCAN alerts
- Suppress the SSDP noise
- Move remote access to Kali behind OPNsense (Tailscale subnet router) instead of putting the attack box on my tailnet
