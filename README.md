# DFIR-LinuxAnalysis
Production Linux system infected with Malware - I had to identify, contain, remediate the system after infection.

A production Linux system was recently infected due to a specific vulnerability we later identified as a configuration issue.

The system is an Asterisk-based phone system that was being utilized to commit toll fraud.

My first step was to identify the issue, which was very easily caught by our carrier notifying us immediately of suspicious calling activity. I then traversed to the phone system and checked the running logs of the system. I could see here that there are is a high amount of calls utilizing trunks I am not familiar with in this system.
<img width="975" height="463" alt="image" src="https://github.com/user-attachments/assets/df4a23dd-5442-42ad-8910-2a58bc83790e" />


After checking the active trunks on the system I couldn't see this malicious trunk in the GUI, only our typical trunks. After running commands, I was able to see that this trunk was active, and clearly hidden from the GUI. 

<img width="780" height="125" alt="image" src="https://github.com/user-attachments/assets/e40c47f6-d2d9-4f9d-b57b-4be57cc6aa6e" />


I started looking through SIP and PJSIP configuration and couldn't find any configuration for it, nor any include statements linking extra files. I then listed the asterisk directory to find a custom config file creating and referncing this trunk.

I then removed the configuration, and reloaded the asterisk configuration. After looking at the logfiles again, fraudulent calls were no longer occurring, as the trunk they were utilizing no longer existed.

This is a win, as we've contained the actual intent of the attack, but it does not change the fact the the system is still infected, and will likely soon reinstate the malicious trunk.

I first looked into the users on the system, and identified an unfamiliar user called Juba. After further inspection it has tied itself to the UID 0, along with root. This seems to be an obfuscation technique, and likely more. 
<img width="798" height="246" alt="image" src="https://github.com/user-attachments/assets/999ef41f-03ec-4e45-999b-c40d63f64a55" />


I used ps aux to check the running process on the system.
I could now see a suspcious process running in /usr/bin/cleanup.

<img width="975" height="26" alt="image" src="https://github.com/user-attachments/assets/31aa7f7a-9f6d-46f6-b9a0-4865210f1f81" />


I check the file type, to find it was a PHP script.
I cat'd the file to find obfuscated PHP. 

<img width="975" height="374" alt="image" src="https://github.com/user-attachments/assets/8e783b6b-bbbb-443a-88ad-0af3e1d69c33" />


Using the stat command I could see that this file was created last year, meaning the malware had established a connection for months now.
<img width="975" height="236" alt="image" src="https://github.com/user-attachments/assets/47fef934-bce7-4c1b-9483-483a0b8e73c0" />


I also checked for any malicious cronjobs using "crontab -e", but there were none.


At this point I needed to make an assessment:
Is the time I would spend investigating and removing this malware worth the time it would take to restore from backup. Furthermore, if restoring from backup was the better choice, how would I circumvent the fact that the malware had been present since at least 6 months?

The answer to the first question is restoring from backups is a more time and cost effective recovery technique. This will also allow me to keep the active system online and functioning during restore time, as the system can currently maintain its use without having any further negative affects.

The answer to the second question is that I can quickly spin up a system from scratch/template, and restore only module configuration. These modules are separate from the actual operating system, and only hold data pertaining to their specific function.

I can pull a recent backup, exclude any modules affected by the vulnerability and exploit, and rebuild those modules from scratch. 
That is exactly what I did. I restore the system on a private-side VM - meaning it was not yet exposed to the public internet. In this state, I reconfigure the modules safely and test to confirm the exploit was no longer an option. This also allowed me to configure the system without taking the active system offline.

Once everything was ready and tested I pulled the old system offline and brought the new one onto the public VM interface with the same IP. I could see attempts using the same exploit that were no longer successful as I had remediately the vulnerability. 

Post-incident analysis - 
After finding and addressing the vulnerability, I deployed an eyes-on check of all similar systems and confirmed that the vulnerability existed nowhere else. I reviewed with the team what had happened they may all be aware of this going forward.

# DFIR — Linux Malware Analysis & Incident Response
> **Production VoIP System Compromised for Toll Fraud**

---

## Overview

| Field | Details |
|---|---|
| **System** | Asterisk-based VoIP / PBX |
| **Attack Type** | Toll Fraud via Hidden SIP Trunk |
| **Persistence Method** | Obfuscated PHP backdoor, rogue root-level user |
| **Discovery** | Carrier alert on anomalous outbound call volume |
| **Malware Present Since** | ~6 months prior to discovery |
| **Resolution** | Clean rebuild from backup; vulnerability patched |

---

## Initial Detection

The incident was first flagged by our carrier, who alerted us to an unusually high volume of suspicious outbound calling activity. Upon logging into the phone system, a review of the active call logs confirmed the concern — numerous calls were being placed through trunks that were unrecognized and unaccounted for in our standard configuration.

> *Anomalous call volume observed through unrecognized trunks in the Asterisk call log.*

