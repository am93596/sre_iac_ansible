# Infrastructure As Code (IAC) with Ansible

## What is IAC?
IAC means managing your IT infrastructure using provisioning scripts instead of carrying out the tasks manually. For example, launching an EC2 instance involves a lot of manual handling (clicks, etc) - a script of instructions could be written to carry this out for you.  

### Types of IAC
There are 2 types of IAC: configuration management (e.g. for managing provisioning for vagrant machines, or setting up EC2 instances), and orchestration (e.g. for setting up EC2 security groups)  
### Which tools are used for push config and pull config
- Push:
  - Ansible
  - SaltStack
- Pull:
  - Puppet
  - Chef  

## What is Ansible?
Ansible is an open-source configuration management tool. It is used for automating IT configuration tasks, such as launching and provisioning an EC2 instance.

## What are the benefits of Ansible?
Ansible is free and easy to use for automating tasks more efficiently. It is also provides scalability, as you can run commands on any number of machines at once, and it would be much faster than individually running them for each machine.  

## Why should we use Ansible
It makes the job of SRE easier, as it provides a simple way of quickly automating tasks, and is flexible in how many machines it is used for, and what you can do with it.  

## Diagram for Ansible on prem, hybrid, and public
*unfinished*

## Installation and setting up Ansible controller with 2 agent nodes
1. Clone this repository  
2. In Git bash, make sure you run `vagrant destroy`/destroy any other vagrant processes  
3. `vagrant up`
4. ssh into each machine, and run:  
```bash
sudo apt-get update -y
sudo apt-get upgrade -y
```  
5. Then ssh into controller machine, and run:  
```bash
sudo apt-get install software-properties-common -y
sudo apt-add-repository ppa:ansible/ansible -y
sudo apt-get install ansible -y
ansible --version
```  
6. To ssh into the web machine from inside the controller machine, run `ssh vagrant@192.168.33.10`; password: `vagrant`. When you are finished, enter `exit` to get back to the controller machine.  
7. To ssh into the db machine from inside the controller machine, run `ssh vagrant@192.168.33.11`; password: `vagrant`. When you are finished, enter `exit` to get back to the controller machine.  

## Default Directory Structure for Ansible  
/etc/ansible |  
              - roles  
              - ansible.cfg  
              - hosts  

## What is the Inventory/hosts file and its purpose  
The inventory file, also known as the hosts file, holds the list of agent nodes that are connected to the ansible controller. The file holds the group names (for classifying nodes), the IP for each machine, the method of connection, the default user for the machine, and the password for accessing the machine.  

## Using hosts file to establish a secure connection between ansible controller and agent nodes  
1. In the controller machine, run `cd /etc/ansible`, then `sudo nano hosts`  
2. Paste in the following:
```
[web]
192.168.33.10 ansible_connection=ssh ansible_user=vagrant ansible_ssh_pass=vagrant
[db]
192.168.33.11 ansible_connection=ssh ansible_user=vagrant ansible_ssh_pass=vagrant
```
## What are ansible `Ad-hoc commands`
Ansible ad-hoc commands are quick functions that can be run individually on more than one machine at a time.  
- Basic structure: `ansible <group name> <flags> <command> "<command to run inside the machine>"`  
- Examples:  
  - `ansible all -m ping`  
  - `ansible db -a "uname -a"`  
  - `ansible all -a "date"`  
  - `ansible web -a "free"`  

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

## Create a playbook to install and configure node and reverse proxy on the web machine
- `sudo nano node_playbook.yml`
```yaml
# Create a playbook to install and configure node.js etc on the web machine
# web 192.168.33.10
---
- hosts: web
  gather_facts: yes

  become: true

  tasks:
  - name: clone repo with the app folders
    git:
      repo: https://github.com/am93596/SRE_Intro_To_Cloud_Computing.git
      dest: /home/ubuntu
      clone: yes
      update: yes
  - name: install python software properties
    apt: pkg=software-properties-common state=present

  - name: Install python-software-properties
    apt: pkg=software-properties-common state=present

  - name: Add nodejs apt key
    apt_key:
      url: https://deb.nodesource.com/gpgkey/nodesource.gpg.key
      state: present

  - name: Add nodejs
    apt_repository:
      repo: deb https://deb.nodesource.com/node_13.x bionic main
      update_cache: yes

  - name: Install nodejs
    apt: pkg=nodejs state=present

  - name: install nodejs-legacy
    command: "apt-get install nodejs-legacy"

  - name: install pm2
    command: "npm install pm2 -g"
    args:
      chdir: "/home/ubuntu/app"

  - name: install npm
    command: "npm install"

  - name: reverse proxy
    shell: |
      rm /etc/nginx/sites-available/default
      ln -s /home/ubuntu/config_files/default /etc/nginx/sites-available/default
      nginx -t
      systemctl restart nginx

  - name: set db_host env variable and start app
    environment:
      DB_HOST: mongodb://192.168.33.11:27017/posts
    shell: |
      cd /home/ubuntu/app
      node seeds/seed.js
      pm2 kill
      pm2 start app.js
```
## Create a playbook to install and configure mongodb on the db machine
- `sudo nano mongodb_playbook.yml`
```yaml
# Create a playbook to install and configure mongodb on the db machine
# db 192.168.33.11
---
- hosts: db
  gather_facts: yes
  become: true

  tasks:
  - name: install mongodb
    apt: pkg=mongodb state=present

  - name: remove mongod.conf file
    file:
      path: /etc/mongod.conf
      state: absent

  - name: touch file and set permissions
    file:
      path: /etc/mongod.conf
      state: touch
      mode: u=rw,g=r,o=r

  - name: insert contents for mongod.conf
    blockinfile:
      path: /etc/mongod.conf
      backup: yes
      block: |
        "storage:
          dbPath: /var/lib/mongodb
          journal:
            enabled: true
        systemLog:
          destination: file
          logAppend: true
          path: /var/log/mongodb/mongod.log
        net:
          port: 27017
          bindIp: 0.0.0.0"
```
## Run the commands to start the app
In the controller machine, in /etc/ansible folder, run the following:
```bash
ansible-playbook nginx_playbook.yml
ansible-playbook mongodb_playbook.yml
ansible-playbook node_playbook.yml
```  
Then type the following into your browser: `192.168.33.10`, and then `192.168.33.10/posts`.
