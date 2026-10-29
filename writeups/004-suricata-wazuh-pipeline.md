# Getting Suricata Alerts from OPNsense into Wazuh (Dead Ends Included)

**Dates:** October 28–29, 2026
**Time spent:** about 5 hours over two sessions
**Outcome:** Working. Suricata on OPNsense detects traffic crossing my lab networks, ships alerts over syslog, and Wazuh decodes them into searchable alerts (rule 86601).

---

## Where this picks up

In my [last writeup](opnsense-labnet-gateway.md), I made OPNsense the gateway for both of my lab networks:

| Network | OPNsense interface | Subnet | What's on it |
|---|---|---|---|
| WAN | vtnet0 | 10.0.0.x (home router) | — |
| LAN | vtnet1 | 192.168.10.0/24 | Kali |
| LABNET | vtnet2 | 192.168.100.0/24 | Wazuh, Metasploitable, DVWA |

Now anything Kali throws at the lab crosses OPNsense. The goal this time was:

**Traffic → Suricata on OPNsense detects it → syslog → Wazuh → alert in the dashboard**

Every arrow in that chain broke at least once.

---

## Part 1: Setting up Suricata

Under **Services → Intrusion Detection → Administration**:

- **Download tab:** enabled all the `ET open/emerging-*` rulesets and downloaded them. ThreatFox and URLhaus (abuse.ch) failed to download. abuse.ch now requires a free account key for some feeds, which is the likely cause. I skipped them since ET Open covers what I need for the lab.
- **Settings:**
  - Capture mode: **PCAP live mode** (IDS, detect only)
  - Promiscuous mode on
  - Home networks: `192.168.10.0/24, 192.168.100.0/24`
- **Policy** (a separate menu item, not a tab): alert on all ET Open rulesets

**Dead end #1:** clicking Apply gave me **"Error reconfiguring IDS."** I had picked **Hyperscan** as the pattern matcher. Hyperscan needs CPU instructions that a Proxmox VM doesn't always pass through to the guest. Switching to **Default** fixed it immediately. To get Hyperscan back later, I'd set the VM's CPU type to `host`.

## Part 2: Suricata runs but catches nothing

My test was the classic one:

```bash
curl http://testmynids.org/uid/index.html
```

The page returns `uid=0(root) gid=0(root) groups=0(root)`, which trips signature 2100498, *GPL ATTACK_RESPONSE id check returned root*. Or at least it's supposed to.

No alerts. I checked things one at a time:

- `pgrep -l suricata` → running
- Rule 2100498 → enabled
- Turned off hardware checksum offload, TSO, and LRO under **Interfaces → Settings** (common advice for IDS in VMs) and rebooted → still nothing

Then I checked whether Suricata was seeing traffic at all:

```bash
grep "decoder.pkts" /var/log/suricata/stats.log | tail -3
```

The counter was climbing, so it *was* capturing packets. Then the main log gave it away:

```
<Notice> -- vtnet0: packets: 2385, drops: 0 (0.00%), invalid chksum: 0
```

**vtnet0 is WAN.** Suricata was only watching WAN, not LAN or LABNET, so my lab traffic was going right past it. I fixed the Interfaces setting to **LAN + LABNET** and the log showed the change:

```
Threads created -> W: 2 FM: 1 FR: 1   Engine started.
```

That's two worker threads, one per interface. Alerts started appearing in OPNsense.

**Lesson:** "running" and "watching the right thing" are different. The startup log tells you which interfaces Suricata is actually attached to.

## Part 3: Shipping logs to Wazuh

**OPNsense side:** under **System → Settings → Logging → Remote**, I added a target: UDP, port 514, host `192.168.100.20` (Wazuh), applications `suricata` and `filter`.