![Call Log - Suspicious Activity](https://github.com/user-attachments/assets/df4a23dd-5442-42ad-8910-2a58bc83790e)

---

## Investigation

### Hidden SIP Trunk

A review of the GUI showed only our legitimate trunks — the malicious trunk was completely hidden from the interface. However, running CLI commands against the Asterisk process confirmed the trunk was very much active.

> *The rogue trunk was invisible in the GUI but visible via CLI — a deliberate obfuscation technique.*

![Hidden Trunk - CLI Discovery](https://github.com/user-attachments/assets/e40c47f6-d2d9-4f9d-b57b-4be57cc6aa6e)

### Rogue Configuration File

A thorough review of all SIP and PJSIP configuration files turned up nothing — no trunk definitions, no suspicious `include` directives. A directory listing of the Asterisk configuration path eventually revealed a **custom configuration file** that had been planted to define and register the fraudulent trunk without appearing in any standard config path.

### Rogue System User — `juba`

Digging into local user accounts revealed an unfamiliar user: **`juba`**. Further inspection confirmed this account had been assigned **UID 0**, effectively making it an alias for root. This is a well-known persistence and privilege escalation technique used to maintain covert root-level access.

> *The `juba` user sharing UID 0 with root — a classic privilege shadowing technique.*

![/etc/passwd - Rogue User](https://github.com/user-attachments/assets/999ef41f-03ec-4e45-999b-c40d63f64a55)

### Malicious Process — `/usr/bin/cleanup`

A review of running processes via `ps aux` revealed a suspicious entry: a process running from `/usr/bin/cleanup`. File inspection showed it was a **PHP script masquerading as a system utility**, containing heavily **obfuscated code** consistent with a remote access backdoor or C2 beacon.

> *A PHP-based backdoor disguised as a system cleanup utility.*

![ps aux - Suspicious Process](https://github.com/user-attachments/assets/31aa7f7a-9f6d-46f6-b9a0-4865210f1f81)

![/usr/bin/cleanup - Obfuscated PHP](https://github.com/user-attachments/assets/8e783b6b-bbbb-443a-88ad-0af3e1d69c33)

### Malware Age — `stat` Analysis

Running `stat` against the malicious file revealed it had been created **approximately six months prior** to discovery. The attacker had maintained persistent, undetected access to a production system for half a year.

> *File metadata confirmed the malware had been active for roughly six months.*

![stat - File Creation Date](https://github.com/user-attachments/assets/47fef934-bce7-4c1b-9483-483a0b8e73c0)

### Crontab Check

A review of all crontabs (`crontab -e`) revealed no malicious scheduled jobs — the malware relied on its persistent process and configuration file for reinstatement rather than cron-based scheduling.

---

## Recovery Decision

With the scope of the infection understood, a recovery strategy decision was required:

**Option A — Manual Remediation**
Continue forensic investigation, remove all artifacts, and harden in place.

**Option B — Clean Rebuild from Backup**
Restore the system from a known-good backup, excluding any affected modules.

**Decision: Option B — Clean Rebuild**

Given the malware's long dwell time, the risk of incomplete remediation was too high to justify manual cleanup. A rebuild approach offered a faster, more reliable path to a confirmed-clean state — while keeping the active production system online throughout the process.

---

## Remediation

### Containment
The malicious trunk configuration was removed and the Asterisk configuration was reloaded. Fraudulent call activity immediately ceased, as the trunk being leveraged for toll fraud no longer existed.

### Rebuild Strategy
Rather than restoring a full system image (which would reintroduce the malware), the recovery was targeted:

1. A **fresh system was provisioned from a clean template** on a private, internet-isolated VM
2. **Module configurations were selectively restored from backup**, explicitly excluding any modules tied to the vulnerability
3. Affected modules were **rebuilt from scratch** and reconfigured securely
4. The vulnerability was **confirmed patched** before exposure to any public network
5. The rebuilt system was **fully tested** in the isolated environment

### Cutover
Once testing was complete, the old system was taken offline and the new system was promoted to the public-facing interface, inheriting the same IP address. Post-cutover monitoring confirmed that further exploitation attempts using the same attack vector **failed against the hardened system**.

---

## Indicators of Compromise (IOCs)

| Type | Indicator |
|---|---|
| **Rogue User** | `juba` with UID 0 |
| **Malicious Binary** | `/usr/bin/cleanup` (PHP script) |
| **Malicious Config** | Custom Asterisk trunk config file in `/etc/asterisk/` |
| **Behavior** | Hidden SIP trunk, high-volume outbound international calls |

---

## Post-Incident Actions

- **Scope review** — All similar Asterisk systems were audited; the vulnerability was confirmed isolated to this one host
- **Team debrief** — The incident was reviewed with the full team to build organizational awareness and prevent recurrence
- **Documentation** — This writeup was produced to serve as a reference for future incident response efforts

---

## Key Takeaways

- **Long dwell times are common.** Six months of undetected access is not unusual for targeted, low-noise attacks. Proactive monitoring — not just reactive alerts — is essential.
- **Hiding in plain sight.** Both the malicious trunk (hidden from GUI but active in CLI) and the backdoor (named `cleanup`, placed in `/usr/bin/`) relied on looking unremarkable. Always verify CLI state against GUI state on critical systems.
- **UID 0 aliasing** is a simple but effective persistence technique that can be easily missed in routine user audits.
- **Targeted restores beat full rollbacks.** Separating OS state from application/module configuration made it possible to restore cleanly without reintroducing a six-month-old infection.
- **Test in isolation before cutover.** Rebuilding on a private VM allowed full validation — including confirming the vulnerability was patched — without any production downtime.

---

*Writeup by Dylan Haarris · Linux Malware Analysis and Incident Response*


I learned a lot about how Linux-based malware operates, and sharpened and tested my incident response and Linux skills through a high-pressure situation.




