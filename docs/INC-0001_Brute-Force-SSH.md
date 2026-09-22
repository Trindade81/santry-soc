# INC-0001 — SSH Brute-Force with Credential Compromise

| | |
|---|---|
| **Incident ID** | INC-0001 |
| **Title** | Successful SSH brute-force against a Linux host (credential compromise) |
| **Detected** | 2026-09-22, 18:54–18:55 (Europe/Dublin, UTC+1) |
| **Analyst** | Rafael "Ghost" Barrella |
| **Environment** | Santry SOC (home lab) — controlled exercise |
| **Severity** | High (Wazuh level 12 — confirmed compromise) |
| **Status** | Contained / Resolved (disposable target) |
| **MITRE ATT&CK** | T1110.001 (Brute Force: Password Guessing), T1078 (Valid Accounts) |

> **Nature of the exercise:** the attack was executed by me, in an isolated lab,
> against a disposable target created for this purpose. The goal was to validate
> the SOC's end-to-end detection capability and produce a case study. No real data
> or third party was involved. Addresses use the RFC 5737 documentation range.

---

## 1. Executive summary

An internal host (`192.0.2.177`) launched an **SSH brute-force** against
`victim-01` (`192.0.2.76`), targeting the user `victim`. After **14 failed
attempts in ~10 seconds**, the attacker **guessed the password** (`123456`) and
**opened a session** — a confirmed credential compromise.

The SOC **detected the full attack in ~13 seconds** through its **HIDS** layer
(Wazuh agent on the target), including the critical **level-12** alert
*"Multiple authentication failures followed by a success"*, which flags not merely
the attempt but the **successful compromise**.

**Key finding:** the **NIDS** layer (Suricata) produced **no alert**. The traffic
was **lateral** (attacker and target on the same LAN segment) and the switch mirror
covers only the internet-uplink port, leaving the sensor **blind to east-west
traffic**. Details and remediation in section 8.

---

## 2. Scope & environment

| Asset | Role | Detail |
|---|---|---|
| `192.0.2.177` | **Attacker** | Workstation (Linux), `hydra` 9.6 |
| `192.0.2.76` (`victim-01`) | **Target** | Debian 12 LXC on Proxmox; SSH + rsyslog; user `victim` (deliberately weak password); Wazuh agent 4.14 |
| `192.0.2.32` (`wazuh`) | **SIEM** | Wazuh manager/indexer/dashboard 4.14 (Proxmox VM) |
| `192.0.2.28` (`santry-pi`) | **NIDS sensor** | Suricata 7.0 (passive, switch mirror port) |

---

## 3. Timeline (Europe/Dublin, IST)

| Time (IST) | Event | Source |
|---|---|---|
| 18:54:51 | `hydra` launched at attacker against `ssh://192.0.2.76` | Attacker host |
| 18:54:54 | 1st SSH authentication failure on target | Wazuh rule 5760 |
| 18:54:54–18:55:01 | Cascade of 14 failures (source ports 46898/46912/46922/46936 — parallel hydra tasks) | auth.log / Wazuh 5760, 2502 |
| 18:55:02 | **Detection escalates:** brute force (5763) + **compromise (40112, level 12)** | Wazuh |
| 18:55:02 | **`Accepted password for victim` — credential `123456` accepted** | auth.log / Wazuh 5501 |
| 18:55:04 | Residual failures (parallel tasks finishing) | auth.log |

**Total intrusion window: ~13 seconds.** Detection was effectively real-time
(same second as the compromise).

---

## 4. Attack narrative (offensive side)

```
hydra -l victim -P wordlist.txt -t 4 -V ssh://192.0.2.76
...
[22][ssh] host: 192.0.2.76   login: victim   password: 123456
```

A 15-entry list of common weak passwords was used, `123456` last. After 14 failures
the attacker obtained a valid interactive session as `victim`.

---

## 5. Detection (defensive side)

### 5.1 HIDS — Wazuh agent on the target (primary detection)

| rule.id | Description | Level |
|---|---|---|
| 5760 | sshd: authentication failed | 5 |
| 5503 | PAM: User login failed | 5 |
| 2502 | syslog: User missed the password more than one time | 10 |
| 5763 | sshd: brute force trying to get access to the system | 10 |
| **40112** | **Multiple authentication failures followed by a success** | **12** |
| 5501 | PAM: Login session opened | 3 |

Dashboard summary for the attack window: **21 authentication failures · 1 success ·
1 alert at level ≥12**. Automatic ATT&CK mapping: **Brute Force, Password Guessing,
Valid Accounts, SSH**.