**Wazuh side:** I added a syslog listener to `/var/ossec/etc/ossec.conf`:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>192.168.100.1</allowed-ips>
</remote>
```

### Dead end #2: nothing arriving

`tcpdump -ni any udp port 514` on Wazuh showed **0 packets**. A capture on OPNsense's LABNET interface showed nothing leaving either. After re-applying the remote logging target and restarting `syslog-ng` (**System → Diagnostics → Services**), packets started flowing:

```
IP 192.168.100.1.44096 > 192.168.100.20.514: SYSLOG local0.info
```

### Dead end #3: arriving, but nobody listening

Packets were reaching Wazuh, but:

```bash
sudo ss -ulnp | grep 514
# (nothing)
```

Nothing was bound to port 514. The config block was there, so I restarted the manager:

```
Job for wazuh-manager.service failed because the control process exited with error code.
```

The log explained why:

```
wazuh-csyslogd: ERROR: (1226): Error reading XML file 'etc/ossec.conf':  (line 0).
```

`grep -n -B6 -A6` around my new block showed the problem:

```
46-  </remote>
47-
48-  </remote>
```

There was an **extra `</remote>`** on line 48, left over from my edit. One stray closing tag took down the whole manager. After deleting it, Wazuh came back up:

```
UNCONN 0  0  0.0.0.0:514  0.0.0.0:*  users:(("wazuh-remoted",pid=4091,fd=4))
```

**Lesson:** after editing `ossec.conf`, restart right away and check `systemctl status`. A broken config means *no* alerts at all, not just missing new ones.

## Part 4: Side quest — Wazuh lost the internet

In the middle of all this, Wazuh couldn't ping `1.1.1.1` anymore, even though Kali could. The OPNsense side checked out: the LABNET pass rule was loaded (`pfctl -sr`) and outbound NAT covered `(vtnet2:network)` (`pfctl -s nat`). The real answer was on Wazuh itself:

```bash
$ ip route get 1.1.1.1
RTNETLINK answers: Network is unreachable
```

**Wazuh had no default route.** It could reach its own subnet, but nothing beyond it. The route had been added by hand at some point and never saved. I added it back temporarily to confirm:

```bash
sudo ip route add default via 192.168.100.1
```

Then I made it permanent in `/etc/systemd/network/20-eth0.network` (the Wazuh appliance uses systemd-networkd, not NetworkManager):

```ini
[Network]
Address=192.168.100.20/24
Gateway=192.168.100.1
DNS=192.168.100.1
```

Setting `DNS=` also fixed the `Could not resolve hostname` errors from Wazuh's vulnerability feed updater that had been filling `ossec.log`.

## Part 5: Next day — still nothing in the dashboard

Everything looked connected, but searching `rule.groups:suricata` in Wazuh returned nothing. I went back to checking one stage at a time.

### Dead end #4: the eve syslog output wasn't on

```bash
grep -c "returned root" /var/log/suricata/latest.log
# 0
```

Suricata wasn't sending alerts to syslog at all. **Enable eve syslog output** is hidden behind **advanced mode** on the Settings page, and it wasn't checked. I turned it on.

I also found that `latest.log` is a symlink to *yesterday's* log file. OPNsense writes a new `suricata_YYYYMMDD.log` every day. Searching `*.log` covers all of them.

### Dead end #5: the test site is dead

Even after that, `eve.json` (where Suricata records every detection) showed zero hits for my test signature, ever. So I looked at what the curl actually returned:

```html
<title>testmynids.org</title>
<script ... src="https://assets.abovedomains.com/javascript/forsale.min.js ...">
```

**testmynids.org is a parked domain that's up for sale now.** It doesn't return the `uid=0(root)` text anymore, so there was nothing to detect. Every test I'd run with it was meaningless. The alerts I'd seen the night before were other signatures entirely.

I built my own trigger instead. Kali serves the string:

```bash
mkdir -p /tmp/t && echo "uid=0(root) gid=0(root) groups=0(root)" > /tmp/t/index.html \
  && cd /tmp/t && python3 -m http.server 8080
```

And Wazuh fetches it, so the traffic crosses OPNsense from LABNET to LAN:

```bash
curl http://192.168.10.102:8080/
```

On OPNsense:

```bash
grep -c "returned root" /var/log/suricata/eve.json                 # 2
grep -c "returned root" /var/log/suricata/suricata_20261029.log    # 2
```

Detection worked, and the alert went to syslog. **Everything on the OPNsense side was now working.**

### Dead end #6: a false positive in my own check

On Wazuh:

```bash
sudo grep -c "returned root" /var/ossec/logs/alerts/alerts.json
# 1
```

I thought that meant success. It didn't. That match was **my own `sudo grep "returned root"` command**, which Wazuh logged as a *Successful sudo to ROOT executed* alert. Searching for the real rule told the truth:

```bash
sudo grep -c '"id":"86601"' /var/ossec/logs/alerts/alerts.json
# 0
```

**Lesson:** when you grep for a string, make sure your own search command isn't what's matching.

## Part 6: The actual root cause — no decoder

To see exactly what Wazuh was receiving, I temporarily turned on raw log archiving (`<logall>yes</logall>`) and pulled the line:

```
Oct 29 16:04:38 fw01.home.arpa suricata[63207]: {"timestamp":"2026-10-29T16:04:38.528636-0500",...,"event_type":"alert",...,"signature":"GPL ATTACK_RESPONSE id check returned root",...}
```

That's a normal syslog header followed by Suricata's eve JSON, so it looked fine. Then I ran it through Wazuh's built-in tester:

```bash
sudo /var/ossec/bin/wazuh-logtest < /tmp/line.txt
```

```
**Phase 1: Completed pre-decoding.
        hostname: 'fw01.home.arpa'
        program_name: 'suricata'

