---
title: "HackTheBox: Bitlab"
date: "2020-01-11"
author: frosty
categories:
  - "writeups"
tags:
  - "hackthebox"
---

![Image](assets/img/writeups/hackthebox/bitlab/bitlab.png)

* * *

> _I would like to preface this post by saying that the privilege escalation is through an unintended method._

## Host Enumeration

As usual, we begin with a full port scan in order to discover open ports. In this case, we see that there are 2 ports open:

- 22: OpenSSH 7.6p1 Ubuntu
- 80: nginx

```sh
nmap -p- -sV 10.10.10.114
```

![Image](assets/img/writeups/hackthebox/bitlab/image-20.png)

### Enumeration - HTTP

The default page when navigating to http://10.10.10.114/ is a redirect to `/users/sign_in`. We see that Bitlab is hosting GitLab community edition. My GoBuster scan was failing to run, so I performed some manual enumeration. Firstly, I view the `/robots.txt` page, there is usually some interesting information in these files when they are present.

![Image](assets/img/writeups/hackthebox/bitlab/image-21-1024x456.png)

![Image](assets/img/writeups/hackthebox/bitlab/image-22.png)

**/profile** - On this page we see a profile `Clave`; a web developer engaged in the development of WWW applications.

![Image](assets/img/writeups/hackthebox/bitlab/image-23.png)

**/help** - Navigating to this page reveals a folder including a backed up bookmarks file named `bookmarks.html`.

![Image](assets/img/writeups/hackthebox/bitlab/image-24.png)

Opening the bookmarks.html, we see an interesting bookmark `Gitlab Login`, which appears to be a JavaScript function. This can be seen when hovering the mouse over the hyperlink. We obtain the JavaScript function by viewing the page source code. We see that there is hexadecimal encoded text. I'll use the [ascii2hex](https://www.asciitohex.com/) website to decode the hexadecimal. I inserted `\x0a` (linebreak) to make the decoding process for every variable easier to read; seen in the image below. We see some potential user credentials; `clave:11********`. We are denied SSH login using these credentials, however we are able to login to the Gitlab application.

![Image](assets/img/writeups/hackthebox/bitlab/image-25.png)

![Image](assets/img/writeups/hackthebox/bitlab/image-26.png)

![Image](assets/img/writeups/hackthebox/bitlab/image-28.png)

### Enumeration - Gitlab

Immediately when logging in, we see two repositories; `Profile` and `Deployer`. When viewing the `Profile` repository, it appears that it is code being hosted on the `/profile` endpoint which we viewed in the HTTP enumeration stage earlier. Interesting ;). In addition, we see that Clave is labelled as a developer for this repository.

My idea to get the initial shell through authenticated code execution. The aim is to upload a php reverse shell file into the profile repository, with the aim that it will be executed when viewing it. Let us try it!

![Image](assets/img/writeups/hackthebox/bitlab/image-29.png)

## Getting the initial shell

I use the generic pentestmonkey php reverse shell file, and setup my netcat listener. Upload the php file to the repository, and create a merge request. Remember, as we have developer permissions for this repository, we are allowed to approve our own merge request.

![Image](assets/img/writeups/hackthebox/bitlab/image-30.png)

![Image](assets/img/writeups/hackthebox/bitlab/image-31.png)

![Image](assets/img/writeups/hackthebox/bitlab/image-32.png)

## Privilege Escalation

> _This is where the unintended route begins. I questioned for a while whether I would publish this post, however I believe that there is still value in writing this. It is not a simple kernel exploit or misconfigured permission of user.txt, but rather a method which bypasses the requirement to pivot to the Clave user on the host._

When performing `sudo -l` in the www-data shell, we can see that we are able to run `git pull` as root user without a password requirement.

![Image](assets/img/writeups/hackthebox/bitlab/image-33.png)

We will exploit the sudo command misconfiguration, and using it to our advantage by the use of [Git Hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks). Git hooks are used to perform automated tasks after _something_ has occurred. For example, a notification to select persons when a merge request has been approved, or something along those lines. We will generate a reverse shell by using the `post-merge` hook, which will be triggered after the`git pull` command. The one pre-requisite for this exploit to be successful is that there must be some change in the repository through the web application before we pull the repository. This will ensure that the `post-merge` hook will be invoked.

The challenge that we face is the "official" profile repository is owned by the root user, and therefore the www-data user will not be able to edit the git hook. To overcome this, we can copy the profile folder to the /tmp directory.

![Image](assets/img/writeups/hackthebox/bitlab/image-35.png)

![Image](assets/img/writeups/hackthebox/bitlab/image-36.png)

![Image](assets/img/writeups/hackthebox/bitlab/image-38.png)
