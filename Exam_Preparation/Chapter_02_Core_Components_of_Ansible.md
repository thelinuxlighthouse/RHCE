# Chapter 2: Understand Core Components of Ansible

## Learning Objectives

By the end of this chapter, you will be able to:

- Create and manage static and dynamic inventories
- Understand inventory groups, nested groups, and group variables
- Use Ansible modules effectively with correct parameters
- Define and scope variables at every level
- Leverage Ansible facts for dynamic decision-making
- Implement loops and conditional logic in playbooks
- Structure plays and playbooks for complex automation
- Handle task failures with error handling strategies
- Configure Ansible using `ansible.cfg`
- Create and use roles for reusable automation
- Navigate and use ansible-navigator documentation effectively

## Concept Overview

Ansible is built on a set of core components that work together to deliver configuration management, application deployment, and task automation. Understanding each component and how they interact is essential for writing effective playbooks and for succeeding on the RHCE EX294 exam.

### The Ansible Architecture

Ansible operates on a control node that pushes tasks to managed nodes over SSH. No agents are installed on managed nodes. The control node connects, executes a module, receives structured JSON output, and reports the result. This agentless architecture makes Ansible simple to deploy and easy to manage.

### Why Core Components Matter

Each component serves a specific purpose:

- **Inventories** define the target infrastructure
- **Modules** perform the actual work on managed nodes
- **Variables** make playbooks dynamic and reusable
- **Facts** provide real-time system information
- **Loops** eliminate task duplication
- **Conditionals** enable intelligent branching
- **Plays** organize tasks into logical units
- **Error handling** ensures resilience
- **Playbooks** combine everything into executable automation
- **Configuration files** control global behavior
- **Roles** package automation for reuse

## Ansible Fundamentals Required

### Control Node vs Managed Node

The control node runs Ansible and executes playbooks. Managed nodes are the target systems being configured. The control node communicates with managed nodes via SSH using Python on the remote end.

### SSH Key-Based Authentication

Ansible requires passwordless SSH access to managed nodes. Keys are generated on the control node and distributed to managed nodes.

```bash
# Generate SSH key pair on control node
ssh-keygen -t ed25519 -C "ansible-control" -N ""

# Distribute public key to managed node
ssh-copy-id -i ~/.ssh/id_ed25519.pub ansible@servera.lab.example.com

# Verify connectivity
ssh ansible@servera.lab.example.com "echo connection successful"
```

### Privilege Escalation

Managed nodes must allow the Ansible user to escalate to root via `sudo`. The managed node's `/etc/sudoers` must include:

```
ansible ALL=(ALL) NOPASSWD: ALL
```

## Inventories

### Static Inventory Files

Static inventories are INI or YAML files that list managed hosts and organize them into groups.

#### INI Format

```ini
# /etc/ansible/hosts

# Individual hosts
servera.lab.example.com
serverb.lab.example.com

# Grouped hosts
[webservers]
servera.lab.example.com
serverb.lab.example.com

[dbservers]
serverc.lab.example.com

# Nested groups
[production:children]
webservers
dbservers

# Host variables inline
[workstations]
workstationa ansible_user=admin ansible_port=22

# Group variables
[webservers:vars]
http_port=8080
app_name=myapp
```

#### YAML Format

```yaml
# inventory.yml
all:
  children:
    webservers:
      hosts:
        servera.lab.example.com:
          http_port: 8080
        serverb.lab.example.com:
          http_port: 8443
      vars:
        app_name: myapp
    dbservers:
      hosts:
        serverc.lab.example.com:
        database_port: 5432
    production:
      children:
        webservers
        dbservers
```

### Inventory Variables

Variables can be defined at three levels in the inventory:

```ini
# Host variable - applies only to this host
servera ansible_user=alice ansible_port=22

# Group variable - applies to all hosts in group
[webservers:vars]
http_port=8080
app_name=myapp

# Group host - adds host to group
[production:children]
webservers
dbservers
```

### Host and Group Variable Files

Variables can also be stored in dedicated directories:

```
inventory/
  hosts                    # inventory file
  host_vars/
    servera.yml            # variables for servera only
    serverb.yml            # variables for serverb only
  group_vars/
    webservers.yml         # variables for webservers group
    dbservers.yml          # variables for dbservers group
    all.yml                # variables for all hosts
```

```yaml
# host_vars/servera.yml
http_port: 8080
max_clients: 256
app_name: myapp

# group_vars/webservers.yml
http_packages:
  - httpd
  - mod_ssl
firewall_ports:
  - "80/tcp"
  - "443/tcp"
```

### Inventory Plugins and Dynamic Inventory

Dynamic inventories generate host lists from external sources (cloud providers, CMDBs). For the exam, focus on static inventories, but be aware that dynamic inventories use plugins.

```bash
# List all hosts in inventory
ansible-inventory --list

# List hosts in a specific group
ansible-inventory --graph

# Get variables for a specific host
ansible-inventory --host servera.lab.example.com
```

## Modules

### Module Categories

Ansible modules fall into several categories:

| Category | Examples | Purpose |
|---|---|---|
| Files | `copy`, `template`, `file`, `lineinfile` | File management |
| Package | `dnf`, `dnf_module`, `yum` | Software installation |
| System | `systemd`, `selinux`, `cron`, `user` | System configuration |
| Networking | `firewalld`, `iptables` | Network configuration |
| Commands | `command`, `shell`, `script` | Command execution |
| Cloud | `ec2`, `azure_rm_virtualmachine` | Cloud resource management |
| Database | `postgresql_user`, `mysql_db` | Database management |

### Module Execution Model

When a task specifies a module, Ansible:

1. Transfers the module payload to the managed node
2. Executes the module with provided parameters
3. Receives structured JSON output
4. Reports changed/failed/skipped status

