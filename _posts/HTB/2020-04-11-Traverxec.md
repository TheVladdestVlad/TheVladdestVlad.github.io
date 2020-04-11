---
permalink: /posts/HTB/Traverxec
title: "HTB Traverxec Write-up"
header-img: "/assets/images/HTB/Traverxec/InfoCard.jpg"
header:
  teaser: "/assets/images/HTB/Traverxec/InfoCard.jpg"
description: "Hackthebox's Traverxec is an easy-rated Linux box that uses the nostromo webservice"
categories: 
  - CTF
date: 2020-04-11 18:34
tags: HTB Linux 
---
# Hackthebox - Traverxec - 10.10.10.165

## Summary

Today [Hackthebox](https://www.hackthebox.eu) retired Traverxec, an easy-rated Linux box that uses the nostromo webservice.

![InfoCard](/assets/images/HTB/Traverxec/InfoCard.jpg)

## Enumeration

Starting enumeration off with an initial nmap scan.
```
nmap -sV -sT -sC -p- -o nmapinitial -T4 10.10.10.165
```
Upon compeltion, the nmap scan reveals that the VM is serving a website on port 80 using the nostromo webserver.

![nmapinitial](/assets/images/HTB/Traverxec/nmapinitial.jpg)

Searching on the web, I find that  nostromo is vulnerable to RCE and [this POC](https://www.exploit-db.com/exploits/47837) that takes advantage of it.
I also check in Metasploit to see if an exploit is already available, and it is.

![msfconsolesearch](/assets/images/HTB/Traverxec/msfconsolesearch.jpg)

I configure the exploit and run it.

![msfcons_exploitrun](/assets/images/HTB/Traverxec/msfcons_exploitrun.jpg)

At this point I have an initial foothold on the target, but it does seem that the exploit in Metasploit is a bit limited so I've switched to the Python script posted in the [POC](https://www.exploit-db.com/exploits/47837)

## Enumeration as www-data and privilege escalation to user

Looking through the directories of the target, I find a home directory for a user named david, but I do not have permission to list its contents.

![home](/assets/images/HTB/Traverxec/home.jpg)

But there is a useful hint in the /var/nostromo/conf/nhttpd.conf file

![nhttpdconf](/assets/images/HTB/Traverxec/nhttpdconf.jpg)

If desired, nostromo can access and expose home directories, and it looks like homedir access has been configured in this case, and this offers a possible path to check in - /home/david/public_www, which should be accessible to www-data

![publicwww](/assets/images/HTB/Traverxec/publicwww.jpg)


the protected-file-area directory contains a zip backup of david's ssh keys

![pfa](/assets/images/HTB/Traverxec/pfa.jpg)

All I need to do now is copy the archive to a directory where I have write permissions so that I can extract the contents and read the ssh rsa key.

Since www-data has write permissions in /tmp, I created a /tmp/test directory where I can do the rest of the work.

I copy the zip file to /tmp/test, extract it's contents and then read the resulting id_rsa file, which I then copy and paste into david_rsa on my machine.

![davidrsa](/assets/images/HTB/Traverxec/davidrsa.jpg)

Now, there are 2 things that i need to do in order to be able to ssh into 10.10.10.165 as david using his rsa key.
1. I need to crack the password for the rsa key with john, so i extract the john-friendly hash from the key 
```
python ~/Scripts/John/ssh2john.py david_rsa > david_rsa_hash.txt
``` 
And I then use john to get the password from the resulting hash.

![davidrsapass](/assets/images/HTB/Traverxec/davidrsapass.jpg)

The password is __hunter__

2. I need to change the permissions on the david_rsa file using ```chmod 400 david_rsa``` so that i can use it with ssh

Now, I can ssh as david and get the user flag

![davidssh](/assets/images/HTB/Traverxec/davidssh.jpg)

## Privilege escalation and root flag

Upon further inspection of david's home directory, I find an interesting script located in /home/david/bin/

![script](/assets/images/HTB/Traverxec/script.jpg)

It looks like david can call [journalctl](https://gtfobins.github.io/gtfobins/journalctl/) using sudo.
journalctl uses [less](https://gtfobins.github.io/gtfobins/less/) as a pager, and less sometimes doesn't exit if the terminal isn't wide enough.
So I make my terminal narrower and then run
```
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service
```

less does not exit after listing the last 5 lines of nostromo's service log

![journalctl](/assets/images/HTB/Traverxec/journalctl.jpg)

So now I can try to spawn a shell as root using ```!/bin/sh```

![root](/assets/images/HTB/Traverxec/root.jpg)

And that's pretty much it.
