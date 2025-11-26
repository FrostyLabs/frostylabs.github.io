---
title: Building My First AD Lab With Infrastructure as Code
author: frosty
date: 2025-11-26 12:00:00 +0000
categories:
- development
- blog
tags:
- active-directory
- vagrant
- ansible
- automation
- IaC
- lab
---

Building test environments has always been a pain point for security researchers and IT professionals. Manually spinning up virtual machines, configuring Active Directory, joining domain members, and setting up services can easily consume hours or even days. When I decided to build a proper Active Directory lab environment for security testing and research, I knew there had to be a better way.

This is when I discovered Infrastructure as Code (IaC), and specifically Vagrant and Ansible. This was my first real dive into the world of IaC, and I have to say – I'm never going back to manual VM configuration again. Well maybe not quite, but this removes a lot of tedious work for sure!

Check out the repository, and then let's talk about it: [GitHub Repository for an ad-lab](https://github.com/FrostyLabs/ad-lab)

## Why an AD Lab Matters

Active Directory is everywhere in enterprise environments. If you're doing penetration testing, security research, or just want to understand how enterprise networks operate, you need a sandbox environment to experiment in. Breaking things in production is not an option, and repeatedly rebuilding test environments manually is tedious and error-prone.

The goal was simple: create a fully functional Active Directory environment that I could spin up with a single command, break things, and then destroy and rebuild just as easily. A true playground for learning and testing.

## First Steps with Vagrant

Vagrant became my starting point. The concept is brilliant – define your virtual machines in code, and Vagrant handles the rest. My `Vagrantfile` defines five machines:

- **win-dc01**: A Windows Server 2019 domain controller
- **win-srv01**: A member server with SQL Server installed
- **win-srv02**: Another member server with file services
- **win-client01**: A Windows 10 workstation
- **lin-srv01**: An Ubuntu server with a LAMP stack, integrated with Active Directory

Each VM gets its own static IP address, port forwarding for RDP/SSH access, and a shared folder for scripts and utilities. The beauty of this approach? I can type `vagrant up` and walk away while Vagrant provisions all five machines automatically.

Learning Vagrant's configuration syntax took some time, especially dealing with Windows-specific quirks like WinRM configuration and boot timeouts. But once I got the hang of it, adding new VMs or modifying existing ones became trivial.

## Ansible: Where the Magic Happens

Vagrant handles the VM provisioning, but Ansible is where the real configuration magic happens. This was my first experience with Ansible, and the learning curve was steeper than I expected – but absolutely worth it.

Ansible uses "playbooks" written in YAML to define what configuration should be applied to each system. I created separate playbooks for each machine:

- The domain controller playbook creates the `frostylabs.local` domain, configures DNS, and creates domain users and security groups
- The member server playbooks join the domain, install roles like File Server and IIS, and create network shares
- The Windows client playbook joins the domain and configures user permissions
- The Linux server playbook installs SSSD for AD integration and sets up a complete LAMP stack

The power of Ansible lies in its idempotency – I can run the same playbook multiple times, and it will only make changes if the system isn't already in the desired state. This makes the playbooks both safe and reliable.

## The Role Structure

One of Ansible's best practices is organizing tasks into "roles" – reusable components that can be shared across playbooks. I created several roles:

- **common**: Base Windows configuration with Chocolatey and essential tools
- **domain_controller**: Everything needed to promote a server to a domain controller
- **member_server**: Domain joining and basic server configuration
- **linux_server_basic**: Base Linux configuration and networking
- **linux_server_domain**: SSSD configuration for Active Directory integration
- **lamp_server**: Apache, MySQL, and PHP installation and configuration

This modular approach means I can easily extend the lab. Want to add another member server? Just apply the `common` and `member_server` roles. Need a Linux workstation? Reuse the `linux_server_basic` and `linux_server_domain` roles.

## Challenges and Learning Moments

Not everything worked perfectly on the first try. Some challenges I faced:

**WinRM Authentication**: Getting Ansible to communicate with Windows machines required understanding WinRM, self-signed certificates, and Windows authentication. The `ConfigureRemotingForAnsible.ps1` script became essential for this.

**Timing Issues**: Domain promotion takes time. Member servers can't join a domain until the domain controller is fully ready. I had to build in proper wait conditions and reboot handling.

**Windows Modules**: Ansible's Windows modules are different from Linux modules. Tasks like installing software, modifying the registry, and managing services all require Windows-specific modules from the `ansible.windows` and `community.windows` collections.

**Network Configuration**: VMware's network configuration sometimes conflicted with Vagrant's expectations. I ended up creating PowerShell scripts to ensure proper network configuration on each Windows VM.

## The Result

After working through the learning curve, I now have a fully automated Active Directory lab. The entire process:

```bash
vagrant up                    # Provision all VMs
ansible-playbook labsetup.yml # Configure everything
```

About 30-40 minutes later (depending on your hardware), I have a complete enterprise environment with:
- A functioning Active Directory domain
- DNS services
- Multiple Windows servers and a workstation
- SQL Server
- A Linux server integrated with Active Directory
- User accounts, security groups, and network shares

When I'm done testing? A simple `vagrant destroy -f` tears it all down. No leftover VMs cluttering my system, no manual cleanup required.

## Why This Matters

Infrastructure as Code transformed how I approach lab environments. The benefits are huge:

**Reproducibility**: The lab is identical every time I build it. No more "it worked last time" mysteries.

**Documentation**: The Vagrantfile and Ansible playbooks serve as living documentation of exactly how the environment is configured.

**Version Control**: Everything is text files that live in Git. I can track changes, roll back mistakes, and branch for experiments.

**Sharing**: I can share this lab with colleagues or the security community. They get the exact same environment I use.

**Learning**: The process of building this taught me both how Active Directory works and how to automate infrastructure.

## Looking Forward

This was my first serious project with IaC, Vagrant, and Ansible, but it won't be my last. I'm already thinking about enhancements:

- Adding more realistic misconfigurations for security testing
- Implementing Group Policy Objects (GPOs)
- Adding monitoring and logging infrastructure
- Creating snapshots at key points for quick rollback
- Expanding to multi-forest scenarios

For anyone doing security research, penetration testing, or just wanting to learn Active Directory – I highly recommend building your own automated lab. The initial time investment pays off quickly, and the skills you learn apply far beyond the lab itself.

## Resources

If you're interested in building something similar:

- **Vagrant Documentation**: [vagrantup.com/docs](https://vagrantup.com/docs)
- **Ansible Documentation**: [docs.ansible.com](https://docs.ansible.com)
- **Ansible Windows Guides**: Essential reading for Windows automation
- **VMware Vagrant Provider**: If you're using VMware like I am

The era of manually clicking through installers is over. Infrastructure as Code is the future, and honestly, it's a lot more fun too.

Happy labbing!
