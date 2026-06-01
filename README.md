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

Using the stat command I could see that this file was created last year, meaning this vulnerability had been

I also wanted to check for any malicious cron jobs, but there were none.

A
