# Ansible Cheatsheet

## Installation
```bash
# Ubuntu/Debian
sudo apt update && sudo apt install ansible -y

# macOS
brew install ansible

# pip
pip install ansible

# Verify
ansible --version
```

---

## Inventory

### Static Inventory (`inventory.ini`)
```ini
# Single hosts
web1 ansible_host=192.168.1.10 ansible_user=ubuntu
web2 ansible_host=192.168.1.11 ansible_user=ubuntu

# Groups
[webservers]
web1
web2

[dbservers]
db1 ansible_host=192.168.1.20

[production:children]
webservers
dbservers

[webservers:vars]
ansible_python_interpreter=/usr/bin/python3
http_port=80
```

### Dynamic Inventory (AWS)
```bash
pip install boto3
ansible-inventory -i aws_ec2.yaml --list
ansible-inventory -i aws_ec2.yaml --graph
```
```yaml
# aws_ec2.yaml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
filters:
  tag:Environment: production
keyed_groups:
  - key: tags.Role
    prefix: role
```

---

## Ad-hoc Commands
```bash
# Ping all hosts
ansible all -i inventory.ini -m ping

# Run shell command
ansible webservers -i inventory.ini -m shell -a "uptime"
ansible all -a "df -h"

# Copy file
ansible webservers -m copy -a "src=/local/file dest=/remote/file"

# Install package
ansible webservers -m apt -a "name=nginx state=present" -b

# Restart service
ansible webservers -m service -a "name=nginx state=restarted" -b

# Gather facts
ansible web1 -m setup
ansible web1 -m setup -a "filter=ansible_distribution*"

# Run as different user / sudo
ansible all -u ubuntu -b --become-user=root -a "whoami"
```

---

## Playbook Structure
```yaml
# site.yml
---
- name: Configure web servers
  hosts: webservers
  become: true
  vars:
    http_port: 80
    app_user: www-data

  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present

    - name: Copy nginx config
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart nginx

    - name: Ensure nginx is started
      service:
        name: nginx
        state: started
        enabled: yes

  post_tasks:
    - name: Verify nginx is running
      uri:
        url: "http://localhost"
        status_code: 200

  handlers:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

---

## Running Playbooks
```bash
ansible-playbook site.yml -i inventory.ini

# Limit to specific hosts/groups
ansible-playbook site.yml -l webservers
ansible-playbook site.yml -l web1,web2

# Dry run (check mode)
ansible-playbook site.yml --check
ansible-playbook site.yml --check --diff

# Step through tasks
ansible-playbook site.yml --step

# Start from specific task
ansible-playbook site.yml --start-at-task="Install nginx"

# Tags
ansible-playbook site.yml --tags "nginx,config"
ansible-playbook site.yml --skip-tags "debug"

# Extra variables
ansible-playbook site.yml -e "env=prod version=1.2.3"
ansible-playbook site.yml -e @vars.yml

# Verbose output
ansible-playbook site.yml -v    # basic
ansible-playbook site.yml -vvv  # connection debug
ansible-playbook site.yml -vvvv # include ssh debug
```

---

## Variables
```yaml
# Group vars: group_vars/webservers.yml
http_port: 80
max_connections: 1000

# Host vars: host_vars/web1.yml
ansible_host: 192.168.1.10

# In playbook
vars:
  app_name: myapp
  packages:
    - nginx
    - git
    - curl

# Register output
- name: Get disk usage
  command: df -h /
  register: disk_output

- name: Show disk usage
  debug:
    msg: "{{ disk_output.stdout }}"

# Variable precedence (highest to lowest)
# extra vars (-e) > task vars > block vars > role vars > play vars >
# host_vars > group_vars > defaults
```

---

## Conditionals & Loops
```yaml
# when
- name: Install on Debian
  apt:
    name: nginx
  when: ansible_os_family == "Debian"

- name: Skip if already done
  command: /opt/setup.sh
  when: not setup_done.stat.exists

# loop
- name: Install packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - git
    - curl

# loop with dict
- name: Create users
  user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
  loop:
    - { name: alice, groups: sudo }
    - { name: bob, groups: docker }

# loop with index
- name: Print items
  debug:
    msg: "{{ index }}: {{ item }}"
  loop: "{{ my_list }}"
  loop_control:
    index_var: index
```

---

## Common Modules

### Files
```yaml
- copy:
    src: files/app.conf
    dest: /etc/app/app.conf
    owner: root
    group: root
    mode: '0644'

- template:
    src: templates/config.j2
    dest: /etc/app/config.conf

- file:
    path: /opt/app
    state: directory
    mode: '0755'

- file:
    path: /tmp/old_file
    state: absent