```yaml
# Module task structure
- name: Task description
  module_name:
    parameter1: value1
    parameter2: value2
```

### Finding Module Documentation

```bash
# List all available modules
ansible-doc -l

# Search for modules by keyword
ansible-doc -l | grep -i firewall

# Get detailed module documentation
ansible-doc dnf
ansible-doc systemd
ansible-doc firewalld

# Get module examples
ansible-doc -s dnf
ansible-doc -s systemd
```

### Common Module Parameters

Many modules share common parameters:

| Parameter | Purpose |
|---|---|
| `name` | Resource identifier (package name, service name, etc.) |
| `state` | Desired state (present, absent, started, stopped, mounted) |
| `src` | Source file path |
| `dest` | Destination file path |
| `owner` | File/resource owner |
| `group` | File/resource group |
| `mode` | File permissions (octal string) |
| `backup` | Create backup of existing file |
| `validate` | Command to validate file before writing |

### Module Idempotency

Ansible modules are designed to be idempotent — running the same task multiple times produces the same result without unintended side effects.

```yaml
# This task is idempotent - safe to run repeatedly
- name: Ensure vim is installed
  dnf:
    name: vim-enhanced
    state: present

# This task is NOT idempotent - always executes
- name: Bad practice
  shell: dnf install -y vim-enhanced
```

## Variables

### Variable Precedence

Variables can be defined at multiple levels. Understanding precedence is critical:

```
Lowest Priority
  1. command line extra vars
  2. inventory file/group vars
  3. play vars, vars_files
  4. role defaults (lowest role level)
  5. role vars
  6. task vars
  7. registered variables
Highest Priority
```

### Defining Variables

#### Extra Variables (Command Line)

```bash
ansible-playbook playbook.yml -e "app_name=myapp http_port=8080"
ansible-playbook playbook.yml -e "@vars_file.yml"
```

#### Play-Level Variables

```yaml
- name: Configure web server
  hosts: webservers
  vars:
    app_name: myapp
    http_port: 8080
    max_clients: 256
```

#### Variables from Files

```yaml
- name: Configure web server
  hosts: webservers
  vars_files:
    - vars/common.yml
    - vars/webserver.yml
```

```yaml
# vars/common.yml
ansible_become: yes
log_level: info

# vars/webserver.yml
http_port: 8080
app_name: myapp
max_clients: 256
```

#### Role Default Variables

```yaml
# roles/webserver/defaults/main.yml
http_port: 80
app_name: default_app
max_clients: 128
```

### Variable Naming Conventions

Variable names must:

- Start with a letter
- Contain only letters, numbers, and underscores
- Not contain hyphens or spaces

```yaml
# Good variable names
app_name: myapp
http_port: 8080
max_clients: 256

# Bad variable names
app-name: myapp    # hyphens not allowed
http port: 8080   # spaces not allowed
2nd_port: 8443    # starts with number
```

### Variable Types

```yaml
# String
app_name: myapp

# Integer
http_port: 8080

# Boolean
enable_ssl: true
enable_ssl: yes    # also valid

# List
packages:
  - httpd
  - mod_ssl
  - firewalld

# Dictionary
server_config:
  port: 8080
  workers: 4
  log_level: debug

# Nested structures
webservers:
  - name: servera
    port: 8080
    zone: us-east
  - name: serverb
    port: 8443
    zone: us-west
```

### Variable Manipulation

#### Default Values

```yaml
# Use default if variable is undefined
- name: Use port variable
  debug:
    msg: "Port is {{ http_port | default(80) }}"
```

#### String Operations

```yaml
- name: String manipulation
  debug:
    msg: "{{ item }}"
  loop:
    - "{{ app_name | upper }}"
    - "{{ app_name | lower }}"
    - "{{ app_name | capitalize }}"
    - "{{ app_name | replace('-', '_') }}"
```

#### Numeric Operations

```yaml
- name: Numeric comparison
  debug:
    msg: "Port is high"
  when: http_port | int > 1024
```

## Facts

### What Are Facts

Facts are variables automatically gathered from managed nodes. They include system information such as OS version, IP addresses, CPU count, memory, disk space, network interfaces, and installed packages.

### Gathering Facts

Facts are gathered automatically at the start of each play. Control fact gathering with `gather_facts`:

```yaml
# Default behavior - facts are gathered
- name: Play with facts
  hosts: all

# Disable fact gathering for speed
- name: Play without facts
  hosts: all
  gather_facts: no

# Selective fact gathering
- name: Play with selective facts
  hosts: all
  gather_facts: yes
  setup:
    gather_subset:
      - "!all"
      - "net"
      - "hardware"
```

### Common Facts

```yaml
# OS information
ansible_os_family          # "RedHat", "Debian", etc.
ansible_distribution       # "RedHat", "CentOS", etc.
ansible_distribution_major_version  # "9"
ansible_distribution_version        # "9.2"

# Hardware
ansible_processor_vcpus    # Number of CPU cores
ansible_memtotal_mb        # Total memory in MB
ansible_swaptotal_mb       # Total swap in MB

# Network
ansible_default_ipv4.address   # Primary IPv4 address
ansible_default_ipv4.alias     # Network interface name
ansible_all_ipv4_addresses     # List of all IPv4 addresses

# Date and time
ansible_date_time.date       # Current date
ansible_date_time.time       # Current time
ansible_date_time.iso8601    # ISO 8601 timestamp

# Filesystem
ansible_mounts               # List of mounted filesystems
ansible_devices              # Block devices
```

### Using Facts in Playbooks

