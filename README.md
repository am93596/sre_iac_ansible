# Infrastructure As Code (IAC) with Ansible

## What is IAC?
IAC means managing your IT infrastructure using provisioning scripts instead of carrying out the tasks manually. For example, launching an EC2 instance involves a lot of manual handling (clicks, etc) - a script of instructions could be written to carry this out for you.

### Types of IAC
- Configuration Management
- Orchestration

### Which tools are used for push config and pull config

## What is Ansible?
Ansible is an open-source configuration management tool. It is used for automating IT configuration tasks, such as launching and provisioning an EC2 instance.

## What are the benefits of Ansible?
Ansible is free and easy to use for automating tasks more efficiently. It is also provides scalability, *unfinished*

## Why should we use Ansible

## Diagram for Ansible on prem, hybrid, and public

## Installation and setting up Ansible controller with 2 agent nodes - including commands

## Default Directory Structure for Ansible

## What is the Inventory/hosts file and its purpose

## What should be added to hosts file to establish secure connection between ansible controller and agent nodes? - include code

## What are ansible `Ad-hoc commands`

- add a structure of creating ad-hoc commands `ansible all -m ping`
- include all the ad-hoc commands we have used today in this documentation

## What are Ansible Playbooks?
- Ansible playbooks provide another way to use Ansible to automate tasks.
- Ansible playbooks are .yaml/.yml files written in Yet Another Markup Language (YAML)
- Playbooks start with 3 dashes (`---`)
```
# Create a playbook to install nginx web server on web machine
# web 192.168.33.10
# Let's add the 3 dashes to start the YAML 
---
# Add the name of the host
- hosts: web

  # Gather facts about the installation steps (optional)
  gather_facts: yes

  # We need admin access
  become: true

  # Add instructions to install nginx on web machine
  tasks:
  - name: Install Nginx
    apt: pkg=nginx state=present
    
# Ensure the server is running
```
- Run playbook with `ansible-playbook nginx_playbook.yml`
- Let's check if the playbook worked for us
- `ansible web -a "sudo systemctl status nginx"`
- Enter the IP address for the machine into the browser, and you should see the nginx page

Task:
- Create a playbook to install and configure node on the web machine
- Create a playbook to install and configure mongodb on the db machine
- Get the node app to work with /posts
- HINT: ansible official documentation available
- Youtube
- Stack Overflow
- Configure nginx reverse proxy


# Ansible controller and agent nodes set up guide
- Clone this repo and run `vagrant up`
- `(double check syntax/intendation)`

## We will use 18.04 ubuntu for ansible controller and agent nodes set up 
### Please ensure to refer back to your vagrant documentation

- **You may need to reinstall plugins or dependencies required depending on the OS you are using.**

```vagrant 
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what

# MULTI SERVER/VMs environment 
#
Vagrant.configure("2") do |config|

# creating first VM called web  
  config.vm.define "web" do |web|
    
    web.vm.box = "bento/ubuntu-18.04"
   # downloading ubuntu 18.04 image

    web.vm.hostname = 'web'
    # assigning host name to the VM
    
    web.vm.network :private_network, ip: "192.168.33.10"
    #   assigning private IP
    
    config.hostsupdater.aliases = ["development.web"]
    # creating a link called development.web so we can access web page with this link instread of an IP   
        
  end
  
# creating second VM called db
  config.vm.define "db" do |db|
    
    db.vm.box = "bento/ubuntu-18.04"
    
    db.vm.hostname = 'db'
    
    db.vm.network :private_network, ip: "192.168.33.11"
    
    config.hostsupdater.aliases = ["development.db"]     
  end

 # creating are Ansible controller
  config.vm.define "controller" do |controller|
    
    controller.vm.box = "bento/ubuntu-18.04"
    
    controller.vm.hostname = 'controller'
    
    controller.vm.network :private_network, ip: "192.168.33.12"
    
    config.hostsupdater.aliases = ["development.controller"] 
    
  end

end
```
