---
title: Configure Github SSH Keys
author: frosty
date: 2020-02-07 12:00:00 +0000
categories: [blog]
tags: [blog]     # TAG names should always be lowercase
---

This is a quick post on how to configure GitHub SSH keys. The GitHub documentation is pretty good with this, so this post is more of a personal note.

The advantage of using GitHub SSH keys is that you do not need to provide the username and password of your GitHub account each time. This makes it easier to work with GitHub, more specifically private repositories in my opinion.

To add the SSH keys you need to

1. Create SSH keys
2. Import the SSH key to GitHub
3. Save SSH key in Linux

# 1. Create SSH Keys

Here we have the option of using RSA 4096 bit keys, or we can use Eliptic Curve 25519 algorithm. The EC-25519 curve is the one that is used in the famous Eliptic Curve Diffie-Hellman key exchange algorithm.

```bash
# EC-25519
ssh-keygen -t ed25519 -C "your_email@example.com"
# RSA 4096
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

![Image](assets\img\blog\github-ssh-keys\1-generate-keys.png)
