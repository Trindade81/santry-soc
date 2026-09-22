# Build Notes — Why, What Broke, What Worked

This is the honest version of the project: why I built it, the things that went
wrong, how I worked them out, and what actually landed. The troubleshooting is the
point. In a SOC, figuring out *why something isn't behaving* is most of the job.

Addresses use the RFC 5737 documentation range (`192.0.2.0/24`) and are illustrative.

---

## Why I built this

I'm moving into cybersecurity, aiming for a SOC analyst role, and I don't learn
well from theory alone. I wanted to understand detection **from the inside**: what
a sensor really sees, what a SIEM does with a log line, and what an attack looks
like from both ends.

So I built a small SOC at home from gear I already had, on purpose. No cloud, no
managed anything. If it broke, I had to understand it and fix it. Two rules I kept
the whole way:

- **Verify by artifact, not by screen.** "I think it worked" is not a state. A
  service being *running* is not the same as the control *working*. I only closed a
  step when there was real evidence on disk (a log line, a hash, an exit code).
- **A diagram is not an inventory.** What I drew was the target; only the live
  machine tells the truth. That rule paid off more than once (see below).

---

## What broke, and how I worked it out

### 1. The "3222" that looked like a ghost
While reading BIOS info, the number `3222` kept showing up right after a password
prompt. I thought I had phantom keyboard input, maybe a stuck key or a macro on the
mouse. I chased it: a bare `cat` to catch untyped input, checked the clipboard,
listed every HID device. All clean.

The truth was boring and useful: `3222` was the **BIOS version**, printed by
`dmidecode`. Reading the screen fooled me; the artifact (the sysfs file) didn't.
**Lesson:** when something looks spooky, get the artifact before inventing a theory.

### 2. A minimal Debian ships with almost nothing
The Wazuh SIEM ran on a minimal Debian VM that had no `curl` and no `sudo`. The
install script just failed. Fix: install what's missing, and when `sudo` isn't even
there, become root with `su -`. **Lesson:** minimal images are minimal on purpose;
don't assume the basics are present.

### 3. The Wazuh agent that pointed at nothing
On the target, the agent installed but wouldn't start. Two reasons stacked up:
a missing `lsb-release` dependency, and the manager address left as the literal
placeholder `MANAGER_IP`. The installer only applies the `WAZUH_MANAGER` variable on
a *first* install, not on a reconfigure. Fix: install the dependency, then set the
manager address directly in `ossec.conf` and restart. **Lesson:** know the
difference between a package installing and a package *configuring*.

### 4. A static IP that killed DNS
I gave the SIEM VM a static IP so agents wouldn't lose it on a DHCP change. It came
back up, pinged the internet by IP, but couldn't resolve any name. A `dhcpcd`
process was rewriting `/etc/resolv.conf` on every boot with no nameserver. Fix: pin
the nameservers and lock the file with `chattr +i` so nothing blanks it again.
**Lesson:** always know *who owns* `resolv.conf` on a box before you go static.

### 5. The switch wasn't where my notes said
When I went to reconfigure the switch, my documentation said one management IP, and
it simply wasn't there. Neither the laptop nor the wired hypervisor could reach it.
Turned out the switch was on DHCP at a completely different address, which I found in
the router's device list. **Lesson:** the classic one again, a diagram is not an
inventory. Trust the live network, then fix the doc.

### 6. Log rotation that never rotated
The Suricata `eve.json` was growing without limit. The rotation config that shipped
with the package had **no schedule**, so it silently fell back to weekly. I set it
to daily with `copytruncate`, which keeps the same file handle so the Wazuh agent
keeps reading without missing a beat. **Lesson:** "there's a config file for it"
does not mean it's doing anything useful. Read it.

### 7. The blind sensor (the big one)
This became the first documented incident, [INC-0001](INC-0001_Brute-Force-SSH.md).
Short version: I attacked a host on my own LAN, the host-based agent caught it in
about 13 seconds, and the network sensor (Suricata) saw *nothing*. The switch mirror
only covered the internet uplink, so lateral traffic never reached the sensor. A
textbook sensor-placement blind spot. The fix (extend the mirror to the server
segment) is the next thing on the list.

---

## What worked

- **The pipeline runs end to end.** Traffic to Suricata to the agent to the manager
  to the dashboard, with host-based detection on top. Real events, on real disk.
- **Detection was near real-time.** The brute-force compromise was flagged in the
  same second it happened, by the rule that fires on failed logins *followed by a
  success* (the one that means "they got in", not "they tried").
- **Defense in depth actually saved the day.** When the network sensor was blind,
  the host layer covered it. That wasn't luck, it was the design.
- **Version discipline paid off.** Keeping the Wazuh agent and manager on the same
  version meant enrolment just worked.

---

## Principles I'm keeping

- Verify by artifact. A diagram is not an inventory. "Acho que fiz" (I think I did
  it) is not a state.
- Back up a config before editing it.
- Finish what already exists before buying new hardware.
- HIDS and NIDS are complementary. Where you place a sensor decides what you can see.

---

*These notes are deliberately unpolished about the failures, because the failures
are where the learning was.*