```yaml
---
- name: Use facts for conditional configuration
  hosts: all
  become: yes
  tasks:
    - name: Display OS information
      debug:
        msg: "Running {{ ansible_distribution }} {{ ansible_distribution_version }}"

    - name: Install packages on Red Hat systems
      dnf:
        name: vim-enhanced
        state: present
      when: ansible_os_family == "RedHat"

    - name: Install packages on Debian systems
      apt:
        name: vim
        state: present
      when: ansible_os_family == "Debian"

    - name: Configure based on CPU count
      lineinfile:
        path: /etc/myapp/config.ini
        regexp: "^workers = "
        line: "workers = {{ ansible_processor_vcpus }}"
      when: ansible_processor_vcpus is defined

    - name: Set memory-based configuration
      lineinfile:
        path: /etc/myapp/config.ini
        regexp: "^max_memory = "
        line: "max_memory = {{ ansible_memtotal_mb // 2 }}"
      when: ansible_memtotal_mb is defined

    - name: Display primary IP address
      debug:
        msg: "Primary IP: {{ ansible_default_ipv4.address }}"
```

### Custom Facts

Custom facts are scripts or data files placed on managed nodes that return additional information.

```bash
# Create custom fact on managed node
echo '{"custom_app_version": "2.1.0", "custom_app_status": "active"}' > /etc/ansible/facts.d/myapp.fact
```

```yaml
# Access custom facts in playbook
- name: Display custom facts
  debug:
    msg: "App version: {{ ansible_facts.ansible_facts.custom_app_version }}"
```

### Setup Module for Manual Fact Gathering

```bash
# Gather all facts
ansible servera -m setup

# Filter specific facts
ansible servera -m setup -a "filter=ansible_os_family"
ansible servera -m setup -a "filter=ansible_processor*"
ansible servera -m setup -a "filter=ansible_mounts"
```

## Loops

### Basic Loop with `loop`

```yaml
---
- name: Loop examples
  hosts: all
  become: yes
  tasks:
    - name: Install multiple packages
      dnf:
        name: "{{ item }}"
        state: present
      loop:
        - vim-enhanced
        - net-tools
        - wget
        - curl
        - tree

    - name: Create multiple users
      user:
        name: "{{ item }}"
        state: present
        createhome: yes
      loop:
        - alice
        - bob
        - charlie

    - name: Open multiple firewall ports
      firewalld:
        port: "{{ item }}"
        permanent: yes
        immediate: yes
        state: enabled
      loop:
        - "80/tcp"
        - "443/tcp"
        - "8080/tcp"
```

### Loop with Dictionaries

```yaml
- name: Create users with attributes
  user:
    name: "{{ item.name }}"
    uid: "{{ item.uid }}"
    groups: "{{ item.groups }}"
    comment: "{{ item.comment }}"
    shell: "{{ item.shell | default('/bin/bash') }}"
    state: present
  loop:
    - name: alice
      uid: 2001
      groups: wheel
      comment: "Alice Admin"
    - name: bob
      uid: 2002
      groups: docker
      comment: "Bob Developer"
      shell: /bin/zsh
    - name: charlie
      uid: 2003
      groups: admins
      comment: "Charlie Operator"
```

### Loop with `with_items`

The `with_items` loop is the legacy syntax. It is functionally equivalent to `loop` but `loop` is the modern preferred form.

```yaml
# Legacy syntax (still works)
- name: Install packages (legacy)
  dnf:
    name: "{{ item }}"
    state: present
  with_items:
    - vim-enhanced
    - wget
```

### Loop Control

```yaml
- name: Install packages with loop control
  dnf:
    name: "{{ item }}"
    state: present
  loop:
    - httpd
    - mod_ssl
    - php
    - mariadb
  loop_control:
    label: "{{ item }}"
    pause: 2

- name: Process items with index
  debug:
    msg: "Processing {{ item }} (index: {{ ansible_loop.index0 }})"
  loop:
    - task1
    - task2
    - task3
```

### Nested Loops with `product`

```yaml
- name: Create user-group combinations
  user:
    name: "{{ item.0 }}"
    groups: "{{ item.1 }}"
    append: yes
    state: present
  loop: "{{ product(users, groups) | list }}"
  vars:
    users:
      - alice
      - bob
    groups:
      - wheel
      - docker
```

### Loop with Registered Variables

```yaml
- name: Check status of multiple services
  command: systemctl is-active {{ item }}
  register: service_status
  changed_when: false
  failed_when: false
  loop:
    - httpd
    - firewalld
    - sshd

- name: Display service statuses
  debug:
    var: service_status.results
```

## Conditional Tasks

### Basic Conditionals with `when`

```yaml
---
- name: Conditional task examples
  hosts: all
  become: yes
  vars:
    enable_firewall: true
    enable_httpd: true
    http_port: 8080
  tasks:
    - name: Install httpd if enabled
      dnf:
        name: httpd
        state: present
      when: enable_httpd

    - name: Open firewall port if firewall is enabled
      firewalld:
        port: "{{ http_port }}/tcp"
        permanent: yes
        immediate: yes
        state: enabled
      when:
        - enable_firewall
        - enable_httpd

    - name: Install package on Red Hat only
      dnf:
        name: vim-enhanced
        state: present
      when: ansible_os_family == "RedHat"

    - name: Install package on specific version
      dnf:
        name: postgresql-server
        state: present
      when: ansible_distribution_major_version == "9"
```

### Comparison Operators

```yaml
- name: Numeric comparison
  debug:
    msg: "High port number detected"
  when: http_port | int > 1024

- name: String comparison
  debug:
    msg: "Using custom app"
  when: app_name != "default"

- name: Check if value is defined
  debug:
    msg: "Custom port configured"
  when: custom_port is defined

- name: Check if value is not defined
  debug:
    msg: "Using default port"
  when: custom_port is not defined

- name: Check if list is empty
  debug:
    msg: "No extra packages to install"
  when: extra_packages | length == 0

- name: Check if string contains substring
  debug:
    msg: "Production environment"
  when: "'prod' in environment"
```

