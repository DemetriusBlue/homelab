# Detection Rules

Custom Wazuh decoders and rules for my homelab, plus a log of every tuning decision and why I made it.

Alerts come from two places:
- **Suricata on OPNsense** (network IDS), forwarded to Wazuh over syslog
- **Wazuh agents** (host events like logins, sudo, and file changes)

See [`../network/network.md`](../network/network.md) for how traffic and logs flow through the lab.

---

## Inventory

| ID | Type | Name | What it does | File | MITRE ATT&CK | Status |
|---|---|---|---|---|---|---|
| — | Decoder | `json` (Suricata syslog) | Parses Suricata eve JSON that arrives behind a syslog header | [`decoders/suricata-syslog.xml`](decoders/suricata-syslog.xml) | — | ✅ Active |
| 100100 | Rule (tuning) | Suppress Tailscale STUN request | Drops STUN alerts *from* wazuh-server to level 0 | [`rules/tuning.xml`](rules/tuning.xml) | — | ✅ Active |
| 100101 | Rule (tuning) | Suppress Tailscale STUN response | Drops STUN alerts *to* wazuh-server to level 0 | [`rules/tuning.xml`](rules/tuning.xml) | — | ✅ Active |

Detection rules (ones that catch attacker behavior) will get an ATT&CK technique in that column. For example, a scan detection would map to T1046, Network Service Discovery.

---

## Decoder: Suricata JSON over syslog

**Why it exists**

OPNsense forwards Suricata alerts as a syslog header followed by eve JSON:

```
Oct 29 16:04:38 fw01.home.arpa suricata[63207]: {"timestamp":"2026-10-29T16:04:38.528636-0500",...,"event_type":"alert",...,"alert":{...,"signature":"GPL ATTACK_RESPONSE id check returned root",...}}
```

Wazuh's built-in JSON decoder didn't match this format. Every Suricata alert reached Wazuh and was silently dropped.

**Before** (`wazuh-logtest`):

```
**Phase 1: Completed pre-decoding.
        program_name: 'suricata'

**Phase 2: Completed decoding.
        No decoder matched.

**Phase 3: Completed filtering (rules).
        id: '1002'
        level: '2'
        description: 'Unknown problem somewhere in the system.'
```

**The decoder**

```xml
<decoder name="json">
  <program_name>^suricata</program_name>
  <plugin_decoder>JSON_Decoder</plugin_decoder>
</decoder>
```

Naming it `json` makes it a sibling of the built-in JSON decoder, so Wazuh's existing Suricata rules (86600/86601, which require `decoded_as json`) work without changes.

**After:**

```
**Phase 2: Completed decoding.
        name: 'json'
        alert.signature: 'GPL ATTACK_RESPONSE id check returned root'
        alert.signature_id: '2100498'
        src_ip: '192.168.10.102'
        dest_ip: '192.168.100.20'

**Phase 3: Completed filtering (rules).
        id: '86601'
        level: '3'
        description: 'Suricata: Alert - GPL ATTACK_RESPONSE id check returned root'
```

Full story: [`../writeups/suricata-wazuh-pipeline.md`](../writeups/suricata-wazuh-pipeline.md)

---

## Rules 100100 / 100101: Suppress Tailscale STUN

**Why it exists**

As soon as Suricata alerts started reaching Wazuh, most of them were the same thing:

> Suricata: Alert - ET INFO Session Traversal Utilities for NAT (STUN Binding Request / Response)

That's Tailscale on `wazuh-server` checking for the best path to its relay servers. It's expected and harmless, but it buried the alerts that matter. When I counted a day's alerts by rule, it wasn't close:

```
4417 Suricata: Alert - ET INFO Session Traversal Utilities for NAT (STUN Binding Request)
4250 Suricata: Alert - ET INFO Session Traversal Utilities for NAT (STUN Binding Response)
 216 Suricata: Alert - ET DOS Possible SSDP Amplification Scan in Progress
  61 Multiple pfSense firewall blocks events from same source.
  ...
```

STUN alone was about **95% of all alerts**.

Counted with:

```bash
sudo grep -o '"description":"[^"]*"' /var/ossec/logs/alerts/alerts.json | sort | uniq -c | sort -rn | head -15
```

