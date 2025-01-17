---
layout: post
collection: automation
permalink: /automation/vagrant
title:  "Commands Vagrant"
author: Paulo Rogério
date:   2025-01-17 06:39:00 -0300
categories: [automation]
published: true
---

# Vagrant cheat-sheet

- [1) Creating a VM](#1-creating-a-vm)
- [2) Starting a VM](#2-starting-a-vm)
- [3) Getting Into a VM](#3-getting-into-a-vm)
- [4) Stopping a VM](#4-stopping-a-vm)
- [5) Cleaning Up a VM](#5-cleaning-up-a-vm)
- [6) Boxes](#6-boxes)
- [7) Saving Progress](#7-saving-progress)
- [8) Tips](#8-tips)
- [9) Plugins](#9-plugins)
- [10) Notes](#10-notes)  


## 1) Creating a VM


***Initialize Vagrant with a Vagrantfile and ./.vagrant directory, using no specified base image. Before you can do vagrant up, you'll need to specify a base image in the Vagrantfile.***

```bash
vagrant init
``` 

***Initialize Vagrant with a specific box. To find a box, go to the public Vagrant box catalog. When you find one you like, just replace it's name with boxpath. For example, vagrant init ubuntu/trusty64.***

```bash
vagrant init <boxpath>
``` 

## 2) Starting a VM

***Starts vagrant environment (also provisions only on the FIRST vagrant up).***

```bash
vagrant up
```

***Resume a suspended machine (vagrant up works just fine for this as well).***

```bash
vagrant resume
```

***Forces reprovisioning of the vagrant machine.***

```bash
vagrant provision 
```

***Restarts vagrant machine, loads new Vagrantfile configuration.***

```bash
vagrant reload 
```

***Restart the virtual machine and force provisioning.***

```bash
vagrant reload --provision 
```

## 3) Getting into a VM

***Connects to machine via SSH.***

```bash
vagrant ssh 
```

***If you give your box a name in your Vagrantfile, you can ssh into it with boxname. Works from any directory.***

```bash
vagrant ssh <boxname> 
```

## 4) Stopping a VM

***Stops the vagrant machine.***

```bash
vagrant halt
```

***Suspends a virtual machine (remembers state).***

```bash
vagrant suspend
``` 

## 5) Cleaning Up a VM

***Stops and deletes all traces of the vagrant machine.***

```bash
vagrant destroy
``` 

***Same as above, without confirmation.***

```bash
vagrant destroy -f 
```

## 6) Boxes

***See a list of all installed boxes on your computer.***

```bash
vagrant box list 
```

***Download a box image to your computer.***

```bash
vagrant box add <name> <url>
```

***Check for updates vagrant box update.***
```bash
vagrant box outdated
``` 

***Deletes a box from the machine.***

```bash
vagrant box remove <name>
```

***Packages a running virtualbox env in a reusable box.***
```bash
vagrant package 
```

## 7) Saving Progress

***Vm-name is often default. Allows us to save so that we can rollback at a later time.***

```bash
vagrant snapshot save [options] [vm-name] <name> 
```

## 8) Tips

***Get the vagrant version.***

```bash
vagrant -v
```

***Outputs status of the vagrant machine.***

```bash
vagrant status
```
***Outputs status of all vagrant machines.***

```bash
vagrant global-status
```

***Same as above, but prunes invalid entries.*** 

```bash
vagrant global-status --prune 
```

***Use the debug flag to increase the verbosity of the output.***

```bash
vagrant provision --debug
```

***Yes, vagrant can be configured to deploy code!***

```bash
vagrant push
```

***Runs vagrant up, forces provisioning and logs all output to a file.***

```bash
vagrant up --provision | tee provision.log 
```

## 9) Plugins

***Install vagrant-hostsupdater to update your /etc/hosts file automatically each time you start/stop your vagrant box.***

```bash
vagrant plugin install vagrant-hostsupdater
```

## 10) Notes 

***If you are using*** **VVV** ***, you can enable xdebug by running vagrant ssh and then xdebug_on from the virtual machine's CLI.***