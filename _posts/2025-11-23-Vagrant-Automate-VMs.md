---
title: "Vagrant: Automate Virtual Machine Creation"
author: frosty
date: 2025-11-23
categories:
- development
- Infrastructure as Code
tags:
- dev
- vmware
- automation
- lab
- IaC
---

Vagrant gives you a simple and repeatable way to build and manage
virtual machines through code. It removes the manual setup work and lets
you recreate whole environments with a single command.

I want to write a short post for myself on how to automate virtual
machine creation for VMWare. There are plenty of resources for
VirtualBox, but not many for VMWare. Although the process is almost identical, 
I thought that this might help someone else.

This is my first real step into infrastructure as code (IaC), and it is
interesting to me. My goal is to build and remove an Active Directory
lab in a clean and repeatable way, to support some future purple team
work.

## Installation

I am using Windows 11. You can install Vagrant from the Hashicorp
website where they always provide the newest release. 
([Download Link](https://developer.hashicorp.com/vagrant/install)).

### Vagrant VMWare Plugin

Hashicorp recently made their VMWare plugin free. Nice change, since it
used to require a license. VirtualBox is free as well, but I still find
myself using VMWare even after the Broadcom acquisition.

Install the plugin with:

```
vagrant plugin install vagrant-vmware-desktop
```

After this, you also need the Vagrant VMWare Utility. The plugin install
output should include the direct download link. A restart is a good idea
once both are installed. 
([VMWare Utility Download Link](https://www.vagrantup.com/docs/providers/vmware/vagrant-vmware-utility))

You can check the setup with:

```
vagrant plugin list
```

## Vagrant Commands

I keep my VMWare machines on a separate drive. Adjust paths to your own
setup.

### Initialising Vagrant
```
PS C:\Users\frosty> E:\
PS E:\> cd "VMWare_Storage\Virtual Machines"
PS E:\VMWare_Storage\Virtual Machines> mkdir vagrant
PS E:\VMWare_Storage\Virtual Machines\vagrant> vagrant init
```

It tells us that a Vagrantfile has been created. 
Looks good so far.

You can find suitable base images on the Vagrant discovery page.
In this example, I will use `svetterIO/UbuntuDesktop22.04`
You can find the box image which you want to use on this 
[discovery page](https://portal.cloud.hashicorp.com/vagrant/discover).

Add some basic content to the Vagrantfile:

``` ruby
Vagrant.configure("2") do |config|

  config.vm.provider "vmware_desktop" do |v|
    v.gui = true
    v.memory = 2048
    v.cpus = 1
  end

  config.vm.define "ubuntu" do |ubuntu|
    ubuntu.vm.box = "svetterIO/UbuntuDesktop22.04"
  end
end
```

That is enough to begin.

### Booting Vagrant Machines

Run `vagrant up` in the folder that contains your Vagrantfile. Keep an
eye on the output in case there are warnings.

```
E:\VMWare_Storage\Virtual Machines\vagrant> vagrant up
Bringing machine 'ubuntu' up with 'vmware_desktop' provider...
```

The Hashicorp documentation is very good, so it is worth reading once
you get comfortable with the basics.

### Cleaning Up Vagrant Machines

Since this was only a test, I removed the machine and the image. 
I want to do more in the future, but with different images. 
More to come on that later I guess ;)

```
PS E:\VMWare_Storage\Virtual Machines\vagrant> vagrant destroy -f
==> ubuntu: Deleting the VM...
PS E:\VMWare_Storage\Virtual Machines\vagrant> vagrant box list
svetterIO/UbuntuDesktop22.04 (vmware_desktop, 1.0, (amd64))
PS E:\VMWare_Storage\Virtual Machines\vagrant> vagrant box remove svetterIO/UbuntuDesktop22.04 --all
Removing box 'svetterIO/UbuntuDesktop22.04' (v1.0) with provider 'vmware_desktop'...
```


Thanks for reading :) Hopefully this helps.

## Vagrant Cheat Sheet

(AI generated be cautious of AI Slop, but I thought it could be useful)

```
VAGRANT(1)                     User Commands                     VAGRANT(1)

NAME
    vagrant - Create and manage reproducible development environments.

SYNOPSIS
    vagrant [command] [options]

DESCRIPTION
    Vagrant manages virtual machine environments. Commands operate on a
    Vagrant project (a directory containing a Vagrantfile) unless otherwise
    noted. Most operations require a Vagrantfile to be present.

COMMANDS
    init [box]
        Create a new Vagrantfile in the current directory.
        If a box name is provided, it is written into the Vagrantfile.

    up [--provider PROVIDER] [--provision]
        Create, configure, and boot the machine.
        If the provider is not specified, Vagrant chooses the default.
        --provision forces provisioners to run during startup.

    halt [-f]
        Gracefully shut down the running virtual machine.
        -f forces an immediate power-off.

    suspend
        Save the running VM’s state to disk and stop it.

    resume
        Restore a suspended VM to the running state.

    reload
        Restart the VM and apply changes from the Vagrantfile.

    destroy [-f]
        Remove the VM and all associated provider resources.
        -f runs without confirmation.

    ssh
        Connect to the guest machine over SSH.

    ssh-config
        Output SSH configuration for connecting manually.

    status
        Display the current state of the VM.

    provision
        Run the provisioners configured in the Vagrantfile.

BOX COMMANDS
    box list
        List boxes currently installed on the system.

    box add BOX
        Download and install a box.

    box remove BOX [--provider PROVIDER] [--all]
        Remove a box. If --all is used, all versions are removed.

GLOBAL COMMANDS
    global-status [--prune]
        Show status of all Vagrant environments on the system.
        --prune removes entries for destroyed or missing VMs.

PROJECT FILES
    Vagrantfile
        Ruby-based configuration file describing the VM environment.

    .vagrant/
        Directory storing local VM state for the current project.

DEFAULT LOCATIONS
    Boxes are stored under:
        C:\Users\<user>\.vagrant.d\boxes\
```