### Boolean Logic

```yaml
- name: AND condition
  debug:
    msg: "Both conditions met"
  when:
    - enable_httpd
    - http_port | int > 0

- name: OR condition
  debug:
    msg: "At least one condition met"
  when: enable_httpd or enable_nginx

- name: NOT condition
  debug:
    msg: "HTTPD is disabled"
  when: not enable_httpd

- name: Complex boolean expression
  debug:
    msg: "Production web server"
  when: (environment == "production") and (enable_httpd or enable_nginx)
```

### Conditional Loops

```yaml
- name: Install packages based on role
  dnf:
    name: "{{ item }}"
    state: present
  loop: "{{ web_packages if server_role == 'web' else db_packages }}"
  vars:
    web_packages:
      - httpd
      - mod_ssl
    db_packages:
      - mariadb-server
      - mariadb
    server_role: "web"
```

### `ignore_errors` vs `failed_when`

```yaml
- name: Ignore errors entirely
  command: systemctl is-active unknown-service
  ignore_errors: yes
  changed_when: false

- name: Custom failure condition
  command: rpm -q postgresql-server
  register: pg_check
  changed_when: false
  failed_when: false

- name: Report result
  debug:
    msg: "PostgreSQL is installed"
  when: pg_check.rc == 0
```

## Plays and Playbooks

### Play Structure

A play defines which hosts to target and what tasks to execute.

```yaml
---
- name: Play description
  hosts: target_group
  become: yes
  vars:
    variable_name: value
  tasks:
    - name: Task description
      module:
        parameter: value
  handlers:
    - name: Handler description
      module:
        parameter: value
```

### Multiple Plays in One Playbook

```yaml
---
# Play 1: Configure all servers
- name: Configure base system
  hosts: all
  become: yes
  tasks:
    - name: Update all packages
      dnf:
        name: "*"
        state: latest

# Play 2: Configure web servers
- name: Configure web servers
  hosts: webservers
  become: yes
  tasks:
    - name: Install httpd
      dnf:
        name: httpd
        state: present
      notify: restart httpd

# Play 3: Verify configuration
- name: Verify web servers
  hosts: webservers
  tasks:
    - name: Check httpd is running
      uri:
        url: http://localhost:80
        status_code: 200
```

### Play Delegation and Gathering

```yaml
- name: Delegate task to different host
  hosts: webservers
  become: yes
  tasks:
    - name: Install package on database server
      dnf:
        name: mariadb-server
        state: present
      delegate_to: serverc.lab.example.com

    - name: Gather facts from delegated host
      setup:
      delegate_to: serverc.lab.example.com
```

### Serial Execution

```yaml
- name: Rolling update
  hosts: webservers
  serial: 1
  become: yes
  tasks:
    - name: Update packages
      dnf:
        name: "*"
        state: latest
      notify: reboot server

  handlers:
    - name: reboot server
      reboot:
```

### Serial with Percentage

```yaml
- name: Update 50 percent at a time
  hosts: webservers
  serial: "50%"
  become: yes
  tasks:
    - name: Update packages
      dnf:
        name: "*"
        state: latest
```

## Handling Task Failure

### `ignore_errors`

Continues playbook execution even if a task fails.

```yaml
- name: Task that may fail
  command: systemctl is-active optional-service
  ignore_errors: yes
  changed_when: false
```

### `failed_when`

Customize when a task is considered failed.

```yaml
- name: Check for optional package
  command: rpm -q optional-package
  register: pkg_check
  changed_when: false
  failed_when: pkg_check.rc > 1

- name: Install if not present
  dnf:
    name: optional-package
    state: present
  when: pkg_check.rc != 0
```

### `changed_when`

Control whether a task reports a change.

```yaml
- name: Read configuration value
  command: grep max_clients /etc/myapp/config.ini
  register: config_value
  changed_when: false
```

### `check_mode`

Test playbooks without making changes.

```bash
# Run playbook in check mode
ansible-playbook playbook.yml --check

# Run with diff output
ansible-playbook playbook.yml --check --diff
```

```yaml
# Task that always changes in check mode
- name: Task with check mode support
  command: echo "hello"
  changed_when: false
  check_mode: no
```

### Block and Rescue

Structured error handling with try-catch-like behavior.

```yaml
- name: Error handling with block
  hosts: all
  become: yes
  tasks:
    - name: Attempt configuration with error handling
      block:
        - name: Install package
          dnf:
            name: risky-package
            state: present

        - name: Deploy configuration
          copy:
            content: "risky config"
            dest: /etc/myapp/risky.conf

        - name: Start service
          systemd:
            name: risky-service
            state: started
      rescue:
        - name: Rollback on failure
          debug:
            msg: "Configuration failed, rolling back"

        - name: Remove risky package
          dnf:
            name: risky-package
            state: absent
          ignore_errors: yes

        - name: Remove risky config
          file:
            path: /etc/myapp/risky.conf
            state: absent
          ignore_errors: yes
      always:
        - name: Log attempt
          debug:
            msg: "Configuration attempt completed"
```

### Freezing State on Failure

```yaml
- name: Stop on first failure
  hosts: all
  any_errors_fatal: yes
  become: yes
  tasks:
    - name: Critical task
      command: /usr/local/bin/critical-setup.sh

    - name: This will not run if previous task fails
      dnf:
        name: httpd
        state: present
```

### Maximum Fail Percentage

```yaml
- name: Allow some failures
  hosts: all
  max_fail_percentage: 25
  become: yes
  tasks:
    - name: Non-critical task
      command: /usr/local/bin/optional-setup.sh
      ignore_errors: yes
```

## Configuration Files

### `ansible.cfg`

The Ansible configuration file controls global behavior. Ansible searches for this file in order:

