---
title: "HackTheBox: Obscurity"
date: "2020-05-09"
author: frosty
categories:
  - "writeups"
tags:
  - "hackthebox"
---

![Image](assets/img/writeups/hackthebox/obscurity/htb-obscurity.png)

* * *

## Host Enumeration

As usual, we begin with an nmap scan to identify listening services.

- 22: OpenSSH 7.6p1
- 80: Closed HTTP
- 8080: BadHTTPServer

![Image](assets/img/writeups/hackthebox/obscurity/image-10.png)

## Host enumeration - 8080

Interestingly, the version of this service is BadHTTPServer... I am unfamiliar with this service. Due to the nature of HTTP, we navigate to it using the browser and we see a note; their motto is _security through obscurity!_ Interesting for sure. In my secure software development class at university, we were taught that security by obscurity offers no security, and it's only a matter of time until an attacker finds out.

When scrolling down a little, there is a second note from the developers. The web server code is in the **SuperSecureServer.py** file, in the secret development directory. This is where security by obscurity comes into action. The secret development directory is only supposed to be known to those authorized, however we are hackers and can use enumeration tools ;)

![Image](assets/img/writeups/hackthebox/obscurity/image-11-1024x187.png)

![Image](assets/img/writeups/hackthebox/obscurity/image-12.png)

Lets use wfuzz to try and find the directory which SuperSecureServer.py is hidden in. The command can be seen below. The `FUZZ` is the placeholder/ field that is fuzzed by the wordlist. We place `/SuperSecureServer.py` afterwards so that when the correct directory is placed in the field, then we should get a HTTP 200 OK response.

```sh
$ wfuzz -w '/usr/share/wordlists/dirb/common.txt' \
-u 'http://10.10.10.168:8080/FUZZ/SuperSecureServer.py' \
--hc 404,500 -c
```

![Image](assets/img/writeups/hackthebox/obscurity/image-13.png)

Bingo! The secret directory is `develop`. Not so secret after all. Alright, now we have access to the software running the web server. Thankfully Python is an interpreted code language, and does not always get compiled. Hence, we are able to read the source code without any binary reversing.

* * *
