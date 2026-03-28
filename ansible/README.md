# Ansible

Ansible is an open-source, agentless automation platform that uses SSH (or WinRM for Windows) to configure systems, deploy applications, and orchestrate complex workflows. It uses human-readable YAML playbooks to describe the desired state of infrastructure, making it approachable for both developers and operations teams. Because it is agentless, there is nothing to install on managed nodes beyond Python and a working SSH connection, drastically reducing operational overhead.

---

## Table of Contents

1. [Installation](#installation)
2. [Inventory](#inventory)
3. [Ad-hoc Commands](#ad-hoc-commands)
4. [Playbooks](#playbooks)
5. [Roles](#roles)
6. [Ansible Vault](#ansible-vault)
7. [Modules](#modules)
8. [ansible-galaxy](#ansible-galaxy)
9. [Configuration (ansible.cfg)](#configuration-ansiblecfg)
10. [Troubleshooting & Debugging](#troubleshooting--debugging)
11. [Common Patterns & Tips](#common-patterns--tips)

---

## Installation

Ansible is distributed as a Python package and is available through most OS package managers.

```bash
# Ubuntu / Debian – add official PPA for the latest release
sudo apt update && sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible

# RHEL / CentOS / Fedora
sudo dnf install -y ansible          # Fedora / RHEL 9+
sudo yum install -y epel-release && sudo yum install -y ansible  # CentOS 7

# macOS (via Homebrew)
brew install ansible

# Any platform – pip (recommended for latest version)
pip install --user ansible            # user install
pip install ansible                   # inside a virtualenv

# Verify installation
ansible --version
# ansible [core 2.16.x]
#   python version = 3.x.x
#   jinja version  = 3.x.x
```

Key packages:
| Package | Purpose |
|---|---|
| `ansible` | Full suite including community modules |
| `ansible-core` | Minimal core only; add collections separately |
| `ansible-lint` | Lint playbooks for best practices |

---

## Inventory

The inventory tells Ansible **which hosts** to manage and how to group them.

### Static Inventory

A static inventory is a plain INI or YAML file maintained manually.

**INI format (`inventory/hosts`):**
```ini
# Ungrouped hosts
192.168.1.10
bastion.example.com

[webservers]
web1.example.com
web2.example.com ansible_port=2222

[dbservers]
db1.example.com ansible_user=postgres
db2.example.com

# Group of groups
[production:children]
webservers
dbservers

# Group variables
[webservers:vars]
http_port=80
nginx_worker_processes=4
```

**YAML format (`inventory/hosts.yml`):**
```yaml
all:
  vars:
    ansible_user: ubuntu
  children:
    webservers:
      hosts:
        web1.example.com:
          http_port: 80
        web2.example.com:
          http_port: 8080
    dbservers:
      hosts:
        db1.example.com:
        db2.example.com:
```

**Common host/group variables:**
| Variable | Description |
|---|---|
| `ansible_host` | IP or FQDN to connect to (overrides inventory name) |
| `ansible_port` | SSH port (default: 22) |
| `ansible_user` | Remote user |
| `ansible_ssh_private_key_file` | Path to SSH private key |
| `ansible_python_interpreter` | Path to Python on managed host |
| `ansible_become` | Enable privilege escalation |
| `ansible_become_method` | `sudo`, `su`, `doas`, etc. |

```bash
# Test inventory parsing
ansible-inventory -i inventory/hosts --list
ansible-inventory -i inventory/hosts --graph

# Ping all hosts in inventory
ansible all -i inventory/hosts -m ping
```

### Dynamic Inventory

Dynamic inventory scripts or plugins query an external source (AWS, GCP, Azure, etc.) at runtime.

```bash
# AWS EC2 plugin – install the collection first
ansible-galaxy collection install amazon.aws

# inventory/aws_ec2.yml
# ---
# plugin: amazon.aws.aws_ec2
# regions:
#   - us-east-1
# filters:
#   tag:Environment: production
# keyed_groups:
#   - key: tags.Role
#     prefix: role

# Run with the dynamic inventory
ansible -i inventory/aws_ec2.yml all -m ping

# List dynamic inventory
ansible-inventory -i inventory/aws_ec2.yml --list
```

---

## Ad-hoc Commands

Ad-hoc commands let you run a single module against hosts without writing a playbook — ideal for quick tasks and smoke tests.

```
ansible <pattern> -i <inventory> -m <module> -a "<args>" [options]
```

| Flag | Description |
|---|---|
| `-m` | Module name (default: `command`) |
| `-a` | Module arguments |
| `-i` | Inventory file or directory |
| `-u` / `--user` | Remote user |
| `-b` / `--become` | Escalate privileges |
| `--become-user` | Target user for privilege escalation |
| `-k` | Prompt for SSH password |
| `-K` | Prompt for become (sudo) password |
| `-f` | Forks (parallel processes, default: 5) |
| `--check` | Dry run — no changes made |
| `-v / -vv / -vvv` | Increase verbosity |

```bash
# Ping all hosts
ansible all -i inventory/hosts -m ping

# Run a shell command
ansible webservers -i inventory/hosts -m shell -a "df -h"

# Copy a file
ansible webservers -i inventory/hosts -m copy \
  -a "src=./nginx.conf dest=/etc/nginx/nginx.conf owner=root mode=0644" -b

# Install a package
ansible dbservers -i inventory/hosts -m apt \
  -a "name=postgresql state=present update_cache=yes" -b

# Restart a service
ansible webservers -i inventory/hosts -m service \
  -a "name=nginx state=restarted" -b

# Gather facts about hosts
ansible web1.example.com -i inventory/hosts -m setup

# Gather specific facts using filter
ansible all -i inventory/hosts -m setup -a "filter=ansible_distribution*"

# Reboot and wait for reconnect
ansible all -i inventory/hosts -m reboot -b
```

---

## Playbooks

Playbooks are YAML files that define a list of **plays**. Each play maps a group of hosts to a set of **tasks**.

### Structure

```yaml
---
# site.yml
- name: Configure web servers           # Play name (human-readable)
  hosts: webservers                     # Target host group
  become: true                          # Escalate privileges for entire play
  gather_facts: true                    # Run setup module first (default: true)
  vars:
    nginx_port: 80
  vars_files:
    - vars/secrets.yml

  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: true
        cache_valid_time: 3600

  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present

    - name: Deploy config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx             # Trigger handler on change

  post_tasks:
    - name: Verify nginx is running
      service_facts:

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

```bash
# Run a playbook
ansible-playbook -i inventory/hosts site.yml

# Dry run (check mode)
ansible-playbook -i inventory/hosts site.yml --check

# Show diff of changed files
ansible-playbook -i inventory/hosts site.yml --diff

# Limit to specific hosts or groups
ansible-playbook -i inventory/hosts site.yml --limit webservers
ansible-playbook -i inventory/hosts site.yml --limit web1.example.com

# Run only tasks with specific tags
ansible-playbook -i inventory/hosts site.yml --tags "install,configure"

# Skip specific tags
ansible-playbook -i inventory/hosts site.yml --skip-tags "restart"

# Start at a specific task
ansible-playbook -i inventory/hosts site.yml --start-at-task "Deploy config"

# Pass extra variables on the command line
ansible-playbook -i inventory/hosts site.yml -e "nginx_port=8080 env=prod"
```

### Variables

Variables can be defined in multiple places; Ansible resolves them by **precedence** (higher number wins):

| Precedence | Source |
|---|---|
| 1 (lowest) | Role defaults (`roles/x/defaults/main.yml`) |
| 5 | Inventory group vars |
| 10 | Inventory host vars |
| 13 | Play `vars` |
| 14 | Play `vars_files` |
| 17 | `include_vars` task |
| 21 (highest) | Extra vars (`-e`) |

```yaml
# Defining variables
vars:
  app_name: myapp
  app_version: "1.2.3"
  app_config:
    port: 8080
    debug: false

# Accessing variables
- name: Print app name
  debug:
    msg: "Deploying {{ app_name }} version {{ app_version }}"

# Nested variable access
- debug:
    msg: "Port is {{ app_config.port }}"    # dot notation
    # or: {{ app_config['port'] }}          # bracket notation

# Registered variables (capture task output)
- name: Get current date
  command: date +%Y-%m-%d
  register: current_date

- debug:
    msg: "Today is {{ current_date.stdout }}"

# Facts (auto-gathered host information)
- debug:
    msg: "OS is {{ ansible_distribution }} {{ ansible_distribution_version }}"
```

### Handlers

Handlers are tasks triggered by `notify` — they run **once** at the end of a play, regardless of how many tasks notified them.

```yaml
tasks:
  - name: Deploy nginx config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify:
      - Validate nginx config
      - Reload nginx

  - name: Deploy SSL cert
    copy:
      src: cert.pem
      dest: /etc/ssl/cert.pem
    notify: Reload nginx     # same handler; runs only once

handlers:
  - name: Validate nginx config
    command: nginx -t

  - name: Reload nginx
    service:
      name: nginx
      state: reloaded

# Force handlers to run immediately (before end of play)
- meta: flush_handlers
```

### Conditionals

```yaml
# when clause — supports Jinja2 expressions
- name: Install apache on Debian
  apt:
    name: apache2
    state: present
  when: ansible_os_family == "Debian"

- name: Install httpd on RedHat
  yum:
    name: httpd
    state: present
  when: ansible_os_family == "RedHat"

# Multiple conditions (AND)
- name: Configure service
  template:
    src: app.conf.j2
    dest: /etc/app.conf
  when:
    - env == "production"
    - app_version is defined

# OR condition
- name: Alert on old OS
  debug:
    msg: "Old OS detected"
  when: ansible_distribution_major_version == "7" or
        ansible_distribution_major_version == "8"

# Check if variable is defined
- debug:
    msg: "{{ my_var }}"
  when: my_var is defined

# Check register result
- name: Check if file exists
  stat:
    path: /etc/myapp.conf
  register: myapp_conf

- name: Create config if missing
  template:
    src: myapp.conf.j2
    dest: /etc/myapp.conf
  when: not myapp_conf.stat.exists
```

### Loops

```yaml
# Simple list loop
- name: Install packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - git
    - curl
    - vim
    - htop

# Loop over list of dicts
- name: Create users
  user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    state: present
  loop:
    - { name: alice, groups: "wheel,docker" }
    - { name: bob,   groups: "wheel" }

# loop_control — customise the loop variable name and label
- name: Deploy configs
  template:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
  loop:
    - { src: nginx.conf.j2,  dest: /etc/nginx/nginx.conf }
    - { src: logrotate.j2,   dest: /etc/logrotate.d/nginx }
  loop_control:
    label: "{{ item.dest }}"   # show dest in output, not full dict

# with_items (legacy, prefer loop:)
- name: Create directories
  file:
    path: "{{ item }}"
    state: directory
  with_items:
    - /opt/myapp
    - /opt/myapp/logs
    - /opt/myapp/config

# Loop with index
- debug:
    msg: "Item {{ idx }}: {{ item }}"
  loop: "{{ ['a', 'b', 'c'] }}"
  loop_control:
    index_var: idx
```

### Tags

Tags let you selectively run or skip parts of a playbook.

```yaml
- name: Install nginx
  apt:
    name: nginx
    state: present
  tags:
    - install
    - nginx

- name: Configure nginx
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  tags:
    - configure
    - nginx

# Special tags
- name: Always run this task
  debug:
    msg: "This always runs"
  tags: always

- name: Never run unless explicitly requested
  debug:
    msg: "Debug task"
  tags: never
```

```bash
# Run only 'configure' tagged tasks
ansible-playbook site.yml --tags configure

# Run 'install' and 'configure' tagged tasks
ansible-playbook site.yml --tags "install,configure"

# Skip 'restart' tagged tasks
ansible-playbook site.yml --skip-tags restart

# List all tags in a playbook
ansible-playbook site.yml --list-tags
```

---

## Roles

Roles are a standardised way to package and reuse automation content. They enforce a directory structure that Ansible understands.

### Structure

```
roles/
└── nginx/
    ├── defaults/       # Default variables (lowest precedence)
    │   └── main.yml
    ├── files/          # Static files to copy verbatim
    │   └── index.html
    ├── handlers/       # Handlers for this role
    │   └── main.yml
    ├── meta/           # Role metadata and dependencies
    │   └── main.yml
    ├── tasks/          # Main task list
    │   └── main.yml
    ├── templates/      # Jinja2 templates (.j2)
    │   └── nginx.conf.j2
    ├── tests/          # Role tests
    │   ├── inventory
    │   └── test.yml
    └── vars/           # Role variables (high precedence)
        └── main.yml
```

### Using Roles in a Playbook

```yaml
# site.yml
- name: Configure web servers
  hosts: webservers
  become: true
  roles:
    - common
    - nginx
    - { role: app_deploy, app_version: "2.1.0", tags: [deploy] }

# Or with include_role for dynamic use
- name: Conditionally include a role
  include_role:
    name: ssl_certs
  when: enable_ssl | bool
```

### Creating a Role with ansible-galaxy

```bash
# Create the scaffolding structure
ansible-galaxy role init roles/nginx

# Init in current directory
ansible-galaxy role init --init-path ./roles webserver

# Resulting structure:
# roles/nginx/
# ├── README.md
# ├── defaults/main.yml
# ├── files/
# ├── handlers/main.yml
# ├── meta/main.yml
# ├── tasks/main.yml
# ├── templates/
# ├── tests/inventory
# ├── tests/test.yml
# └── vars/main.yml
```

### meta/main.yml — Role Dependencies

```yaml
# roles/nginx/meta/main.yml
galaxy_info:
  role_name: nginx
  author: yourname
  description: Install and configure Nginx
  license: MIT
  min_ansible_version: "2.12"
  platforms:
    - name: Ubuntu
      versions:
        - jammy
        - focal

dependencies:
  - role: common
  - role: firewall
    vars:
      allowed_ports: [80, 443]
```

---

## Ansible Vault

Ansible Vault encrypts sensitive data (passwords, API keys, certificates) stored in YAML files using AES-256.

```bash
# Create a new encrypted file
ansible-vault create vars/secrets.yml

# Encrypt an existing file
ansible-vault encrypt vars/secrets.yml

# Decrypt a file in-place (use with caution)
ansible-vault decrypt vars/secrets.yml

# View encrypted file without decrypting to disk
ansible-vault view vars/secrets.yml

# Edit an encrypted file (opens $EDITOR)
ansible-vault edit vars/secrets.yml

# Re-key (change the vault password)
ansible-vault rekey vars/secrets.yml

# Encrypt a single string (embed in a YAML file)
ansible-vault encrypt_string 'supersecret' --name 'db_password'
# Output:
# db_password: !vault |
#   $ANSIBLE_VAULT;1.1;AES256
#   38633363396335623739...
```

### Using Vault at Runtime

```bash
# Prompt for vault password
ansible-playbook site.yml --ask-vault-pass

# Use a password file (do NOT commit this file)
ansible-playbook site.yml --vault-password-file ~/.vault_pass

# Use an environment variable via a script
ansible-playbook site.yml --vault-password-file ./vault_pass.sh

# Multiple vault IDs (label vaults for multi-password setups)
ansible-vault encrypt vars/dev.yml --vault-id dev@prompt
ansible-vault encrypt vars/prod.yml --vault-id prod@~/.vault_prod

ansible-playbook site.yml \
  --vault-id dev@prompt \
  --vault-id prod@~/.vault_prod
```

### vault_password_file in ansible.cfg

```ini
[defaults]
vault_password_file = ~/.vault_pass
```

---

## Modules

Modules are the units of work that Ansible executes on managed hosts.

### apt / yum — Package Management

Manages system packages on Debian (`apt`) and RHEL (`yum`/`dnf`) systems.

```yaml
# apt – Debian/Ubuntu
- name: Install and update nginx
  apt:
    name: nginx
    state: present          # present, absent, latest
    update_cache: true      # equivalent to apt-get update
    cache_valid_time: 3600  # skip update if cache < 1 hour old

- name: Install multiple packages
  apt:
    name:
      - git
      - curl
      - unzip
    state: present

# yum – RHEL/CentOS
- name: Install httpd
  yum:
    name: httpd
    state: latest
    enablerepo: epel

# package – OS-agnostic (uses the right module automatically)
- name: Install curl (any OS)
  package:
    name: curl
    state: present
```

### copy — Copy Files to Remote Hosts

```yaml
- name: Copy a static file
  copy:
    src: files/nginx.conf     # local path (relative to playbook or role)
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    backup: true              # keep backup of original

- name: Write content directly to a file
  copy:
    content: |
      server_name={{ server_name }};
    dest: /etc/nginx/conf.d/custom.conf
```

### template — Render Jinja2 Templates

Like `copy`, but processes Jinja2 expressions before writing.

```yaml
# tasks
- name: Deploy nginx config from template
  template:
    src: nginx.conf.j2        # relative to templates/ in role
    dest: /etc/nginx/nginx.conf
    owner: root
    mode: '0644'
    validate: nginx -t -c %s  # validate before placing file

# templates/nginx.conf.j2
# worker_processes {{ nginx_worker_processes | default(2) }};
# server {
#     listen {{ http_port }};
#     server_name {{ inventory_hostname }};
# }
```

### file — Manage Files, Directories, and Symlinks

```yaml
- name: Create a directory
  file:
    path: /opt/myapp/logs
    state: directory          # directory, file, link, absent, touch
    owner: app
    group: app
    mode: '0755'
    recurse: true             # apply permissions recursively

- name: Create a symlink
  file:
    src: /opt/myapp/current
    dest: /opt/myapp/live
    state: link

- name: Remove a file
  file:
    path: /tmp/oldfile.txt
    state: absent
```

### service — Manage System Services

```yaml
- name: Ensure nginx is started and enabled at boot
  service:
    name: nginx
    state: started            # started, stopped, restarted, reloaded
    enabled: true

# Use systemd module for more control
- name: Reload systemd and start app service
  systemd:
    name: myapp
    state: started
    enabled: true
    daemon_reload: true
```

### user — Manage User Accounts

```yaml
- name: Create application user
  user:
    name: appuser
    comment: "Application Service Account"
    uid: 1050
    group: appgroup
    groups:
      - docker
      - sudo
    shell: /bin/bash
    home: /home/appuser
    create_home: true
    password: "{{ vault_user_password | password_hash('sha512') }}"
    state: present

- name: Remove a user
  user:
    name: olduser
    state: absent
    remove: true              # also remove home directory
```

### shell / command — Run Commands

`command` runs a command without a shell (safer); `shell` runs via `/bin/sh` (supports pipes, redirects).

```yaml
- name: Run a simple command (no shell features needed)
  command: /usr/bin/myapp --init
  args:
    chdir: /opt/myapp         # working directory
    creates: /opt/myapp/.initialized  # skip if this file exists

- name: Run a shell pipeline
  shell: |
    ps aux | grep nginx | grep -v grep | wc -l
  register: nginx_procs
  changed_when: false         # this is a read-only command

- name: Run script with environment variables
  shell: ./deploy.sh
  args:
    chdir: /opt/myapp
  environment:
    APP_ENV: production
    DB_HOST: "{{ db_host }}"
```

### get_url — Download Files from URLs

```yaml
- name: Download application binary
  get_url:
    url: https://releases.example.com/myapp-1.2.3-linux-amd64.tar.gz
    dest: /opt/downloads/myapp-1.2.3.tar.gz
    checksum: sha256:abc123def456...   # verify integrity
    mode: '0644'
    timeout: 60

- name: Download with authentication
  get_url:
    url: https://private.example.com/artifact.zip
    dest: /opt/artifact.zip
    url_username: "{{ vault_username }}"
    url_password: "{{ vault_password }}"
```

### git — Clone / Update Git Repositories

```yaml
- name: Clone application repository
  git:
    repo: https://github.com/myorg/myapp.git
    dest: /opt/myapp
    version: main             # branch, tag, or commit SHA
    depth: 1                  # shallow clone
    force: false              # do not overwrite local changes
    update: true              # pull if already cloned

- name: Clone private repo via SSH
  git:
    repo: git@github.com:myorg/private-repo.git
    dest: /opt/private
    key_file: /home/deploy/.ssh/id_rsa
    accept_hostkey: true
```

### docker_container — Manage Docker Containers

Requires `community.docker` collection.

```yaml
- name: Run a Docker container
  community.docker.docker_container:
    name: myapp
    image: myorg/myapp:1.2.3
    state: started            # started, stopped, absent, present
    restart_policy: unless-stopped
    ports:
      - "8080:80"
    env:
      APP_ENV: production
      DB_URL: "postgresql://{{ db_host }}/myapp"
    volumes:
      - /opt/myapp/data:/data:rw
    networks:
      - name: myapp_network
```

### k8s — Manage Kubernetes Resources

Requires `kubernetes.core` collection.

```yaml
- name: Deploy Kubernetes Deployment
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: myapp
        namespace: production
      spec:
        replicas: 3
        selector:
          matchLabels:
            app: myapp
        template:
          metadata:
            labels:
              app: myapp
          spec:
            containers:
              - name: myapp
                image: myorg/myapp:1.2.3
                ports:
                  - containerPort: 8080
```

---

## ansible-galaxy

`ansible-galaxy` is the CLI tool for installing, managing, and publishing roles and collections from [Ansible Galaxy](https://galaxy.ansible.com) or private hubs.

```bash
# Install a role from Galaxy
ansible-galaxy role install geerlingguy.nginx

# Install a specific version
ansible-galaxy role install geerlingguy.nginx,3.2.0

# Install roles from a requirements file
ansible-galaxy role install -r requirements.yml

# Install a collection
ansible-galaxy collection install community.docker
ansible-galaxy collection install amazon.aws:==7.0.0

# Install collections from requirements
ansible-galaxy collection install -r requirements.yml

# requirements.yml example:
# ---
# roles:
#   - name: geerlingguy.nginx
#     version: "3.2.0"
#   - src: https://github.com/myorg/ansible-role-app
#     name: myapp
#     version: main
#
# collections:
#   - name: community.docker
#     version: ">=3.0.0"
#   - name: amazon.aws
#     version: "7.0.0"

# List installed roles
ansible-galaxy role list

# List installed collections
ansible-galaxy collection list

# Remove a role
ansible-galaxy role remove geerlingguy.nginx

# Search Galaxy
ansible-galaxy search nginx --author geerlingguy

# Publish a role (requires Galaxy API key)
ansible-galaxy role import myorg myrole
```

---

## Configuration (ansible.cfg)

Ansible reads configuration from (in precedence order):
1. `ANSIBLE_CONFIG` environment variable
2. `./ansible.cfg` (current directory)
3. `~/.ansible.cfg`
4. `/etc/ansible/ansible.cfg`

```ini
[defaults]
# Inventory
inventory          = ./inventory/hosts

# Remote user
remote_user        = ubuntu

# SSH key
private_key_file   = ~/.ssh/id_rsa

# Connection settings
forks              = 20          # parallel hosts (default: 5)
timeout            = 30          # SSH connection timeout in seconds
host_key_checking  = False       # disable for dynamic environments
gather_facts       = smart       # smart, always, explicit

# Output
stdout_callback    = yaml        # yaml, json, minimal, dense
display_skipped_hosts = False

# Logging
log_path           = ./ansible.log

# Roles path
roles_path         = ./roles:~/.ansible/roles:/etc/ansible/roles

# Vault
vault_password_file = ~/.vault_pass

# Retry files
retry_files_enabled = False

[privilege_escalation]
become             = True
become_method      = sudo
become_user        = root
become_ask_pass    = False

[ssh_connection]
ssh_args           = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no
pipelining         = True        # reduces SSH connections, speeds up playbooks
control_path       = %(directory)s/%%h-%%r

[inventory]
enable_plugins     = host_list, yaml, ini, auto, aws_ec2
```

---

## Troubleshooting & Debugging

### Verbosity

```bash
# Each -v adds more detail
ansible-playbook site.yml -v       # task results
ansible-playbook site.yml -vv      # task input/output
ansible-playbook site.yml -vvv     # SSH connection details
ansible-playbook site.yml -vvvv    # include connection plugin debugging
```

### debug Module

```yaml
- name: Print a variable
  debug:
    var: my_variable           # print value

- name: Print a message
  debug:
    msg: "The hostname is {{ inventory_hostname }}"

- name: Print only when verbose
  debug:
    msg: "Detailed info: {{ result }}"
    verbosity: 2               # only appears with -vv or more
```

### assert Module

```yaml
- name: Verify that required variables are set
  assert:
    that:
      - db_host is defined
      - db_host | length > 0
      - app_version is match('^[0-9]+\.[0-9]+\.[0-9]+$')
    fail_msg: "Required variables missing or invalid"
    success_msg: "All required variables are valid"
```

### fail Module

```yaml
- name: Fail with a clear message on unsupported OS
  fail:
    msg: "This playbook only supports Ubuntu 20.04+. Got: {{ ansible_distribution }} {{ ansible_distribution_version }}"
  when: ansible_distribution != "Ubuntu" or ansible_distribution_major_version | int < 20
```

### Checking Facts

```bash
# Gather and display all facts for a host
ansible web1.example.com -i inventory/hosts -m setup

# Filter facts by pattern
ansible web1.example.com -i inventory/hosts -m setup -a "filter=ansible_eth*"
ansible web1.example.com -i inventory/hosts -m setup -a "filter=ansible_memory_mb"

# Cache facts to speed up subsequent runs
# In ansible.cfg:
# [defaults]
# fact_caching         = jsonfile
# fact_caching_connection = ./fact_cache
# fact_caching_timeout = 3600
```

### Syntax Check and Lint

```bash
# Check playbook syntax only (no connection needed)
ansible-playbook site.yml --syntax-check

# Lint with ansible-lint
pip install ansible-lint
ansible-lint site.yml

# Step through tasks interactively
ansible-playbook site.yml --step
```

### Common Issues

```bash
# Host unreachable – test SSH manually
ssh -i ~/.ssh/id_rsa ubuntu@192.168.1.10

# Wrong Python interpreter on managed host
ansible webservers -m setup -e "ansible_python_interpreter=/usr/bin/python3"

# Connection refused – check SSH and firewall
ansible webservers -m ping -vvv

# Privilege escalation failure – ensure sudo is configured
ansible webservers -m command -a "whoami" -b -K
```

---

## Common Patterns & Tips

1. **Use `--check` before every production run.**  
   Always run `ansible-playbook --check --diff` before applying changes to production. This shows what would change without touching anything.

2. **Pin role and collection versions in `requirements.yml`.**  
   Avoid pulling the `latest` of a dependency. Pinning versions ensures consistent, reproducible automation across environments.

3. **Never store secrets in plaintext — use Vault.**  
   Use `ansible-vault encrypt_string` to embed encrypted values directly in YAML, or encrypt whole `vars/secrets.yml` files. Add `*.vault.yml` patterns to `.gitignore` for unencrypted files.

4. **Use `changed_when: false` for read-only shell tasks.**  
   When running `command` or `shell` tasks that only query state (e.g., `grep`, `cat`, `stat`), set `changed_when: false` to prevent misleading "changed" reports.

5. **Set `no_log: true` for tasks handling secrets.**  
   Any task that deals with passwords, tokens, or keys should have `no_log: true` to prevent sensitive data from appearing in logs or verbose output.

6. **Use `block` / `rescue` / `always` for error handling.**  
   Group related tasks in a `block`, use `rescue` like a try/catch to handle failures gracefully, and `always` for cleanup that must run regardless of outcome.

7. **Speed up playbooks with pipelining.**  
   Enable `pipelining = True` in `ansible.cfg` under `[ssh_connection]`. This reduces the number of SSH connections by running multiple tasks in a single connection, significantly improving performance.

8. **Prefer YAML over INI inventory for complex setups.**  
   YAML inventory is easier to extend with nested groups, group vars, and conditional host variables without external files.

9. **Use `delegate_to` for tasks that should run on a different host.**  
   For example, add a host to a load balancer from the web server's task context: `delegate_to: lb.example.com`.

10. **Keep roles small and single-purpose.**  
    A role should do one thing well (e.g., `nginx`, `postgres`, `users`). Compose complex configurations from multiple roles rather than building monolithic roles.