1. `ANSIBLE_CONFIG` environment variable
2. `ansible.cfg` in current directory
3. `~/.ansible.cfg` in home directory
4. `/etc/ansible/ansible.cfg` system-wide

```ini
# ansible.cfg
[defaults]
# Inventory file location
inventory = /etc/ansible/hosts

# Remote user for SSH connections
remote_user = ansible

# Privilege escalation
become = true
become_method = sudo
become_user = root

# Disable host key checking (exam environment only)
host_key_checking = false

# Enable fact caching
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts_cache

# Log file
log_path = /var/log/ansible.log

# Number of forks (parallel connections)
forks = 10

# Timeout for SSH connections
timeout = 30

# Enable collection paths
collections_paths = /home/ansible/.ansible/collections:/usr/share/ansible/collections

# Callback plugins
callback_whitelist = timer, profile_tasks

[privilege_escalation]
become = true
become_method = sudo
become_user = root
become_ask_pass = false

[ssh_connection]
# Use SSH pipelining for performance
pipelining = true
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

### Creating `ansible.cfg` for the Exam

```bash
# Create ansible.cfg in working directory
cat > ansible.cfg << 'EOF'
[defaults]
inventory = ./inventory
remote_user = ansible
host_key_checking = false
log_path = /home/ansible/ansible.log
forks = 10

[privilege_escalation]
become = true
become_method = sudo
become_user = root
become_ask_pass = false

[ssh_connection]
pipelining = true
EOF
```

### `ansible-navigator` Configuration

```yaml
# ansible-navigator.yml
ansible-navigator:
  execution-environment:
    enabled: false
  mode: stdout
  ansible:
    diff: true
  logging:
    level: warning
```

## Roles

### Role Directory Structure

```
roles/
  webserver/
    tasks/
      main.yml          # Main task list
    handlers/
      main.yml          # Handlers
    templates/
      httpd.conf.j2     # Jinja2 templates
    files/
      index.html        # Static files
    vars/
      main.yml          # Role variables
    defaults/
      main.yml          # Default variables
    meta/
      main.yml          # Role metadata
    tests/
      test.yml          # Role tests
```

### Creating a Role

```bash
# Create role skeleton
ansible-galaxy init webserver

# Creates standard directory structure
# roles/webserver/
#   tasks/main.yml
#   handlers/main.yml
#   templates/
#   files/
#   vars/main.yml
#   defaults/main.yml
#   meta/main.yml
```

### Role Task File

```yaml
# roles/webserver/tasks/main.yml
---
- name: Install httpd packages
  dnf:
    name:
      - httpd
      - mod_ssl
    state: present
  notify: restart httpd

- name: Create document root
  file:
    path: "{{ document_root }}"
    state: directory
    owner: apache
    group: apache
    mode: "0755"

- name: Deploy default page
  copy:
    src: index.html
    dest: "{{ document_root }}/index.html"
    owner: apache
    group: apache
    mode: "0644"

- name: Deploy httpd configuration
  template:
    src: httpd.conf.j2
    dest: /etc/httpd/conf.d/myapp.conf
    owner: root
    group: root
    mode: "0644"
    validate: "httpd -t -f %s"
  notify: restart httpd

- name: Open HTTP port in firewall
  firewalld:
    port: "{{ http_port }}/tcp"
    permanent: yes
    immediate: yes
    state: enabled

- name: Enable and start httpd
  systemd:
    name: httpd
    state: started
    enabled: yes
```

### Role Handlers

```yaml
# roles/webserver/handlers/main.yml
---
- name: restart httpd
  systemd:
    name: httpd
    state: restarted
```

### Role Defaults

```yaml
# roles/webserver/defaults/main.yml
---
document_root: /var/www/html
http_port: 80
app_name: myapp
```

### Role Variables (Higher Precedence)

```yaml
# roles/webserver/vars/main.yml
---
httpd_packages:
  - httpd
  - mod_ssl
```

### Role Metadata

```yaml
# roles/webserver/meta/main.yml
---
galaxy_info:
  author: RHCE Student
  description: Web server configuration role
  company: Example Corp
  license: MIT
  min_ansible_version: "2.14"
  platforms:
    - name: EL
      versions:
        - "9"
  galaxy_tags:
    - web
    - httpd
    - rhel

dependencies: []
```

### Using Roles in Playbooks

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: yes
  vars:
    document_root: /var/www/myapp
    http_port: 8080
  roles:
    - webserver

- name: Configure database servers
  hosts: dbservers
  become: yes
  roles:
    - database
```

### Role with Conditional Application

```yaml
- name: Conditional role application
  hosts: all
  become: yes
  roles:
    - role: webserver
      when: "'web' in groups"

    - role: database
      when: "'db' in groups"

    - role: monitoring
      tags: monitoring
```

### Installing Roles from Ansible Galaxy

```bash
# Search for roles
ansible-galaxy role search nginx

# Install a role
ansible-galaxy role install geerlingguy.nginx

# Install from requirements file
ansible-galaxy role install -r requirements.yml

# Remove a role
ansible-galaxy role remove geerlingguy.nginx
```

```yaml
# requirements.yml
---
- name: geerlingguy.nginx
  version: "3.0.0"
- name: geerlingguy.postgresql
  version: "3.2.0"
```

## Ansible Content Collections

### What Are Collections

Collections are distributable packages of Ansible content that can include modules, plugins, roles, and collections of content from specific providers.

### Managing Collections

```bash
# List installed collections
ansible-galaxy collection list

# Install a collection
ansible-galaxy collection install community.general
ansible-galaxy collection install community.mysql
ansible-galaxy collection install community.postgresql

# Install from requirements file
ansible-galaxy collection install -r collections-requirements.yml

# Remove a collection
ansible-galaxy collection remove community.general

# Search for collections
ansible-galaxy collection search community
```

