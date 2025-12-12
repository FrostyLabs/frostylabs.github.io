---
title: Deploying Let's Encrypt Certificates to Proxmox VE
author: frosty
date: 2025-12-12 12:00:00 +0000
categories: [homelab]
tags: [proxmox, ssl, npm, automation]
---

I've been using Nginx Proxy Manager to handle SSL certificates for my Docker containers, which works great. NPM grabs certificates from Let's Encrypt via DNS challenge, and everything just works. But Proxmox was still showing browser warnings because of its self-signed certificate.

## The Issue

Unlike (most of my) Docker containers that expose HTTP and let NPM handle the SSL termination, Proxmox runs its own HTTPS server. You can't just point NPM at it and call it a day. You need to actually deploy the certificate files to Proxmox itself.

## The Solution

I wrote a Python script that pulls certificates from NPM's Docker volume (on my Synology) and deploys them to Proxmox via SSH. It handles the backup, permission setting, and service restart automatically.

The setup is pretty straightforward:
- NPM manages the Let's Encrypt certificate with DNS-01 challenge
- Script runs on Proxmox, pulls certs via SCP
- Deploys to `/etc/pve/local/` with proper permissions
- Restarts the pveproxy service

I run it manually whenever NPM renews the certificate (every 60-90 days), though you could automate it with cron if you wanted.

## A Few Gotchas

There were two issues I ran into:

**SCP subsystem failure**: Newer OpenSSH versions use SFTP protocol by default, but Synology's SSH doesn't play nice with it. Had to add the `-O` flag to force legacy SCP protocol.

**FUSE filesystem**: `/etc/pve` is a special cluster filesystem in Proxmox. Using `shutil.copy2()` failed because it tries to preserve file metadata. Had to read and write files directly instead.

## Result

Now Proxmox shows a valid Let's Encrypt certificate, no more browser warnings. The script is on [GitHub](https://github.com/FrostyLabs/pve-npm-certs) if you're dealing with the same thing.
