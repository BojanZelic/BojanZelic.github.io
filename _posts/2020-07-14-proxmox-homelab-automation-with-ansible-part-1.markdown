---
layout: post
title:  "Proxmox Homelab automation with Gitlab & Ansible (Part 1)"
date:   2020-07-14 10:00:00 -0700
categories: devops
---

This post will act as a guide on how I managed to automate my home infrastructure 
utilizing Gitlab and Ansible. Using the power of ansible we can create VMs & LXC
containers to ensure that they are in our desired state. With a Gitlab runner we
can schedule & automate jobs to ensure that our LXC containers don't drift.

* Part 1 will cover how to automate Proxmox through Ansible
* Part 2 will cover how to automate the Gitlab runner through Ansible
* Part 3 will cover how to incorporate Gitlab into the build process

### Installing Ansible on Ubuntu

See the [official instructions](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-ubuntu) 
for the latest version of these instructions
```bash
sudo apt update
sudo apt install software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

### Bootstrapping your nodes ( Install Python with Ansible )
Proxmox needs to be able to run with ansible. It can technically run on a basic level,
but cannot gather facts without Python.

To get Ansible fully working we need to install Python... but first we need to create some 
files so that ansible has something to run;

_(replace your-domain.io with your hostname)_

**ansible.cfg**
```ini
[defaults]
inventory = hosts.ini
host_key_checking = False
interpreter_python = auto_silent
```

**hosts.ini**
```ini
[proxmox]
proxmox.your-domain.io
```

**playbook.yml**
```yaml
---
- name: bootstrap the server to get ready for ansible
  hosts: all
  remote_user: root
  gather_facts: false
  roles:
    - bootstrap
```

**roles/bootstrap/tasks/main.yml**
```yaml
---
- name: Check for Python
  raw: test -e /usr/bin/python
  changed_when: false
  failed_when: false
  register: check_python
  check_mode: no

- name: Install Python
  raw: test -e /usr/bin/apt && (apt -y update && apt install -y python) || (yum -y install python libselinux-python)
  when: check_python.rc != 0
```

This bootstrap code will get applied to every single node that 
we have (and will have in the future). It's super basic in what it does... but it 
just ensures that we install Python on our nodes. 

To test if it works simply run:

```bash
ansible-playbook playbook.yml 
```

### Install a Common configuration

We need to have a common configuration that can be shared between 
all nodes (including the proxmox node). This can setup our user. 

update playbook.yml and add the following entry:
**playbook.yml**
```yaml
- name: apply common configuration to all nodes
  hosts: all
  remote_user: root
  roles:
    - common
```

Add the following role aswell

**roles/common/tasks/main.yml**
{% raw %}
```yaml
---
- name: Include vars
  include_vars: common.yml

- name: Install Prerequisites
  apt: name=aptitude update_cache=yes state=present force_apt_get=yes

# Sudo Group Setup
- name: Make sure we have a 'wheel' group
  group:
    name: wheel
    state: present

# @todo, disable this in future,
# maybe add LDAP Server to get around having to storing password in ansible
- name: Allow 'wheel' group to have passwordless sudo
  lineinfile:
    path: /etc/sudoers
    state: present
    regexp: '^%wheel'
    line: '%wheel ALL=(ALL) NOPASSWD: ALL'
    validate: '/usr/sbin/visudo -cf %s'

# User + Key Setup
- name: Create a new regular user with sudo privileges
  user:
    name: "{{ create_user }}"
    state: present
    groups: wheel
    append: true
    create_home: true
    shell: /bin/bash

- name: Set authorized key for root user
  authorized_key:
    user: "root"
    state: present
    key: "{{ ansible_ssh_pub_key }}"

- name: Set authorized key for remote user
  authorized_key:
    user: "{{ create_user }}"
    state: present
    key: "{{ user_ssh_pub_key }}"

- name: Disable coredump in sudo
  lineinfile:
    path: /etc/sudo.conf
    state: present
    create: true
    regexp: '^Set disable_coredump'
    line: 'Set disable_coredump false'

- name: Disable password authentication for root
  lineinfile:
    path: /etc/ssh/sshd_config
    state: present
    regexp: '^#?PermitRootLogin'
    line: 'PermitRootLogin prohibit-password'

# Install Packages
- name: Install required system packages
  apt:
    name: [ 'curl', 'vim', 'git', 'net-tools' ]
    state: present
    update_cache: yes
```
{% endraw %}
And lastly add the following variables:

**vars/common.yml**
{% raw %}
```yaml
---
create_user: <your-username>
user_ssh_pub_key: "{{ lookup('url', 'https://github.com/<your-github-username>.keys') }}"
ansible_ssh_pub_key: "{{ lookup('file', lookup('env','HOME') + '/.ssh/root_id_rsa.pub') }}"
```
{% endraw %}
Replace the following variables:
* _your-username_ - replace this with the username you want to be created on all nodes
* *your-github-usernam*e - replace this with your github username. 
( The script will use your public github keys and allow you to SSH with the private key equivalent ) 