### Using Collection Modules

```yaml
---
- name: Use collection modules
  hosts: all
  become: yes
  tasks:
    - name: Use fully qualified collection name
      community.general.sysctl:
        name: vm.swappiness
        value: "10"
        state: present
        sysctl_set: yes

    - name: Use short name if collection is loaded
      sysctl:
        name: net.ipv4.ip_forward
        value: "0"
        state: present
```

### Collection Requirements File

```yaml
# collections-requirements.yml
---
collections:
  - name: community.general
    version: ">=7.0.0"
  - name: community.mysql
    version: ">=3.7.0"
  - name: ansible.posix
    version: ">=1.5.0"
```

### Finding Modules in ansible-navigator

During the exam, use ansible-navigator to search for modules:

1. Open ansible-navigator: `ansible-navigator`
2. Navigate to Documentation browser
3. Search for modules by keyword
4. Review module parameters and examples
5. Copy parameter syntax directly from documentation

## Verification Procedures

### Verify Inventory Configuration

```bash
# List all hosts
ansible-inventory --list

# Show inventory tree
ansible-inventory --graph

# Check host variables
ansible-inventory --host servera.lab.example.com

# Ping all hosts
ansible all -m ping
```

### Verify Variable Resolution

```bash
# Debug variables during playbook run
ansible-playbook playbook.yml -e "test_var=value" -vv

# Check fact values
ansible all -m debug -a "msg={{ ansible_os_family }}"
ansible all -m debug -a "msg={{ ansible_default_ipv4.address }}"
```

### Verify Role Structure

```bash
# List installed roles
ansible-galaxy role list

# Verify role directory structure
tree roles/webserver/

# Test role execution
ansible-playbook -i inventory test_role.yml
```

### Verify Collection Installation

```bash
# List collections
ansible-galaxy collection list

# Verify module availability
ansible-doc community.general.sysctl
```

### Verify Configuration

```bash
# Show effective configuration
ansible-config dump

# Show specific setting
ansible-config dump | grep -i inventory
ansible-config dump | grep -i remote_user

# Verify configuration file location
ansible-config dump --only-changed
```

## Troubleshooting

### Connectivity Issues

```bash
# Test SSH connectivity
ansible all -m ping

# Test with verbose output
ansible all -m ping -vvv

# Check SSH key authentication
ssh ansible@servera.lab.example.com "whoami"

# Verify sudo access
ssh ansible@servera.lab.example.com "sudo whoami"
```

**Common causes:**
- SSH key not distributed to managed node
- Wrong username in inventory or ansible.cfg
- Firewall blocking SSH port
- SELinux blocking SSH connections

### Variable Issues

```bash
# Debug undefined variable
ansible-playbook playbook.yml -e "app_name=test" -vvv

# Check variable precedence
ansible-playbook playbook.yml -vvv 2>&1 | grep "app_name"

# Test variable in template
ansible localhost -m template -a "src=test.j2 dest=/dev/null"
```

**Common causes:**
- Variable name typo
- Variable defined at wrong scope
- Variable overridden by higher precedence
- Missing `default` filter for optional variables

### Module Errors

```bash
# Get module documentation
ansible-doc dnf

# Test module ad hoc
ansible all -m dnf -a "name=vim state=present" -vvv

# Check module version
ansible all -m debug -a "msg={{ ansible_python.version.full }}"
```

**Common causes:**
- Module parameter typo
- Module not available in installed collection
- Python version incompatibility on managed node
- Missing dependencies on managed node

### Playbook Syntax Errors

```bash
# Validate playbook syntax
ansible-playbook playbook.yml --syntax-check

# Validate YAML syntax
python3 -c "import yaml; yaml.safe_load(open('playbook.yml'))"
```

### Permission Errors

```bash
# Check become configuration
ansible all -m ping -b

# Verify sudoers configuration on managed node
ssh ansible@servera "sudo -l"

# Check file permissions
ansible all -m command -a "ls -la /etc/myapp/"
```

### Fact Gathering Failures

```bash
# Test fact gathering manually
ansible all -m setup

# Disable fact gathering if not needed
# Add gather_facts: no to play

# Check Python on managed node
ansible all -m command -a "python3 --version"
```

## Real-World Automation Examples

### Complete Infrastructure Setup

```yaml
---
# Play 1: Base configuration for all servers
- name: Base system configuration
  hosts: all
  become: yes
  gather_facts: yes
  vars_files:
    - vars/common.yml
  tasks:
    - name: Update all packages
      dnf:
        name: "*"
        state: latest
        update_cache: yes

    - name: Install common packages
      dnf:
        name: "{{ common_packages }}"
        state: present

    - name: Configure NTP
      systemd:
        name: chronyd
        state: started
        enabled: yes

    - name: Set hostname based on inventory
      hostname:
        name: "{{ inventory_hostname_short }}"

# Play 2: Web server configuration
- name: Web server configuration
  hosts: webservers
  become: yes
  vars_files:
    - vars/webserver.yml
  roles:
    - role: webserver
      document_root: /var/www/myapp
      http_port: 8080
  tags: webserver

# Play 3: Database server configuration
- name: Database server configuration
  hosts: dbservers
  become: yes
  vars_files:
    - vars/database.yml
  roles:
    - role: database
      db_port: 5432
  tags: database

# Play 4: Verification
- name: Verify services
  hosts: all
  gather_facts: no
  tasks:
    - name: Verify system is responsive
      ping:
```

### Multi-Environment Deployment