**Phase 2: Completed decoding.
        No decoder matched.

**Phase 3: Completed filtering (rules).
        id: '1002'
        level: '2'
        description: 'Unknown problem somewhere in the system.'
```

That's the whole problem. Wazuh read the syslog header correctly, but **its built-in JSON decoder didn't pick up JSON that comes after a syslog header**. With no decoder, the JSON was never parsed, so the Suricata rules (which expect `decoded_as json`) never had a chance. The event fell through to a generic level-2 rule, below the alert threshold, and was silently dropped.

### The fix

I added a custom decoder to `/var/ossec/etc/decoders/local_decoder.xml`:

```xml
<decoder name="json">
  <program_name>^suricata</program_name>
  <plugin_decoder>JSON_Decoder</plugin_decoder>
</decoder>
```

It says: *when the program is `suricata`, parse the rest of the message as JSON.* Naming it `json` makes it a sibling of the built-in JSON decoder, so Wazuh's existing Suricata rules match it with no changes.

After restarting the manager, I ran logtest again:

```
**Phase 2: Completed decoding.
        name: 'json'
        alert.signature: 'GPL ATTACK_RESPONSE id check returned root'
        alert.signature_id: '2100498'
        src_ip: '192.168.10.102'
        dest_ip: '192.168.100.20'
        ...

**Phase 3: Completed filtering (rules).
        id: '86601'
        level: '3'
        description: 'Suricata: Alert - GPL ATTACK_RESPONSE id check returned root'
```

And in the dashboard, searching `rule.groups:suricata` showed **22 hits**:

- **GPL ATTACK_RESPONSE id check returned root**: my test
- **ET INFO Python SimpleHTTP ServerBanner**: Suricata also flagged the Python server I used for the test, which was a nice bonus catch
- **ET INFO Session Traversal Utilities for NAT (STUN)**: Tailscale on Wazuh doing NAT traversal. It's harmless, but it's the noisiest alert, so it's the first thing to tune.

Then I turned `logall` back off so the archive doesn't fill the disk.

> **Bonus find:** OPNsense's firewall logs were *already* decoding correctly. Wazuh's built-in pfSense rules (like 87702, *Multiple pfSense firewall blocks events from same source*) handle OPNsense's `filterlog` format as-is, so no custom decoder was needed there.

---

## What actually broke, in order

1. Hyperscan pattern matcher not supported in the VM → switched to Default
2. Suricata watching **WAN** instead of LAN and LABNET
3. Remote syslog target not sending until re-applied and `syslog-ng` restarted
4. A stray `</remote>` in `ossec.conf` stopping the Wazuh manager from starting
5. Wazuh missing its default route (a manual route that was never saved)
6. Eve syslog output turned off (hidden behind advanced mode)
7. testmynids.org now a parked domain, so the test never could have fired
8. **No decoder** for Suricata JSON behind a syslog header, the actual reason nothing showed up in Wazuh

## What I learned

- **Test one stage at a time.** For a pipeline with five links, check each one on its own: is Suricata detecting (`eve.json`)? Is it sending to syslog (the daily log)? Is it reaching Wazuh (`tcpdump`)? Is Wazuh listening (`ss`)? Does a rule match (`alerts.json`)? Each check eliminated a whole section of the pipeline.
- **`wazuh-logtest` is the most useful tool in Wazuh.** It showed exactly which phase failed. Without it, I'd have been guessing at rules when the problem was decoding.
- **Verify your test before trusting it.** I spent a long time on a test that couldn't work because the test site was dead. Now I check that the trigger content is actually in the traffic.
- **Watch for your own commands in your results.** My grep matched the sudo log of my grep.
- **Check the logs for what a service is *actually* doing.** "Running" didn't mean "on the right interfaces." The startup log showed vtnet0.
- **One XML typo takes everything down.** Restart and check status right after any config edit.
- **SSH beats the VM console.** Partway through, I started SSHing into Wazuh from my Mac over Tailscale so I could copy and paste commands and output instead of retyping and screenshotting. I should have started there.

## What's next

1. **Tune out the STUN noise** with a local rule that drops it to level 0 (or by disabling that signature in OPNsense). This is my first real tuning exercise.
2. **Run real attacks:** nmap and Metasploit from Kali against Metasploitable and DVWA, then investigate the resulting alerts in Wazuh the way an analyst would.
3. **Tighten the temporary allow-all rule** on LABNET.
4. **Look at the pfSense block alerts** (87702) to see what's being blocked and from where.