- fetch:
    src: /var/log/app.log
    dest: logs/{{ inventory_hostname }}/
    flat: no
```

### Packages
```yaml
- apt:
    name: "{{ packages }}"
    state: present
    update_cache: yes
  vars:
    packages: [nginx, git, curl]

- yum:
    name: httpd
    state: latest

- pip:
    name: flask
    version: "2.0.1"
    virtualenv: /opt/venv
```

### Services
```yaml
- service:
    name: nginx
    state: started    # started | stopped | restarted | reloaded
    enabled: yes

- systemd:
    name: myapp
    state: restarted
    daemon_reload: yes
```

### Users & Groups
```yaml
- user:
    name: deploy
    groups: docker,sudo
    shell: /bin/bash
    create_home: yes

- authorized_key:
    user: deploy
    key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"

- group:
    name: docker
    state: present
```

### Command Execution
```yaml
- command: /opt/app/migrate.sh
  args:
    chdir: /opt/app

- shell: "echo $HOME && whoami"

- script: scripts/setup.sh
  args:
    creates: /opt/app/.initialized   # skip if file exists
```

### Git
```yaml
- git:
    repo: https://github.com/org/repo.git
    dest: /opt/app
    version: main
    force: yes
```

---

## Roles

### Directory Structure
```
roles/
  webserver/
    tasks/
      main.yml
    handlers/
      main.yml
    templates/
      nginx.conf.j2
    files/
      index.html
    vars/
      main.yml
    defaults/
      main.yml
    meta/
      main.yml
```

### Using Roles
```yaml
# site.yml
- hosts: webservers
  roles:
    - common
    - webserver
    - { role: database, db_port: 5432 }
```
```bash
# Create role scaffold
ansible-galaxy init roles/webserver
```

---

## Ansible Galaxy
```bash
# Install role
ansible-galaxy install geerlingguy.nginx

# Install collection
ansible-galaxy collection install community.general
ansible-galaxy collection install amazon.aws

# Install from requirements file
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml

# List installed roles
ansible-galaxy list
```

### requirements.yml
```yaml
roles:
  - name: geerlingguy.nginx
    version: "3.1.0"

collections:
  - name: community.general
    version: ">=5.0.0"
  - name: amazon.aws
```

---

## Vault (Secrets Management)
```bash
# Encrypt a file
ansible-vault encrypt secrets.yml

# Decrypt
ansible-vault decrypt secrets.yml

# View encrypted file
ansible-vault view secrets.yml

# Edit encrypted file
ansible-vault edit secrets.yml

# Encrypt a string (inline)
ansible-vault encrypt_string 'mysecretpassword' --name 'db_password'

# Run playbook with vault
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass

# Re-key vault
ansible-vault rekey secrets.yml
```

```yaml
# secrets.yml (encrypted with vault)
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  38663762...
```

---

## Templates (Jinja2)
```jinja2
{# templates/nginx.conf.j2 #}
server {
    listen {{ http_port }};
    server_name {{ ansible_hostname }};

    {% for vhost in virtual_hosts %}
    location /{{ vhost.name }} {
        proxy_pass http://{{ vhost.backend }};
    }
    {% endfor %}

    {% if ssl_enabled %}
    ssl_certificate {{ ssl_cert_path }};
    {% endif %}
}
```

---

## Error Handling
```yaml
- name: Might fail but that's OK
  command: /opt/risky.sh
  ignore_errors: yes

- name: Run and capture result
  command: /opt/check.sh
  register: result
  failed_when: result.rc != 0 and result.rc != 2

- name: Always run cleanup
  command: /opt/cleanup.sh
  when: always    # use in combination with block/rescue/always

# Block with error handling
- block:
    - name: Try this
      command: /opt/deploy.sh
    - name: And this
      command: /opt/migrate.sh
  rescue:
    - name: Rollback on failure
      command: /opt/rollback.sh
  always:
    - name: Always notify
      debug:
        msg: "Deploy attempted on {{ inventory_hostname }}"
```

---

## Useful ansible.cfg
```ini
[defaults]
inventory       = inventory.ini
remote_user     = ubuntu
private_key_file = ~/.ssh/id_rsa
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml
forks           = 10

[privilege_escalation]
become          = True
become_method   = sudo
become_user     = root
```

---

## Tips & Tricks
```bash
# List all hosts in inventory
ansible all --list-hosts

# List tasks in playbook without running
ansible-playbook site.yml --list-tasks
ansible-playbook site.yml --list-hosts
ansible-playbook site.yml --list-tags

# Test connectivity
ansible all -m ping

# Run on localhost
ansible localhost -m setup

# Pass SSH password (avoid; use keys instead)
ansible all -k   # --ask-pass
ansible all -K   # --ask-become-pass
```