```yaml
---
- name: Deploy application
  hosts: "{{ target_environment }}"
  become: yes
  vars:
    app_name: myapp
  vars_files:
    - "vars/{{ target_environment }}.yml"
  pre_tasks:
    - name: Validate environment variable
      assert:
        that:
          - target_environment is defined
          - target_environment in ['development', 'staging', 'production']
        fail_msg: "Invalid environment: {{ target_environment }}"
  roles:
    - webserver
    - monitoring
  post_tasks:
    - name: Verify deployment
      uri:
        url: "http://localhost:{{ http_port }}/health"
        status_code: 200
      retries: 5
      delay: 3
```

Run with:
```bash
ansible-playbook deploy.yml -e "target_environment=production"
```

## RHCE Exam Notes

### Critical Exam Points

1. **Inventories are fundamental**: You will need to create and modify inventory files. Know both INI and YAML formats. Understand host variables, group variables, and nested groups.

2. **Variable precedence matters**: When variables conflict, the highest precedence wins. Role defaults have the lowest priority; extra vars on the command line have the highest.

3. **Facts are available automatically**: You do not need to manually gather facts unless `gather_facts: no` is set. Know the most common facts and their syntax.

4. **Loops reduce repetition**: Use `loop` for iterating over lists. Use dictionary loops for complex data structures.

5. **Conditionals enable flexibility**: Use `when` with facts, variables, and registered variables. Know comparison operators and boolean logic.

6. **Error handling prevents cascading failures**: Use `ignore_errors`, `failed_when`, `changed_when`, and `block/rescue/always` appropriately.

7. **Roles promote reusability**: Know how to create roles with `ansible-galaxy init` and use them in playbooks. Understand the role directory structure.

8. **ansible.cfg controls behavior**: Know how to create and configure `ansible.cfg`. Understand the search order for configuration files.

9. **Documentation is your friend**: The exam provides access to Ansible documentation. Know how to use `ansible-doc` and the ansible-navigator documentation browser.

10. **Collections extend functionality**: Know how to install collections and use fully qualified collection names (FQCN).

### Exam Tips

- Always use `--syntax-check` before running playbooks
- Use `--check --diff` to preview changes
- Use `-vvv` for detailed debugging output
- Save progress frequently and commit to Git
- Use ansible-navigator documentation browser to look up module parameters
- Test with ad hoc commands before writing full playbooks
- Keep playbooks simple and focused
- Use roles for repeated configurations

## Common Mistakes

### Mistake 1: Wrong Variable Precedence

```yaml
# WRONG - role defaults override play vars
# Role defaults have LOWEST priority, play vars win
# But extra vars on command line override everything

# CORRECT - understand precedence order
# Role defaults < role vars < play vars < extra vars
```

### Mistake 2: Forgetting Quotes on Octal Modes

```yaml
# WRONG - interpreted as decimal
mode: 0644

# CORRECT - string preserves octal
mode: "0644"
```

### Mistake 3: Using `shell` Instead of `command`

```yaml
# WRONG - unnecessary shell invocation
- name: List files
  shell: ls /tmp

# CORRECT - direct command execution
- name: List files
  command: ls /tmp
```

### Mistake 4: Missing `become: yes`

```yaml
# WRONG - will fail for root-only tasks
- name: Install package
  dnf:
    name: httpd
    state: present

# CORRECT - include become at play or task level
- name: Configure servers
  hosts: all
  become: yes
  tasks:
    - name: Install package
      dnf:
        name: httpd
        state: present
```

### Mistake 5: Incorrect `when` Syntax

```yaml
# WRONG - missing quotes around string comparison
when: ansible_os_family == RedHat

# CORRECT - quoted string
when: ansible_os_family == "RedHat"

# WRONG - incorrect boolean syntax
when: enable_httpd = true

# CORRECT - boolean variable reference
when: enable_httpd
```

### Mistake 6: Loop Variable Conflicts

```yaml
# WRONG - loop variable shadows existing variable
- name: Bad loop
  debug:
    msg: "{{ item }}"
  loop:
    - value1
    - value2
  vars:
    item: "original"  # this gets overwritten

# CORRECT - use loop_control to customize variable name
- name: Good loop
  debug:
    msg: "{{ pkg }}"
  loop:
    - value1
    - value2
  loop_control:
    loop_var: pkg
```

### Mistake 7: Not Using Handlers

```yaml
# WRONG - restarts service every run
- name: Deploy config
  copy:
    src: httpd.conf
    dest: /etc/httpd/conf/httpd.conf

- name: Restart httpd
  systemd:
    name: httpd
    state: restarted

# CORRECT - restarts only when config changes
- name: Deploy config
  copy:
    src: httpd.conf
    dest: /etc/httpd/conf/httpd.conf
  notify: restart httpd

handlers:
  - name: restart httpd
    systemd:
      name: httpd
      state: restarted
```

### Mistake 8: YAML Indentation Errors

```yaml
# WRONG - inconsistent indentation
- name: Task
  dnf:
  name: httpd    # wrong indentation
    state: present

# CORRECT - consistent 2-space indentation
- name: Task
  dnf:
    name: httpd
    state: present
```

## Chapter Summary

This chapter covered all core components of Ansible:

- **Inventories** define managed hosts and organize them into groups with INI or YAML format. Host and group variables provide per-target customization.
- **Modules** are the units of work executed on managed nodes. They are idempotent and return structured JSON. Use `ansible-doc` to look up module parameters.
- **Variables** inject dynamic values into playbooks. Understand precedence: role defaults (lowest) through extra vars (highest).
- **Facts** provide automatic system information. Use them for conditional logic and dynamic configuration.
- **Loops** eliminate task duplication. Use `loop` with lists and dictionaries.
- **Conditionals** with `when` enable intelligent task execution based on facts, variables, and registered outputs.
- **Plays** organize tasks into logical units targeting specific host groups. Multiple plays can chain in one playbook.
- **Error handling** includes `ignore_errors`, `failed_when`, `changed_when`, and `block/rescue/always` for structured error recovery.
- **Configuration files** (`ansible.cfg`) control global behavior including inventory location, remote user, privilege escalation, and SSH settings.
- **Roles** package automation into reusable, shareable units with a standard directory structure.
- **Collections** distribute modules, plugins, and roles from the Ansible community and vendors.

