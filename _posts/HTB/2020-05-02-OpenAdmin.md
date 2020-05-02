---
permalink: /posts/HTB/OpenAdmin
title: "HTB OpenAdmin Write-up"
header-img: "/assets/images/HTB/OpenAdmin/InfoCard.jpg"
header:
  teaser: "/assets/images/HTB/OpenAdmin/InfoCard.jpg"
description: "Hackthebox's OpenAdmin is an easy-rated Linux box hosting a few websites and using OpenNetAdmin."
categories: 
  - CTF
date: 2020-05-02 19:40
tags: HTB Linux OpenNetAdmin RCE
---
# Hackthebox - OpenAdmin - 10.10.10.171

## Summary
Today, [Hackthebox](https://www.hackthebox.eu) retired OpenAdmin, an easy-rated Linux box hosting a few websites and using OpenNetAdmin.

![InfoCard](/assets/images/HTB/OpenAdmin/InfoCard.jpg)

>*Note that the screenshots are taken today (2020-05-02) because I didn't do a proper write-up during my first run on the box.*

## Enumeration

Starting enumeration off with an initial nmap scan.
```
nmap -sV -sT -sC -p- -o nmapinitial -T4 10.10.10.171
```
Nmap doesn't find anything out of the ordinary, the usual ssh port and a web service on port 80

![nmapinitial](/assets/images/HTB/OpenAdmin/nmapinitial.jpg)

Looks like it's a newly set up Apache server.

![apachepage](/assets/images/HTB/OpenAdmin/apachepage.jpg)

Time to use gobuster to see what other pages/directories this web server might be hosting.
```
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -u http://10.10.10.171/ -t 100 -x .txt,.html,.php,.xml --timeout 18s -q -o gobuster_out.txt
```
gobuster is able to find 4 directories.

![gobustermain](/assets/images/HTB/OpenAdmin/gobustermain.jpg)

While I run gobuster again, this time on the sierra directory
```
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -u http://10.10.10.171/sierra -t 100 -x .txt,.html,.php,.xml --timeout 18s -q
```
I also start manually navigating through http://10.10.10.171/music

![musicmain](/assets/images/HTB/OpenAdmin/musicmain.jpg)

Pressing on the Login link, takes me to http://10.10.10.171/ona

![ona](/assets/images/HTB/OpenAdmin/ona.jpg)

Which stands for [OpenNetAdmin](https://opennetadmin.com/about.html) - a tool used to track IP network attributes.

Based on this, and the message about the version on the target server being out of date, I start searching online for OpenNetAdmin 18.1.1 vulnerabilities, and I find [this RCE vulnerability exploit](https://github.com/amriunix/ona-rce/blob/master/ona-rce.py).
I download the Python script to my VM using curl.
```
curl https://raw.githubusercontent.com/amriunix/ona-rce/master/ona-rce.py >ona-rce.py
```

## Initial Foothold 

I execute the script and am able to get a basic shell as www-data
![ona-rce](/assets/images/HTB/OpenAdmin/ona-rce.jpg)

Navigating through www-data's directories, I find the database_settings.inc.php file

![dbsettingsphp](/assets/images/HTB/OpenAdmin/dbsettingsphp.jpg)

Which contains a password used for connecting to the MySQL database hosted on the target server.

Now I'm interested in trying to ssh into the target server using this password and one of the users found in /home
![users](/assets/images/HTB/OpenAdmin/users.jpg)

The password doesn't work with Joanna, but it works with Jimmy's user.
![jimmyssh](/assets/images/HTB/OpenAdmin/jimmyssh.jpg)

Navigating through the Apache related directories, I find the following config file

![internalconf](/assets/images/HTB/OpenAdmin/internalconf.jpg)

And it looks like /var/www/internal/main.php is able to do something interesting.

![mainphp](/assets/images/HTB/OpenAdmin/mainphp.jpg)

Calling main.php using curl prints Joanna's rsa key

![rsa_key_joanna](/assets/images/HTB/OpenAdmin/rsa_key_joanna.jpg)


## User flag

I save the contents of Joanna's rsa key as joanna_rsa on my VM.
```
python ~/Scripts/John/ssh2john.py joanna_rsa > jrsa_hash.txt
```

And use ssh2john.py to format it for cracking with john.
```
john --wordlist=/usr/share/wordlists/rockyou.txt jrsa_hash.txt
```

![joannapass](/assets/images/HTB/OpenAdmin/joannapass.jpg)

I do ```chmod 400 joanna_rsa``` so that i can use the key with ssh, and proceed to connect via ssh as joanna using her key and the newly obtained password.

![Joannassh](/assets/images/HTB/OpenAdmin/Joannassh.jpg)

And I can now read the user.txt file.

![userflag](/assets/images/HTB/OpenAdmin/userflag.jpg)

## Privilege escalation and root flag

Running ```sudo -l``` shows that Joanna is able to call nano using sudo

![sudo-l](/assets/images/HTB/OpenAdmin/sudo-l.jpg)

This is great, since [nano](https://gtfobins.github.io/gtfobins/nano/) can be used to break out of restrictied environments and can be used to spawn a system shell on the machine.

So I run
```
sudo /bin/nano /opt/priv
```
In nano, I can press ^R (ctrl+r) followed by ^X (ctrl+x) to get a prompt that lets me type a command to execute.
Executing ```reset; sh 1>&0 2>&0``` will spawn a shell as root which I can then use to reat the root.txt file.

![root](/assets/images/HTB/OpenAdmin/root.jpg)
