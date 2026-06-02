# DFIR — Linux Malware Analysis & Incident Response
> **Production VoIP System Compromised for Toll Fraud**

---

## Overview

| Field | Details |
|---|---|
| **System** | Asterisk-based Linux VoIP System |
| **Attack Type** | Toll Fraud via Hidden SIP Trunk |
| **Persistence Method** | Obfuscated PHP backdoor, rogue root-level user |
| **Discovery** | Carrier alert on anomalous outbound call volume |
| **Vulnerability** | ACL-based configuration vulnerability |
| **Resolution** | Clean rebuild from backup; vulnerability patched |

---

## Initial Detection

The incident was first flagged by our carrier, who alerted us to an unusually high volume of suspicious outbound calling activity. Upon logging into the phone system, a review of the active call logs confirmed the concern. Numerous calls were being placed through trunks that were unrecognized and unaccounted for in our standard configuration.

> *Anomalous call volume observed through unrecognized trunks in the Asterisk call log.*

![Call Log - Suspicious Activity](https://github.com/user-attachments/assets/df4a23dd-5442-42ad-8910-2a58bc83790e)

---

## Investigation

### Hidden SIP Trunk

A review of the GUI showed only our legitimate trunks. The malicious trunk was completely hidden from the interface. However, running CLI commands against the Asterisk process confirmed the trunk was very much active.

> *The rogue trunk was invisible in the GUI but visible via CLI. This was an intentional obfuscation technique.*

![Hidden Trunk - CLI Discovery](https://github.com/user-attachments/assets/e40c47f6-d2d9-4f9d-b57b-4be57cc6aa6e)

### Rogue Configuration File

A thorough review of all SIP and PJSIP configuration files turned up nothing. There were no trunk definitions, no suspicious include directives referencing other config files. A directory listing of the Asterisk configuration path eventually revealed a **custom configuration file** that had been planted to define and register the fraudulent trunk without appearing in any standard config path.

### Rogue System User — juba

Digging into local user accounts revealed an unfamiliar user: **juba**. Further inspection confirmed this account had been assigned **UID 0**, effectively making it an alias for root. This is a well-known persistence and privilege escalation technique used to maintain covert root-level access.

> *The juba user sharing UID 0 with root.*

![/etc/passwd - Rogue User](https://github.com/user-attachments/assets/999ef41f-03ec-4e45-999b-c40d63f64a55)

### Malicious Process — /usr/bin/cleanup

A review of running processes via **ps aux** revealed a suspicious entry: a process running from **/usr/bin/cleanup**. File inspection showed it was a **PHP script masquerading as a system utility**, containing heavily **obfuscated code** likely being used as a backdoor.

> *A PHP-based backdoor disguised as a system cleanup utility.*

![ps aux - Suspicious Process](https://github.com/user-attachments/assets/31aa7f7a-9f6d-46f6-b9a0-4865210f1f81)

![/usr/bin/cleanup - Obfuscated PHP](https://github.com/user-attachments/assets/8e783b6b-bbbb-443a-88ad-0af3e1d69c33)

### Malware Age — stat Analysis

Running **stat** against the malicious file revealed it had been created approximately six months prior to discovery. 

> *File metadata confirmed the malware had been active for at least six months.*

![stat - File Creation Date](https://github.com/user-attachments/assets/47fef934-bce7-4c1b-9483-483a0b8e73c0)

### Crontab Check

A review of all crontabs using **crontab -e** revealed no malicious scheduled jobs. The malware relied on its persistent process and configuration file for reinstatement rather than cron-based scheduling.

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
Rather than restoring a full system image (which would reintroduce the malware), the recovery was done in a much more intentional manner:

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
| **Rogue User** | juba with UID 0 |
| **Malicious Binary** | /usr/bin/cleanup (PHP script) |
| **Malicious Config** | Custom Asterisk trunk config file in main asterisk directory |
| **Behavior** | Hidden SIP trunk, high-volume outbound international calls |

---

## Post-Incident Actions

- **Scope review** — All similar Asterisk systems were audited; the vulnerability was confirmed isolated to this one host
- **Team debrief** — The incident was reviewed with the team to build organizational awareness and prevent recurrence
- **Documentation** — This incident and resolution has been documented to be referenced in the future should a similar incident occur.

---

## What I learned/sharpened

- **Linux-based incident response** I learned more about how to investigate active and passive attacks on Linux-based systems, and sharpened my existing skills. It gave me new IOCs and techniques to be on the look out for, and write SIEM/SOAR rules to catch and respond to.
- **Hiding in plain sight.** Both the malicious trunk (hidden from GUI but active in CLI) and the backdoor (named cleanup, placed in /usr/bin/) relied on looking non-malicious or hidden. Always verify CLI state against GUI state, and investigate everything, even if it looks normal.
- **UID 0 aliasing** is a simple but effective persistence technique to have a user account running everything as the UID 0 administrative user. This can easily have a SIEM rule written to catch this occurance again.
- **Targeted restores can be better than a full system image recovery** Separating OS state from application/module configuration made it possible to restore cleanly without reintroducing a six-month-old infection.
- **Test in isolation before cutover.** Rebuilding on a private VM allowed for validation of system health, including confirming the vulnerability was patched, without any production downtime.

---

*Writeup by Dylan Harris · Linux Malware Analysis and Incident Response*