## Quick Reference

| Component | Command/Keyword | Purpose |
|---|---|---|
| Inventory check | `ansible-inventory --list` | List hosts and groups |
| Inventory graph | `ansible-inventory --graph` | Show group hierarchy |
| Module docs | `ansible-doc <module>` | View module documentation |
| Module search | `ansible-doc -l \| grep` | Search available modules |
| Module snippet | `ansible-doc -s <module>` | Show module parameters |
| Ping hosts | `ansible all -m ping` | Test connectivity |
| Gather facts | `ansible all -m setup` | View system facts |
| Syntax check | `ansible-playbook --syntax-check` | Validate playbook |
| Dry run | `ansible-playbook --check --diff` | Preview changes |
| Verbose output | `ansible-playbook -vvv` | Debug output |
| Extra vars | `-e "key=value"` | Pass variables |
| Create role | `ansible-galaxy init <name>` | Generate role structure |
| Install role | `ansible-galaxy role install` | Install from Galaxy |
| Install collection | `ansible-galaxy collection install` | Install collection |
| List collections | `ansible-galaxy collection list` | View installed collections |
| Config dump | `ansible-config dump` | Show effective config |
| Loop keyword | `loop:` | Iterate over items |
| Conditional | `when:` | Conditional execution |
| Error ignore | `ignore_errors: yes` | Continue on failure |
| Custom failure | `failed_when:` | Define failure condition |
| No change report | `changed_when: false` | Suppress change report |
| Error handling | `block/rescue/always` | Try-catch-finally pattern |
| Privilege | `become: yes` | Escalate to root |
| Delegate | `delegate_to:` | Run task on different host |
| Serial | `serial: 1` | Limit parallel execution |
| Tags | `tags:` | Selective task execution |

## Review Questions

1. What is the order in which Ansible searches for the `ansible.cfg` configuration file?

2. What is the difference between `loop` and `with_items`?

3. How does variable precedence work between role defaults, play variables, and extra variables passed on the command line?

4. What is the purpose of `gather_facts: no` in a play, and when would you use it?

5. Explain the difference between `ignore_errors`, `failed_when`, and `changed_when`.

6. What are the three directories where host and group variables can be stored, and how do they differ?

7. What is the purpose of the `block/rescue/always` structure in Ansible?

8. How do you install an Ansible collection, and how do you reference a module from a collection in a playbook?

9. What is the difference between a play and a playbook?

10. How would you create a new role and what directories does `ansible-galaxy init` create?

## Answers

1. Ansible searches for `ansible.cfg` in this order: (1) the `ANSIBLE_CONFIG` environment variable, (2) `ansible.cfg` in the current working directory, (3) `~/.ansible.cfg` in the user's home directory, (4) `/etc/ansible/ansible.cfg` system-wide. The first file found is used.

2. `loop` is the modern, preferred syntax for iterating over items. `with_items` is the legacy dynamic loop syntax that predates `loop`. Both produce the same result, but `loop` is simpler and more consistent. The `with_` prefix loops (like `with_first_found`, `with_together`) still require the `with_` syntax.

3. Role defaults have the lowest priority and can be overridden by any other variable source. Play variables (defined in the `vars` section of a play) override role defaults. Extra variables passed with `-e` on the command line have the highest priority and override all other variable sources. The full order is: role defaults < inventory vars < play vars < extra vars.

4. `gather_facts: no` disables automatic fact collection at the start of a play. Use it when you do not need system information (facts) to make decisions in your tasks. This speeds up playbook execution, especially when managing many hosts. Facts take time to collect, so disabling them saves time when they are not required.

5. `ignore_errors: yes` tells Ansible to continue playbook execution even if the task fails, treating the failure as non-fatal. `failed_when` customizes the condition under which a task is considered failed, allowing you to define failure beyond just a non-zero return code. `changed_when` controls whether a task reports a state change, which is important for read-only operations that should never trigger handlers or report modifications.

6. Host variables are stored in `host_vars/<hostname>.yml` and apply only to that specific host. Group variables are stored in `group_vars/<groupname>.yml` and apply to all hosts in that group. Variables for all hosts go in `group_vars/all.yml`. These directories must be located next to the inventory file. Host variables override group variables for the same key.

7. The `block/rescue/always` structure provides structured error handling similar to try-catch-finally in programming languages. Tasks in `block` are the main operations. If any task in the block fails, tasks in `rescue` execute for error recovery or rollback. Tasks in `always` execute regardless of whether the block succeeded or failed, making them suitable for cleanup or logging.

8. Install a collection with `ansible-galaxy collection install <namespace>.<collection>`, for example `ansible-galaxy collection install community.general`. Reference a module using its fully qualified collection name (FQCN): `community.general.sysctl`. If the collection is in the default paths, you can also use the short name `sysctl`, but FQCN is more explicit and avoids ambiguity.

9. A play is a single unit within a playbook that targets a group of hosts and executes a list of tasks. A playbook is a YAML file containing one or more plays. A playbook is the file; a play is a section within that file. Each play can target different hosts, use different variables, and execute different tasks.

10. Run `ansible-galaxy init <role_name>` to create a new role. This creates a directory with the role name containing subdirectories: `tasks/` (main task list), `handlers/` (handlers), `templates/` (Jinja2 templates), `files/` (static files), `vars/` (role variables), `defaults/` (default variables), `meta/` (role metadata), and `tests/` (role tests). Each directory contains a `main.yml` file (except `tests/` which contains `test.yml` and `inventory`).
