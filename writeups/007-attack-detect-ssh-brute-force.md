# Attack & Detect: Recon and SSH Brute Force

**Date:** November 12–13, 2026
**Goal:** Run common attacker techniques against a vulnerable host and measure what my detection stack catches, what it misses, and why.

## Lab setup

```
 Kali (attacker)            OPNsense                 Metasploitable 3 (target)
 192.168.10.102   ───────►  firewall + Suricata ───► 192.168.100.30
 LAN                        (network detection)      labnet (VXLAN)
                                   │                        │
                                   │ alerts                 │ Wazuh agent
                                   ▼                        ▼
                              Wazuh manager (SIEM) — 192.168.100.20
```

- **Attacker:** Kali Linux on its own subnet. Every attack has to route through OPNsense.
- **Target:** Metasploitable 3 (Ubuntu 14.04) with the Wazuh 4.14.6 agent installed.
- **Network detection:** Suricata IDS on OPNsense, forwarding alerts to Wazuh.
- **Host detection:** Wazuh agent shipping auth, syslog, and web server logs.

Two layers means I can compare what the network sees with what the host sees for the same attack.

## Attacks mapped to MITRE ATT&CK

| # | Attack | Tool | ATT&CK technique |
|---|---|---|---|
| 1 | Port and service scan | `nmap -sS -sV` | T1046 Network Service Discovery |
| 2 | SSH password guessing | `hydra` | T1110.001 Brute Force: Password Guessing |
| 3 | Login with guessed credentials | `hydra` (success) | T1078.003 Valid Accounts: Local Accounts |

## Detection results

### Attack 1: Recon (`nmap -sS -sV 192.168.100.30`)

| Layer | Rule | Level | Alert |
|---|---|---|---|
| Suricata | 86601 | 3 | ET SCAN Possible Nmap User-Agent Observed |
| Host | 31101 | 5 | Web server 400 error code |

**Detected, but only because of `-sV`.** Both alerts came from Nmap's version-detection probes, which send HTTP requests with an Nmap user agent. Nothing fired on the SYN scan itself.

**Gap:** a scan without `-sV`, or with a spoofed user agent, likely goes unseen. *(Not yet tested.)*

### Attacks 2 & 3: SSH brute force with a successful login

I ran this twice with different wordlist sizes. Same tool, same target.

| Rule | Level | Alert | 6 attempts | 15 attempts (after fixes) |
|---|---|---|---|---|
| 5760 | 5 | sshd: authentication failed | ✅ | ✅ |
| 5503 | 5 | PAM: user login failed | ✅ | ✅ |
| 5715 | 3 | sshd: authentication success | ✅ | ✅ |
| 5501 / 5502 | 3 | PAM session opened / closed | ✅ | ✅ |
| 2502 | 10 | User missed the password more than once | — | ✅ |
| **5763** | **10** | **sshd: brute force trying to get access** | ❌ | ✅ |
| **40112** | **12** | **Multiple authentication failures followed by a success** | ❌ | ✅ |

The two bold rows are the alerts that matter. Individual failed logins are noise; **40112** tells an analyst that someone guessed their way in.

**Why they didn't fire at first:**
- With 6 attempts, the attack stayed under the correlation threshold (roughly 8 failures from one source).
- With 15 attempts, they *still* didn't fire, because detection was broken. The OS logging config was collapsing repeated failures into "message repeated N times" lines, which hid the attacker's IP from Wazuh. Fixing that, plus two lab infrastructure problems, is covered in [the troubleshooting writeup](ssh-brute-force-detection-gap.md).

**Network side:** Suricata raised nothing SSH-related for the 6-attempt run. *(Not yet checked for the 15-attempt runs.)*

### A useful behavioral tell

Every successful Hydra login produced **5501 session opened** and **5502 session closed** within the same second. A person who logs in stays logged in. A tool connects, confirms the password works, and disconnects. That pattern alone is worth hunting for.

## Detection gaps

| Gap | Impact | Fix |
|---|---|---|
| Low-volume guessing stays under the brute-force threshold | A patient attacker with a short, targeted list gets in with no high-severity alert | Custom rule: success after 3+ failures from the same IP |
| Plain SYN scan likely undetected | Recon goes unnoticed | Test `nmap -sS` alone; tune or add a Suricata scan rule |
| All Suricata alerts are level 3 | Network alerts get buried in a real queue | Raise ET SCAN alerts to level 6–8 |
| SSDP alerts about once a minute | Noise that trains analysts to ignore Suricata | Scoped suppression rule |
| Host logging defaults can break correlation | Detection silently fails | Check that `data.srcip` is extracted for every new log source |

## Takeaways

- **Layered detection matters.** The network saw the recon; the host saw the brute force. Neither layer saw everything.
- **An alert firing isn't the same as detection working.** I had plenty of low-level alerts the whole time. The one that mattered didn't fire until I found why.
- **Know your thresholds.** The same attack is invisible or critical depending on how many guesses it takes. Attackers know this too.

## Next labs

- Plain SYN scan to confirm the recon gap
- Custom Wazuh rule for low-and-slow brute force, then rerun the 6-attempt attack to prove it catches it
- Exploit one of the services the scan found (Samba, Jetty, MySQL) and see what the host agent picks up
