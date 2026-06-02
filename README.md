# DFIR-LinuxAnalysis
Production Linux system infected with Malware - I had to identify, contain, remediate the system after infection.

A production Linux system was recently infected due to a specific vulnerability we later identified as a configuration issue.

The system is an Asterisk-based phone system that was being utilized to commit toll fraud.

My first step was to identify the issue, which was very easily caught by our carrier notifying us immediately of suspicious calling activity. I then traversed to the phone system and checked the running logs of the system. I could see here that there are is a high amount of calls utilizing trunks I am not familiar with in this system.

After checking the active trunks on the system I couldn't see this malicious trunk in the gui. After running commands, I was able to see that this trunk was active, and clearly hidden from the GUI. 

I started looking through SIP and PJSIP configuration and couldn't find any configuration for it, nor any include statements linking extra files. I then listed the asterisk directory to find a custom config file creating and referncing this trunk.

I then removed the configuration, and reloaded the asterisk configuration. After looking at the logfiles again, calls fraudulent calls were no longer occurring, as the trunk they were utilizing no longer existed.

This is a win, as we've contained the actual intent of the attack, but it does not change the fact the the system is still infected, and will likely soon reinstate the malicious trunk.

I first looked into the users on the system, and identified an unfamiliar user called Juba. After further inspection it has tied itself to the UID 0, along with root. This seems to be an obfuscation technique, and likely more. I used ps aux to check the running process on the system.

I could now see a suspcious process running in /usr/bin/cleanup.

I check the file type, to find it was a PHP script.

I cat'd the file to find obfuscated PHP. 

Using the stat command I could see that this file was created last year, meaning the malware had established a connection for months now.

I also wanted to check for any malicious cron jobs, but there were none.


At this point I needed to make an assessment:
Is the time I would spend investigating and removing this malware worth the time it would take to restore from backup. Furthermore, if restoring from backup was the better choice, how would I circumvent the fact that the malware had been present since at least 6 months?

The answer to the first question is restoring from backups is a more time and cost effective recovery technique. This will also allow me to keep the active system online and functioning during restore time, as the system can currently maintain its use without having any further negative affects.

The answer to the second question is that I can quickly spin up a system from scratch/template, and restore only module configuration. These modules are separate from the actual operating system, and only hold data pertaining to their specific function.

I can pull a recent backup, exclude any modules affected by the vulnerability and exploit, and rebuild those modules from scratch. 
That is exactly what I did. I restore the system on a private-side VM - meaning it was not yet exposed to the public internet. In this state, I reconfigure the modules safely and test to confirm the exploit was no longer an option. This also allowed me to configure the system without taking the active system offline.

Once everything was ready and tested I pulled the old system offline and brought the new one onto the public VM interface with the same IP. I could see attempts using the same exploit that were no longer successful as I had remediately the vulnerability. 

Post-incident analysis - 
After finding and addressing the vulnerability, I deployed an eyes-on check of all similar systems and confirmed that the vulnerability existed nowhere else. I reviewed with the team what had happened they may all be aware of this going forward.




