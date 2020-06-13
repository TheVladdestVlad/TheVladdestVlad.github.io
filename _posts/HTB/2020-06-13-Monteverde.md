---
permalink: /posts/HTB/Monteverde
title: "HTB Monteverde Write-up"
header-img: "/assets/images/HTB/Monteverde/InfoCard.png"
header:
  teaser: "/assets/images/HTB/Monteverde/InfoCard.png"
description: "Hackthebox's Monteverde is a medium-rated Windows machine acting as the on-prem DC for MEGABANK and hosting a SQL Server instance used by the ADSync service"
categories: 
  - CTF
date: 2020-06-13 19:40
tags: HTB Windows ADSync SQLServer
---
# Hackthebox - Monteverde - 10.10.10.172

## Summary
Today, [Hackthebox](https://www.hackthebox.eu) retired Monteverde, a medium-rated Windows machine acting as the on-prem DC for MEGABANK and hosting a SQL Server instance used by the ADSync service.
Here's my write-up.

![InfoCard](/assets/images/HTB/Monteverde/InfoCard.png)

## Enumeration

Starting enumeration off with an initial nmap scan.
```
nmap -sV -sT -sC -p- -o nmapinitial -T4 10.10.10.172
```

![nmapinitial](/assets/images/HTB/Monteverde/nmapinitial.png)

It looks like the machine's name is MONTEVERDE and, judging by the fact that it's listening on port 389, it also acts as the Domain Controller for the MEGABANK.LOCAL0 domain.

I decide to also run enum4linux and see if I can get any useful user information since my nmap scan didn't return any possible entry points.

```
enum4linux -o -U -G -S -P 10.10.10.172 > enum4linux.txt
```

The results of the enum4linux scan offer some pretty useful results.
The local users, one of which is named ADD, which points to [Azure Active Directory Sync](https://docs.microsoft.com/en-us/azure/active-directory/hybrid/concept-adsync-service-account) being configured for this Domain.

![enum4linux_users](/assets/images/HTB/Monteverde/enum4linux_users.png)

Another piece of useful information is the fact that password complexity requirements are disabled and the minimum password length is relatively short - 7 characters.

![enum4linux_passpolicy](/assets/images/HTB/Monteverde/enum4linux_passpolicy.png)

The local groups confirm my initial AADSync suspicion.

![enum4linux_localgroups](/assets/images/HTB/Monteverde/enum4linux_localgroups.png)

## Initial foothold and user flag

Seeing as there isn't any obvious way in that I could identify, I decide to try and brute-force my way in, even if this isn't the usual MO for HTB machines.

I have saved the user names in a txt file and also saved the top most common weak passwords in another txt file.

![users_commonpass](/assets/images/HTB/Monteverde/users_commonpass.png)

I fire up msfconsole and set it up for smb_login brute-forcing.

![msfconsole_setup](/assets/images/HTB/Monteverde/msfconsole_setup.png)

Run it, and, after a few seconds, I get a match with user SABatchJobs having its own user name set as the password.
![msfconsole_success](/assets/images/HTB/Monteverde/msfconsole_success.png)

I can't connect via Evil-WinRM with this user account, but I can access the network file shares.

![shareslist](/assets/images/HTB/Monteverde/shareslist.png)

And I can access the users$ network share.

![sharesusers](/assets/images/HTB/Monteverde/sharesusers.png)

Since enum4linux also pointed out that mhope is a member of the Azure Admins group.

![AzureADM_mhope](/assets/images/HTB/Monteverde/AzureADM_mhope.png)

I decide to go straight to his user directory and see if i can find anything.
mhope's directory does contain something interesting, an azure.xml file, which I copy to my machine

![copy_azurexml](/assets/images/HTB/Monteverde/copy_azurexml.png)

Where I proceed to open it. 

![azurexml](/assets/images/HTB/Monteverde/azurexml.png)

It looks like I might have just found mhope's password and I'm able to use it to connect to the target via Evil-WinRM.

![mhope_evilwinrm](/assets/images/HTB/Monteverde/mhope_evilwinrm.png)

And here is the user flag.

![UserFlag2](/assets/images/HTB/Monteverde/UserFlag2.png)


## Privilege escalation and root flag

Checking the running porcesses using the _ps_ command, I see that the target might also hosts the SQL Server instance used by Azure ADSync

![processes](/assets/images/HTB/Monteverde/processes.png)

To make sure, I need to dig a bit more.

First, I need to find out the name of the instance so that I can attempt to connect to it.
I do that by querying the registry for the name of the installed instance.

```
(get-itemproperty 'HKLM:\SOFTWARE\Microsoft\Microsoft SQL Server').InstalledInstances
```

It looks like it's a default instance (no specific instance name/ID provided during the installation).

![MSSQLSERVER](/assets/images/HTB/Monteverde/MSSQLSERVER.png)

Since this is a default instance, I don't have to specify the instance name when querying it, and, chances are that mhope has access to it via the Azure Admins group.

Now all I have to do is check the databases on the instance to confirm that it is hosting the ADSync database.

```
Invoke-Sqlcmd -Query "SELECT name from sys.databases;"
```

![databases](/assets/images/HTB/Monteverde/databases.png)

And the ADSync is indeed hosted here, which means that [this POC](https://blog.xpnsec.com/azuread-connect-for-redteam/) can be used to get credentials for a user/service account with elevated privileges.

Now, to get the script from the POC to work with this environment the connection string needs to be changed to match this instance.
From this

```
$client = new-object System.Data.SqlClient.SqlConnection -ArgumentList "Data Source=(localdb)\.\ADSync;Initial Catalog=ADSync"
```

To this

```
$client = new-object System.Data.SqlClient.SqlConnection -ArgumentList "Server=MONTEVERDE;Database=ADSync;Trusted_Connection=True;"
```

>For more info on SQL Server connection strings you can check out [this site](https://www.connectionstrings.com/sql-server/).

Time to set up an HTTP server to serve the script.

```
python -m SimpleHTTPServer 80 
```

![httpserv](/assets/images/HTB/Monteverde/httpserv.png)

And download the script in C:\Users\mhope\Documents\test\.

![download](/assets/images/HTB/Monteverde/download.png)

Run it and get the admin credentials.

![DA](/assets/images/HTB/Monteverde/DA.png)

Now all that's left to do is connect to the target using the newly obtained admin credentials and get the root flag.

![RootFlag](/assets/images/HTB/Monteverde/RootFlag.png)

Do some cleanup by removing the script and the test directory, and that's it.