**Why two rules, and why they're narrow**

- Requests have Wazuh as the **source**, and responses have Wazuh as the **destination**, so one rule covers each direction.
- Both are scoped to `192.168.100.20` only. If STUN shows up from any **other** host (Kali, a target VM, something new on the network), it still alerts. Unexpected STUN can indicate tunneling or a remote-access tool, so I don't want it suppressed everywhere.
- Level 0 means the event is still processed, but not stored or shown as an alert.

```xml
<rule id="100100" level="0">
  <if_sid>86601</if_sid>
  <field name="alert.signature">Session Traversal Utilities for NAT</field>
  <field name="src_ip">^192.168.100.20$</field>
  <description>Tuning: Tailscale STUN request from wazuh-server (expected, suppressed)</description>
</rule>
```

Rule 100101 is identical but matches `dest_ip`.

**How I verified it**

I compared the manager's restart time to the newest STUN alert:

```bash
$ systemctl show -p ActiveEnterTimestamp wazuh-manager
ActiveEnterTimestamp=Fri 2026-10-30 04:16:01 UTC

$ sudo grep "STUN" /var/ossec/logs/alerts/alerts.json | tail -1 | grep -o '"timestamp":"[^"]*"'
"timestamp":"2026-10-30T04:15:40.991+0000"
```

The last STUN alert came in 21 seconds *before* the rules loaded, and none after. At the previous rate, there would have been dozens in the following five minutes.

> **Gotcha:** my first check (`tail -n 300 alerts.json | grep -c STUN`) returned 48, which looked like a failure. Those were all from before the restart. Compare timestamps, not counts, when checking whether a change took effect.

**Still to do:** re-run the custom trigger test (Kali's Python server and a curl from Wazuh) to confirm real alerts like *GPL ATTACK_RESPONSE id check returned root* still come through.

---

## Tuning log

Every suppression gets a dated entry with the reason. If I can't justify it, it doesn't get suppressed.

| Date | Change | Rule | Reason | Scope |
|---|---|---|---|---|
| 2026-10-29 | Suppressed ET INFO STUN alerts | 100100, 100101 | Tailscale NAT traversal on wazuh-server. Expected, benign, about 95% of all alert volume (~8,700 alerts in one day). Verified by timestamp after restart. | `192.168.100.20` only |

---

## Conventions

- **Rule IDs:** custom rules start at **100000**. Wazuh reserves the lower ranges for its built-in rules.
  - `100100–100199` — tuning and suppression
  - `100200+` — detections (planned)
- **Decoders** go in `/var/ossec/etc/decoders/local_decoder.xml` on the Wazuh server.
- **Rules** go in `/var/ossec/etc/rules/local_rules.xml`.
- **Suppressions are scoped** to the specific host, IP, or signature. No blanket disables.

## Testing and deploying

1. **Test first** with a real log line:
   ```bash
   sudo /var/ossec/bin/wazuh-logtest < sample.log
   ```
2. **Deploy:** add the XML to the right local file on Wazuh.
3. **Restart and check status right away.** One XML typo stops the whole manager.
   ```bash
   sudo systemctl restart wazuh-manager
   sudo systemctl status wazuh-manager
   ```
4. **Confirm in the dashboard** that the change did what it should, and nothing else.

Deployment will eventually move into an Ansible playbook in [`../automation`](../automation).

---

## Planned

- [x] Verify and activate rules 100100 / 100101
- [ ] Re-run the trigger test to confirm real Suricata alerts still come through
- [ ] Investigate the ET DOS SSDP Amplification Scan alerts (216 in one day): find the source, then tune or escalate
- [ ] Review the pfSense firewall block alerts (87702) and tune or escalate
- [ ] Suppress the ET INFO Tailscale DERP Relay alert, scoped to wazuh-server
- [ ] Alert when a lab target hits the LABNET default-deny rule. A target trying to reach the internet could mean command-and-control (T1071) or exfiltration attempts.
- [ ] Detection for nmap scans from the attacker LAN (T1046)
- [ ] Detection for brute-force attempts against lab hosts (T1110)
