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

### To import another playbook at the end/beginning of a playbook
```yaml
- name: running the next playbook
  import_playbook: 
```
*unfinished*

## Hybrid Cloud Infrastructure
### Install dependencies
- `sudo apt-get install tree -y`
- `cd /etc/ansible`
- `tree` -> displays the directory tree structure
- `sudo apt-add-repository --yes --update ppa:ansible/ansible`
- `sudo apt install python3-pip`
- `pip3 install awscli`
- `pip3 install boto boto3`
- `sudo apt-get upgrade -y`

### Checking installation
- `aws --version`
- outcome should be: `aws-cli/1.20.41 Python/3.6.9 Linux/4.15.0-151-generic botocore/1.21.41`
- if command not found, enter `exit`, then ssh back into the controller machine, and rerun the `aws --version` command
- `python --version` -> if this is 2.7.17, run the following command:
    - `alias python=python3`
    - if you run `python --version` again, it should be `Python 3.6.9`

### Create an ansible vault file to secure the AWS keys
- need to make .pem file accessible to ansible
    - copy the .pem file into ansible controller to ssh into ec2 instance
        - in a new git bash, `cd ~/.ssh` and `cat sre_key.pem`
        - in the controller git bash, in the same folder, `sudo nano sre_key.pem`, and paste in the contents, then save and close
        - run `sudo chmod 400 sre_key.pem`
    - generate ssh key in ansible controller called sre_key
        - `ssh-keygen -t rsa -b 4096`
        - name: `sre_key`, then press `Enter` for the other questions
- `cd /etc/ansible`
- `tree`
- `sudo mkdir group_vars`
- `cd group_vars`
- `sudo mkdir all`
- `cd all`
- `sudo ansible-vault create pass.yml`
    - New Vault password: `12345678`
    - enter the following:
    ```
    aws_access_key: <copy from excel file>
    aws_secret_key: <copy from excel file>
    ```
    - to exit, use `Esc` then `:wq!`
    - `sudo cat pass.yml` -> you can't see the contents, because they are encrypted :)
    - `sudo chmod 666 pass.yml`

*For running ad-hoc command on a machine, add `--ask-vault-pass` to the end, then enter your New Vault password*
*E.g. `ansible db -m ping --ask-vault-pass`*

### Making the ec2 playbook
- `cd /etc/ansible`
- `sudo nano create_ec2.yml`
- Enter the following contents:
```yaml
# launch ec2
---
- hosts: localhost
  connection: local
  gather_facts: true
  become: true
  vars:
    key_name: sre_key
    region: eu-west-1
    image: ami-0943382e114f188e8
    id: "SRE_Amy_Ansible_EC2"
    sec_group: "sg-055b237d59507ce50"
    subnet_id: "subnet-0429d69d55dfad9d2"
    # add if ansible by default uses python 2.7.17
    ansible_python_interpreter: /usr/bin/python3
  tasks:

    - name: Facts
      block:

        - name: Get instances facts
          ec2_instance_facts:
            aws_access_key: "{{aws_access_key}}"
            aws_secret_key: "{{aws_secret_key}}"
            region: "{{ region }}"
          register: result


    - name: Provisioning EC2 instances
      block:

      - name: Upload public key to AWS
        ec2_key:
          name: "{{ key_name }}"
          key_material: "{{ lookup('file', '~/.ssh/{{ key_name }}.pub') }}"
          region: "{{ region }}"
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"


      - name: Provision instance(s)
        ec2:
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"
          assign_public_ip: true
          key_name: "{{ key_name }}"
          id: "{{ id }}"
          vpc_subnet_id: "{{ subnet_id }}"
          group_id: "{{ sec_group }}"
          image: "{{ image }}"
          instance_type: t2.micro
          region: "{{ region }}"
          wait: true
          count: 1
          instance_tags:
            Name: sre_amy_ansible_ec2_app
        tags: ['never', 'create_ec2']
```
- Save and close

- Run `sudo ansible-playbook create_ec2.yml --ask-vault-pass --tags create_ec2`

- Edit hosts file in /etc/ansible: paste in the following (put in your instance's IP)
```yaml
[local]
localhost ansible_python_interpreter=/usr/local/bin/python3

[aws]
ec2-instance ansible_host=ec2-ip ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/sre_key.pem
```
- ssh into the machine, then exit

- then do `sudo ansible aws -m ping --ask-vault-pass`

