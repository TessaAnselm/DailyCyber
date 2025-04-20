Photographer: (VulnHub)
A beginner-friendly box with multiple attack surfaces — including web, SMB, and privilege escalation via misconfigured SUID binaries. Let’s walk through the exploitation path.

Vulnerable Machine: https://vulnhub.com/entry/photographer-1,519/

** Initial Enumeration**
1.  Nmap Scan

 Look for Photograph Machine IP (target) using subnet or running ping sweep internal ip.
![2c1ab68d70015c8e2e89cb0c1a14fb40.png](_resources/2c1ab68d70015c8e2e89cb0c1a14fb40.png)

![9f868d1ebcf8d057dfa1ff4e3c8cd18c.png](_resources/9f868d1ebcf8d057dfa1ff4e3c8cd18c.png)

From the results, Ports 80, 8000, and SMB appear interesting.
![b46c545a90e4f72629a333c308a8cfc4.png](_resources/b46c545a90e4f72629a333c308a8cfc4.png)

2. Web enumeration
Port 80: Nothing interesting when viewed in a browser.
Port 8000: Shows an upload interface
![028f469c07b30f58c4a63c7b33e430aa.png](_resources/028f469c07b30f58c4a63c7b33e430aa.png)

Directory Bruteforce
We used gobuster to search for hidden paths:
![db6beb2fed42f3ce960757d707817221.png](_resources/db6beb2fed42f3ce960757d707817221.png)

You can also try dirb as an alternative:
`dirb http://example.com -N 302`

Eventually, we discovered the /admin directory:
![d233be8fe781cf525961f778d7a29d4c.png](_resources/d233be8fe781cf525961f778d7a29d4c.png)

3. SMB Enumeration
We scan using enum4linux:
`enum4linux-ng 172.16.1.132`

Findings:
Weak or misconfigured SMB security
Accessible shares
Anonymous access allowed
Poor password and lockout policies

This is low-hanging fruit — anyone can connect and access files:
![f0028a92ba3b7f9ac6e1bcda3dbfb693.png](_resources/f0028a92ba3b7f9ac6e1bcda3dbfb693.png)

We also found a very weak password policy:
![236f84ac07bbd55359b1bf0f5ccbb608.png](_resources/236f84ac07bbd55359b1bf0f5ccbb608.png)

**Initial Access**
1. File Access via SMB
We were able to connect and retrieve files using guest account:
`smbclient \\\\172.16.1.132\\sambashare --user=guest`

![978cdcf101a8023d7f6cbbe22a2370c1.png](_resources/978cdcf101a8023d7f6cbbe22a2370c1.png)

From the files retrieved, we discovered a user credentials:
daisa@photographer.com
password: babygirl
![4e75ffca9117d027fcfcf62a60c3c6d2.png](_resources/4e75ffca9117d027fcfcf62a60c3c6d2.png)

`Lateral Movement`
1. Searchsploit
A previous Nmap scan likely revealed that the target is running Koken CMS, and we verified that there's a known exploit:
![afe0e7cc277adee8ccaed3d8438aec59.png](_resources/afe0e7cc277adee8ccaed3d8438aec59.png)

We found existing Koken vulnerability that we can exploit
![ef4c837df7fe2dc18c1dd9e5a53e9914.png](_resources/ef4c837df7fe2dc18c1dd9e5a53e9914.png)

`searchsploit -x php/webapps/48706.txt`
![6fd40f1a6aadd13d8a9ac10071876cab.png](_resources/6fd40f1a6aadd13d8a9ac10071876cab.png)

2. Exploitation via BurpSuite
Using the /admin login, we intercepted the upload request with BurpSuite, removed the .jpg extension, and uploaded a PHP reverse shell.

![26aab2434430fdae7fa4cc84cb1ebdcb.png](_resources/26aab2434430fdae7fa4cc84cb1ebdcb.png)

Since file upload was possible, we uploaded a PHP reverse shell:
We then set up a Netcat listener:
`nc -lvnp 1234`

`cp /usr/share/webshells/php/php-reverse-shell.php .`
![5051898ed7360996c02f5102e04e7a42.png](_resources/5051898ed7360996c02f5102e04e7a42.png)

Once the shell connected, we had limited shell access. After gaining shell access, we were able to retrieve the first flag:
![4b5f74f7cee71b2596d8d58ffc1d113b.png](_resources/4b5f74f7cee71b2596d8d58ffc1d113b.png)

**Privilege Escalation**
1. SUID Binary Discovery
Run the following to look for SUID binaries with root privileges:

`find / -user root -perm -4000 -print 2>/dev/null | xargs ls -lh`
Look for non default root access
![572090a37aed7e2078e0073fc67da882.png](_resources/572090a37aed7e2078e0073fc67da882.png)

We discovered a suspicious binary:
usr/bin/php7.2 that should not have root. 

We used GTFOBins to escalate privileges through PHP:
https://gtfobins.github.io/gtfobins/php/

![34884f519adf8a07c5a7a6c39bf92634.png](_resources/34884f519adf8a07c5a7a6c39bf92634.png)

2. Root Shell
  
We were able to become root
`usr/bin/php7.2 -r ./php -r "pcntl_exec('/bin/sh', ['-p']);"`
![879b5cdbf8a92ffef710ed707db8ab23.png](_resources/879b5cdbf8a92ffef710ed707db8ab23.png)

We found the final flag in the /root directory:
![70f865115dc9fb66739d299e74494869.png](_resources/70f865115dc9fb66739d299e74494869.png)

Big thanks to David P. and Rob for doing this live with CCSF students. Their walkthrough can be found here:https://www.rootandbeer.com/photographer-1-vulnhub-walkthrough/

Lessons Learned
SMB enumeration can reveal credentials if anonymous access is allowed.
Directory fuzzing can uncover admin panels or hidden uploads.
Weak password policies combined with known CMS exploits = easy access.
Misconfigured SUID binaries can often lead to full root.