Rule **40112 (level 12)** is the highest-value signal: it does not just record the
brute-force — it identifies that it **succeeded**, indicating an **active
compromise** rather than a mere attempt.

### 5.2 NIDS — Suricata (no detection)

Suricata produced **no alert** for this attack. See section 8.

---

## 6. Indicators of Compromise (IOCs)

- **Source IP:** `192.0.2.177` (internal)
- **Target:** `192.0.2.76:22`, user `victim`
- **Compromised credential:** `victim` / `123456`
- **Network pattern:** burst of SSH failures from a single source using multiple
  concurrent source ports (46898, 46912, 46922, 46936) — signature of a parallel
  brute-force tool
- **Process:** `sshd`

---

## 7. Evidence & chain of custody

| Artifact | Detail |
|---|---|
| `/var/log/auth.log` (`victim-01`) | Target authentication log — primary evidence |
| **SHA256** | `6b5e41bd472900aba430fd0db167f15ee243bd974e1d40424d7dbd6ed5a8c352` |
| Wazuh alerts | Index `wazuh-alerts`, agent `victim-01`, window 18:54–18:55 IST |

> **Forensic note:** because this was an **east-west** attack not seen by the
> network IDS, **no PCAP** of the intrusion exists in the Suricata ring buffer.
> Evidence is entirely **host-based**. That is itself a finding — see section 8.

---

## 8. Root cause analysis & key finding

**Why was the NIDS blind?** The managed switch mirrors **port 1 → port 4 (sensor)**.
Port 1 is the **internet uplink**, so Suricata only sees traffic entering/leaving
the internet (north-south). This attack was **LAN-local**: the switch forwards the
frames **directly between the two access ports**, never crossing the mirrored port.
Result: **east-west traffic is invisible to the sensor.**

This is a classic **sensor-placement** error: an uplink SPAN does not cover lateral
movement — precisely the traffic most relevant to catching an attacker already
inside the network.

**Why detection still worked (defense in depth):** because the target ran a Wazuh
agent, host-based detection captured the attack in full. Layered detection is what
saved the outcome.

---

## 9. Impact

- **Confidentiality:** compromised — a valid credential was obtained.
- **Integrity / Availability:** no observed change (disposable target; session not
  exploited beyond login).
- **Real-world reach:** none (isolated lab, ephemeral host).

---

## 10. Response & containment

In a production setting the response would be: isolate the host, revoke/rotate the
`victim` credential, block the source IP, review sessions and persistence. In this
exercise the target is disposable and was retained only for evidence collection.

---

## 11. Recommendations

**Close the detection gap (NIDS):**
1. Add the server segment's switch port as an additional mirror source (the switch
   supports multiple sources → one destination) so the sensor sees host/VM traffic;
   then **re-validate** by repeating this attack. *(Tracked as INC-0001b.)*

**Harden SSH on targets:**
2. Disable password authentication (**public-key only**).
3. `PermitRootLogin no`; strong passwords where a password is unavoidable.
4. **Fail2Ban** (or sshd `MaxAuthTries` / rate-limiting) for automatic blocking.

**SOC operations:**
5. Configure **active notification** for level ≥12 rules (compromise) — currently
   dashboard-only.

---

## 12. Lessons learned

- **HIDS and NIDS are complementary, not interchangeable.** The NIDS was blind to
  lateral movement; the HIDS covered it. Defense in depth is what delivered the
  detection.
- **Sensor placement defines visibility.** An uplink SPAN does not detect east-west
  attacks.
- **The alert that matters is not "they tried" — it's "they got in."** Rule 40112
  (failures followed by a success) is the pivot from noise to incident.
- **Near-real-time detection is achievable on modest hardware** — the pipeline
  flagged the compromise within the same second.

---

## Appendix A — `auth.log` excerpt (`victim-01`, UTC)

```
2026-09-22T17:54:54 victim-01 sshd[4438]: Failed password for victim from 192.0.2.177 port 46912 ssh2
2026-09-22T17:54:54 victim-01 sshd[4437]: Failed password for victim from 192.0.2.177 port 46936 ssh2
... (14 failures total, ports 46898/46912/46922/46936) ...
2026-09-22T17:55:02 victim-01 sshd[4437]: Accepted password for victim from 192.0.2.177 port 46936 ssh2
```

## Appendix B — Attack command

```
hydra -l victim -P wordlist.txt -t 4 -V ssh://192.0.2.76
[22][ssh] host: 192.0.2.76   login: victim   password: 123456
```

---

*INC-0001 — Santry SOC v0.1. First documented incident. Next: close the NIDS gap
(extend the mirror to the server segment) and re-validate (INC-0001b).*
