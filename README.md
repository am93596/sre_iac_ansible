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
  - name: clone repo with the app folders
    git:
      repo: https://github.com/am93596/SRE_Intro_To_Cloud_Computing.git
      dest: /home/ubuntu
      clone: yes
      update: yes

  - name: add key for mongodb
    shell: |
      wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add

  - name: connect to mongodb repo
    shell: |
      echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list

  - name: install and start mongodb
    shell: |
      sudo apt-get update -y
      sudo apt-get install -y mongodb-org
      sudo systemctl start mongod
      sudo systemctl enable mongod

  - name: remove original mongod.conf and replace with new one
    shell: |
      sudo rm /etc/mongod.conf
      sudo ln -s /home/ubuntu/config_files/mongod.conf /etc/mongod.conf

  - name: restart mongodb
    shell: |
      sudo systemctl restart mongod
```
## Run the commands to start the app
In the controller machine, in /etc/ansible folder, run the following:
```bash
ansible-playbook nginx_playbook.yml
ansible-playbook mongodb_playbook.yml
ansible-playbook node_playbook.yml
```  
Then type the following into your browser: `192.168.33.10`, and then `192.168.33.10/posts`.
