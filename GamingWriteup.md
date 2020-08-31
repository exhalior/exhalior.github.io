---
permalink: /GamingWriteup.html
---

## Tryhackme Gaming Server Writeup.

This writeup is on the [Tryhackme GamingServer](https://tryhackme.com/room/gamingserver) machine.
It is an easy level machine, and is described as: > An Easy Boot2Root box for beginners

This is how I got both flags.

## Enumerating.

First, I started off with a simple nmap scan.

```
nmap -sC -sV -v -oA gamingserver *machine_ip*
```

This showed me that there were two ports open on the machine, port 22 for *SSH* and port 80 for *HTTP*

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 34:0e:fe:06:12:67:3e:a4:eb:ab:7a:c4:81:6d:fe:a9 (RSA)
|   256 49:61:1e:f4:52:6e:7b:29:98:db:30:2d:16:ed:f4:8b (ECDSA)
|_  256 b8:60:c4:5b:b7:b2:d0:23:a0:c7:56:59:5c:63:1e:c4 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: House of danak
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Since there was a webpage, I directed my attention there first. 
Simply checking the source of the websites index page showed a comment at the bottom referring to a user named *john*
I ran hydra in the background to attempt to bruteforce his password, but nothing came up.

```
hydra -l john -P *path to wordlist* *machine_ip* ssh
```

Running gobuster using the common.txt wordlist from dirb revealed a /secret directory and an /uploads directory.
The /secret directory seemed out of place, so I checked there first.

<img src=https://i.imgur.com/FYrdjNd.png/>

In the /secret index there was a single file named *"SecretKey"* and opening it revealed a private ssh key.
I copied it to my folder and gave it the right permissions.

```
chmod 600 SecretKey
```
Under the folder /uploads there were a few files and a wordlist which I assumed would be for bruteforcing something. So I saved it to my machine.

Simply ssh'ing into the machine under *John* prompted us with a password for the ssh key. No problem.
John the ripper should be good enough for this job!.
First i used ssh2john to format the key into something john the ripper can read.

```
/usr/share/john/ssh2john.py SecretKey > keyhash
```
Next, I ran the wordlist I found earlier in the uploads directory on the hash.

```
john keyhash -w=dict.lst
```

