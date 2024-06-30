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

![Image](assets/img/blog/github-ssh-keys/1-generate-keys.png)

# 2. Imprt the SSH Key to GitHub

Go to your icon on the top right, Settings, then SSH and GPG keys. Click New SSH Key on the top right. Then copy and paste your public key into the text box, like in the screenshots below.

![Image](assets/img/blog/github-ssh-keys/2-import-keys.png)

![Image](assets/img/blog/github-ssh-keys/3-import-keys.png)

Import public key

# 3. Configure Linux with the Key

Two simple commands! The agent PID is likely going to be different for you. Also you need to make sure that you use the correct path to your private key. If you configured a passphrase for your SSH key, you need to type it in here.

```bash
$ eval `ssh-agent -s`
# Agent PID ####
$ ssh-add /home/user/.ssh/id_ed_25519
```

![Image](assets/img/blog/github-ssh-keys/4-configure-linux.png)

## Optional: Persistent SSH Keys

If you would like your configured SSH keys to remain persistent, such as after system reboot, then you should execute the commands listed below. I would like to note that this is for [ZSH](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins/ssh-agent), it could be different for the Bash shell. Thanks to [@UnRunDead](https://unrundead.dev/) for this tip!

Edit your .zshrc and go to plugins=(...) and add ssh-agent

IMPORTANT: put these settings before the line that sources oh-my-zsh

zstyle :omz:plugins:ssh-agent identities id_rsa id_rsa2 id_github

Then reload your terminal: source ~/.zshrc

And that’s all it takes! Now you can test it out 🙂

# Test it out

```bash
git config --global user.name YourUserName
mkdir project-folder
cd project-folder
git init
echo "# Hello World" > README.md
git add README.md
git commit -m "Frosty Labs Demonstration"
git remote add origin git@github.com:YourUserName/project-folder.git
git push -u origin master
```

![Image](assets/img/blog/github-ssh-keys/5-test.png)

![Image](assets/img/blog/github-ssh-keys/6-verification.png)
