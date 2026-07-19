# Chapter 3: Configure Ansible


> **2026 Review / Alignment Note**
>
> This chapter was reviewed and enhanced for the current Red Hat EX294 direction: performance-based Ansible automation for system-administration tasks using Red Hat Ansible Automation Platform. Use this material as a self-contained study manuscript, but keep the project source policy strict: official Red Hat EX294 objectives, RHEL 10 documentation, Red Hat Ansible Automation Platform documentation, and version-pinned ansible-core documentation override older local content.
>
> In this reviewed edition, examples prefer clear YAML, repeatable state, explicit verification, and production-safe thinking. Where a command or module can vary by the exam environment or execution environment image, the chapter tells you how to verify it with `ansible-doc`, `ansible-navigator doc`, `ansible-config`, `ansible-navigator config`, or the locally available product documentation.


## How to Read Ansible YAML Naturally

Read every task as a plain English sentence:

```yaml
- name: Ensure httpd is installed
  ansible.builtin.dnf:
    name: httpd
    state: present
```

This means:

- `- name:` is the human sentence explaining the action.
- `ansible.builtin.dnf:` is the tool Ansible will use.
- `name: httpd` is the target package.
- `state: present` means the desired final state is “installed”.

The easiest mental model is:

```text
host group + privilege + ordered tasks + desired states + verification
```

So when you read a playbook, ask five questions:

1. **Where will this run?** Check `hosts:`.
2. **Does it need root?** Check `become:`.
3. **What state is required?** Check `state:`, `enabled:`, `mode:`, `owner:`, `group:`, `path:`, and `src:`.
4. **What changes when input changes?** Check variables, facts, loops, and conditionals.
5. **How do I prove it worked?** Check verification tasks or commands.

YAML indentation is not decoration. It shows ownership. A value indented under a module belongs to that module. A value indented under a task belongs to that task. A value aligned with `tasks:` belongs to the play.


## Exam-Safe Writing Rules for This Chapter

Use these rules whenever you create your own playbooks:

- Prefer FQCNs such as `ansible.builtin.copy`, `ansible.builtin.template`, and `ansible.builtin.command` in study examples. Short names often work, but FQCNs remove ambiguity and make documentation lookup easier.
- Prefer `become: true` at the play level for system-administration tasks unless a task must intentionally run as the remote user.
- Prefer `state: present` for package installation unless the requirement explicitly says to update to the newest available version.
- Prefer modules over shell commands when a reliable module exists.
- Use `ansible.builtin.command` instead of `ansible.builtin.shell` unless shell features such as pipes, redirects, globbing, or environment expansion are required.
- Treat `ignore_errors: true` as a last resort. Prefer `failed_when`, `changed_when`, or `block` / `rescue` / `always` when you need controlled failure handling.
- Do not assume a collection exists. Verify with `ansible-navigator collections`, `ansible-galaxy collection list`, or `ansible-doc -l` in the environment provided to you.
- When using automation execution environments, keep `ansible.cfg`, inventory, playbooks, roles, templates, and project-adjacent collections inside the project directory so navigator can mount and find them cleanly.
- Every task should be repeatable. Running the same playbook twice should normally result in `ok` rather than unnecessary `changed` output.
- Every important state change needs verification.


## Added Review: Configuration Files Must Be Local, Predictable, and Visible

For exam and daily work, put the main project files together:

```text
project/
├── ansible.cfg
├── ansible-navigator.yml
├── inventory
├── playbooks/
├── roles/
├── group_vars/
├── host_vars/
├── templates/
└── files/
```

This avoids the most common problem: Ansible reading a different configuration file than the one you think it is reading.

Always verify configuration with:

```bash
ansible-config dump --only-changed
ansible-navigator config -m stdout
```

### Safe `ansible.cfg` baseline

```ini
[defaults]
inventory = ./inventory
remote_user = ansible
host_key_checking = False
roles_path = ./roles
collections_path = ./collections:~/.ansible/collections:/usr/share/ansible/collections
stdout_callback = yaml

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
pipelining = True
```

Do not blindly copy a large `ansible.cfg`. Add only settings you understand and can verify.


## Learning Objectives

By the end of this chapter, you will be able to:

- Create and customize `ansible.cfg` for different environments
- Configure `ansible-navigator.yml` for optimal exam workflow
- Create static host inventory files in INI and YAML format
- Define host groups, nested groups, and group variables in inventories
- Configure managed nodes for Ansible automation
- Generate and distribute SSH keys to managed nodes
- Configure privilege escalation on managed nodes
- Deploy files to managed nodes using Ansible modules
- Run playbooks with both `ansible-navigator` and `ansible-playbook`
- Use ansible-navigator to search for and discover new modules

## Concept Overview

Before writing playbooks, Ansible must be properly configured. Configuration establishes how the control node communicates with managed nodes, what defaults apply globally, and how the automation environment behaves. Poor configuration leads to connection failures, permission errors, and inconsistent behavior.

### Why Configuration Matters

Every Ansible deployment requires:

- **Inventory** — which hosts to manage
- **Authentication** — how to reach those hosts securely
- **Privilege escalation** — how to perform root-level tasks
- **Global defaults** — behavior that applies across all playbooks
- **Navigation interface** — ansible-navigator for the exam environment

The RHCE exam expects you to configure all of these from scratch in the provided environment.

### How Ansible Configuration Works

Ansible reads configuration from multiple sources in a defined order. Understanding this order prevents unexpected behavior when settings conflict. The configuration hierarchy is:

1. Environment variables (highest priority)
2. `ansible.cfg` in current directory
3. `~/.ansible.cfg` in home directory
4. `/etc/ansible/ansible.cfg` system-wide (lowest priority)

## Ansible Fundamentals Required

### The Control Node

The control node is the machine where Ansible is installed and where playbooks are executed. All configuration files, inventories, and playbooks reside on the control node.

### Managed Nodes

Managed nodes are remote systems configured by Ansible. They require:

- SSH access from the control node
- Python 3 installed
- Sudo access for the Ansible user
- No Ansible software installed (agentless architecture)

### Communication Flow

```
Control Node                          Managed Node
─────────────                         ────────────
ansible-playbook playbook.yml
         │
         ├── SSH connection (key-based)
         │
         ├── Transfer module payload
         │
         ├── Execute module as root (via sudo)
         │
         └── Receive JSON result
```

## Commands and Tools

```bash
# View effective Ansible configuration
ansible-config dump
ansible-config dump --only-changed

# Validate configuration
ansible-config list

# Test connectivity
ansible all -m ping
ansible all -m ping -vvv

# Verify SSH key authentication
ansible all -m raw -a "whoami"

# Check privilege escalation
ansible all -m ping -b
```

## Creating and Modifying `ansible.cfg`

### Configuration File Structure

The `ansible.cfg` file uses INI format with named sections. Each section controls a different aspect of Ansible behavior.

```ini
# ansible.cfg

[defaults]
# Path to inventory file
inventory = /home/ansible/inventory

# Remote user for SSH connections
remote_user = ansible

# Disable SSH host key checking
host_key_checking = false

# Default privilege escalation
become = true

# Logging
log_path = /home/ansible/ansible.log

# Number of parallel connections
forks = 10

# SSH timeout in seconds
timeout = 30

# Collections search paths
collections_paths = /home/ansible/.ansible/collections:/usr/share/ansible/collections

# Enable fact caching for performance
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts_cache

# Callback plugins for additional output
callbacks_enabled = timer, profile_tasks

[privilege_escalation]
# Method for privilege escalation
become_method = sudo

# User to escalate to
become_user = root

# Do not prompt for sudo password
become_ask_pass = false

[ssh_connection]
# Enable SSH pipelining for performance
pipelining = true

# SSH connection arguments
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no

# Private key file
private_key_file = /home/ansible/.ssh/id_ed25519
```

### Essential `ansible.cfg` Settings for the Exam

```ini
[defaults]
inventory = /home/ansible/inventory
remote_user = ansible
host_key_checking = false
log_path = /home/ansible/ansible.log
forks = 10
timeout = 30
collections_paths = /home/ansible/.ansible/collections:/usr/share/ansible/collections

[privilege_escalation]
become = true
become_method = sudo
become_user = root
become_ask_pass = false

[ssh_connection]
pipelining = true
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

### Section-by-Section Explanation

#### `[defaults]` Section

| Parameter | Purpose | Exam Relevance |
|---|---|---|
| `inventory` | Path to inventory file | Required to define managed hosts |
| `remote_user` | SSH username | Must match the user on managed nodes |
| `host_key_checking` | SSH host key verification | Set to `false` in exam environment |
| `become` | Default privilege escalation | `true` for most tasks |
| `log_path` | Ansible log file location | Useful for troubleshooting |
| `forks` | Parallel connections | Default 5; increase for more hosts |
| `timeout` | SSH connection timeout | Increase for slow networks |
| `collections_paths` | Where to find collections | Must include exam collection paths |
| `fact_caching` | Cache gathered facts | Improves performance on repeated runs |
| `callbacks_enabled` | Additional output plugins | `timer` shows task duration |

#### `[privilege_escalation]` Section

| Parameter | Purpose | Exam Relevance |
|---|---|---|
| `become_method` | Escalation method | `sudo` for RHEL systems |
| `become_user` | Target user | `root` for system administration |
| `become_ask_pass` | Prompt for password | `false` for non-interactive runs |

#### `[ssh_connection]` Section

| Parameter | Purpose | Exam Relevance |
|---|---|---|
| `pipelining` | Reduce SSH round trips | `true` for better performance |
| `ssh_args` | Extra SSH options | ControlMaster for connection reuse |
| `private_key_file` | SSH private key path | Must match key distributed to nodes |

### Creating `ansible.cfg` Programmatically

```yaml
---
- name: Configure Ansible control node
  hosts: localhost
  connection: local
  become: true
  tasks:
    - name: Create ansible.cfg
      copy:
        content: |
          [defaults]
          inventory = /home/ansible/inventory
          remote_user = ansible
          host_key_checking = false
          log_path = /home/ansible/ansible.log
          forks = 10
          timeout = 30
          collections_paths = /home/ansible/.ansible/collections:/usr/share/ansible/collections

          [privilege_escalation]
          become = true
          become_method = sudo
          become_user = root
          become_ask_pass = false

          [ssh_connection]
          pipelining = true
          ssh_args = -o ControlMaster=auto -o ControlPersist=60s
        dest: /home/ansible/ansible.cfg
        owner: ansible
        group: ansible
        mode: "0644"
```

### Verifying `ansible.cfg` Configuration

```bash
# Show all effective settings
ansible-config dump

# Show only non-default settings
ansible-config dump --only-changed

# Verify specific setting
ansible-config dump | grep -i inventory
ansible-config dump | grep -i remote_user
ansible-config dump | grep -i host_key_checking

# Test configuration with a simple task
ansible all -m ping
```

## Modifying `ansible-navigator.yml`

### What Is ansible-navigator

ansible-navigator is a text-based user interface (TUI) for Ansible. It provides:

- Playbook execution with real-time output
- Documentation browser for modules and collections
- Inventory editor
- Configuration editor
- Job event viewer

### ansible-navigator Configuration File

```yaml
# ansible-navigator.yml
ansible-navigator:
  # Execution environment settings
  execution-environment:
    enabled: false

  # Display mode
  mode: stdout

  # Ansible settings
  ansible:
    diff: true
    inventory:
      - /home/ansible/inventory

  # Logging configuration
  logging:
    level: warning

  # Editor settings
  editor:
    command: code --wait
    command-timeout: 120

  # Display settings
  display:
    color: true
```

### Configuration Options Explained

#### Execution Environment

```yaml
execution-environment:
  enabled: false
```

The execution environment (EE) is a containerized Ansible runtime. In the exam environment, set `enabled: false` to use the system-installed Ansible directly.

#### Mode

```yaml
mode: stdout
```

- `stdout` — displays output in the terminal (recommended for exam)
- `interactive` — full TUI with menus and panels

#### Ansible Settings

```yaml
ansible:
  diff: true
  inventory:
    - /home/ansible/inventory
```

Setting `diff: true` shows before/after changes when running playbooks. The `inventory` list specifies which inventory files to load.

#### Logging

```yaml
logging:
  level: warning
```

Log levels from most to least verbose: `debug`, `info`, `warning`, `error`, `critical`. Use `warning` for the exam to reduce noise while capturing important messages.

### Creating `ansible-navigator.yml`

```bash
# Create ansible-navigator configuration
cat > /home/ansible/ansible-navigator.yml << 'EOF'
ansible-navigator:
  execution-environment:
    enabled: false
  mode: stdout
  ansible:
    diff: true
    inventory:
      - /home/ansible/inventory
  logging:
    level: warning
EOF
```

```yaml
---
- name: Configure ansible-navigator
  hosts: localhost
  connection: local
  tasks:
    - name: Create ansible-navigator.yml
      copy:
        content: |
          ansible-navigator:
            execution-environment:
              enabled: false
            mode: stdout
            ansible:
              diff: true
              inventory:
                - /home/ansible/inventory
            logging:
              level: warning
        dest: /home/ansible/ansible-navigator.yml
        owner: ansible
        group: ansible
        mode: "0644"
```

### Using ansible-navigator

```bash
# Run a playbook
ansible-navigator run playbook.yml

# Run with specific inventory
ansible-navigator run playbook.yml --inventory /path/to/inventory

# Run in interactive mode
ansible-navigator run playbook.yml --mode interactive

# Open documentation browser
ansible-navigator

# Run ad hoc command
ansible-navigator execute all -m ping

# View configuration
ansible-navigator configuration
```

## Creating Static Host Inventory Files

### INI Format Inventory

The INI format is the most common inventory format used in the RHCE exam.

```ini
# /home/ansible/inventory

# Ungrouped hosts
servera.lab.example.com
serverb.lab.example.com

# Web server group
[webservers]
servera.lab.example.com
serverb.lab.example.com

# Database server group
[dbservers]
serverc.lab.example.com

# Application server group
[appservers]
serverd.lab.example.com

# Nested group combining multiple groups
[production:children]
webservers
dbservers
appservers

# Group variables
[webservers:vars]
http_port=8080
app_name=myapp

[dbservers:vars]
db_port=5432
db_name=production

# Host variables inline
[workstations]
workstationa ansible_user=admin ansible_port=22
```

### YAML Format Inventory

```yaml
# /home/ansible/inventory.yml
all:
  children:
    webservers:
      hosts:
        servera.lab.example.com:
          http_port: 8080
          app_name: myapp
        serverb.lab.example.com:
          http_port: 8443
          app_name: myapp
      vars:
        http_packages:
          - httpd
          - mod_ssl

    dbservers:
      hosts:
        serverc.lab.example.com:
          db_port: 5432
          db_name: production
      vars:
        db_packages:
          - mariadb-server
          - mariadb

    appservers:
      hosts:
        serverd.lab.example.com:
          app_port: 8080

    production:
      children:
        - webservers
        - dbservers
        - appservers
      vars:
        environment: production
        log_level: warning
```

### Creating Inventory Programmatically

```yaml
---
- name: Create static inventory
  hosts: localhost
  connection: local
  tasks:
    - name: Create inventory directory
      file:
        path: /home/ansible/inventories
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create production inventory
      copy:
        content: |
          [webservers]
          servera.lab.example.com
          serverb.lab.example.com

          [dbservers]
          serverc.lab.example.com

          [production:children]
          webservers
          dbservers

          [webservers:vars]
          http_port=8080
          app_name=myapp

          [dbservers:vars]
          db_port=5432
        dest: /home/ansible/inventories/production
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create development inventory
      copy:
        content: |
          [webservers]
          devservera.lab.example.com

          [dbservers]
          devserverb.lab.example.com

          [webservers:vars]
          http_port=8080
          app_name=myapp-dev
          debug_mode=true

          [dbservers:vars]
          db_port=5432
          debug_mode=true
        dest: /home/ansible/inventories/development
        owner: ansible
        group: ansible
        mode: "0644"
```

### Using Multiple Inventory Sources

```ini
# ansible.cfg
[defaults]
# Comma-separated inventory sources
inventory = /home/ansible/inventories/production,/home/ansible/inventories/development
```

```bash
# Or specify on command line
ansible-playbook playbook.yml -i /home/ansible/inventories/production
ansible-playbook playbook.yml -i inventories/production,inventories/development
```

### Verifying Inventory Configuration

```bash
# List all hosts and groups
ansible-inventory --list

# Show group hierarchy
ansible-inventory --graph

# Get variables for a specific host
ansible-inventory --host servera.lab.example.com

# List hosts in a specific group
ansible webservers --list-hosts

# Test connectivity to all hosts
ansible all -m ping

# Test connectivity to specific group
ansible webservers -m ping
```

### Inventory Best Practices

```ini
# Good: Organized inventory with clear group structure
[webservers]
servera.lab.example.com
serverb.lab.example.com

[dbservers]
serverc.lab.example.com

[production:children]
webservers
dbservers

# Good: Use group variables for shared settings
[webservers:vars]
http_port=8080

# Good: Use host variables for unique settings
[workstations]
workstationa ansible_user=admin

# Bad: Mixing concerns
[all_servers]
servera ansible_user=admin http_port=8080 role=web db_port=5432
```

## Configuring Ansible Managed Nodes

### Managed Node Requirements

Each managed node must have:

1. SSH server running and accessible
2. Python 3 installed
3. An Ansible user with SSH key access
4. Sudo privileges for the Ansible user
5. Firewall allowing SSH connections from the control node

### Configuring Managed Nodes with Ansible

```yaml
---
- name: Configure managed nodes
  hosts: all
  become: true
  tasks:
    - name: Ensure SSH server is installed and running
      dnf:
        name: openssh-server
        state: present
      notify: restart sshd

    - name: Ensure Python 3 is installed
      dnf:
        name: python3
        state: present

    - name: Ensure required Python modules are installed
      dnf:
        name:
          - python3-libselinux
          - python3-firewall
          - python3-cryptography
        state: present

    - name: Ensure firewall allows SSH from control node
      firewalld:
        port: "22/tcp"
        permanent: yes
        immediate: yes
        state: enabled

    - name: Ensure SELinux is not blocking Ansible
      seboolean:
        name: sshd_sysadm_login
        state: yes
        persistent: yes

  handlers:
    - name: restart sshd
      systemd:
        name: sshd
        state: restarted
```

## Creating and Distributing SSH Keys

### Generating SSH Keys on the Control Node

```bash
# Generate Ed25519 key pair (recommended)
ssh-keygen -t ed25519 -C "ansible-control-node" -N "" -f /home/ansible/.ssh/id_ed25519

# Generate RSA key pair (alternative)
ssh-keygen -t rsa -b 4096 -C "ansible-control-node" -N "" -f /home/ansible/.ssh/id_rsa
```

### Distributing SSH Keys to Managed Nodes

#### Manual Distribution

```bash
# Copy public key to managed node
ssh-copy-id -i /home/ansible/.ssh/id_ed25519.pub ansible@servera.lab.example.com

# Verify key-based authentication
ssh ansible@servera.lab.example.com "echo connection successful"

# Verify sudo access
ssh ansible@servera.lab.example.com "sudo whoami"
```

#### Automated Distribution with Ansible

```yaml
---
- name: Distribute SSH keys and configure managed nodes
  hosts: all
  become: true
  vars:
    ansible_user_name: ansible
    ssh_key_path: /home/ansible/.ssh/id_ed25519.pub
  tasks:
    - name: Create Ansible user
      user:
        name: "{{ ansible_user_name }}"
        state: present
        createhome: yes
        shell: /bin/bash
        groups: wheel

    - name: Create .ssh directory
      file:
        path: "/home/{{ ansible_user_name }}/.ssh"
        state: directory
        owner: "{{ ansible_user_name }}"
        group: "{{ ansible_user_name }}"
        mode: "0700"

    - name: Deploy authorized key
      authorized_key:
        user: "{{ ansible_user_name }}"
        state: present
        key: "{{ lookup('file', ssh_key_path) }}"
        manage_dir: yes

    - name: Configure sudoers for Ansible user
      copy:
        content: "{{ ansible_user_name }} ALL=(ALL) NOPASSWD: ALL"
        dest: "/etc/sudoers.d/{{ ansible_user_name }}"
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"

    - name: Ensure SSH server is running
      systemd:
        name: sshd
        state: started
        enabled: yes
```

### SSH Key Distribution Playbook

```yaml
---
- name: Set up SSH key authentication
  hosts: all
  become: true
  vars:
    control_node_key: "{{ lookup('file', '/home/ansible/.ssh/id_ed25519.pub') }}"
  tasks:
    - name: Ensure ansible user exists
      user:
        name: ansible
        state: present
        createhome: yes
        shell: /bin/bash

    - name: Add ansible user to wheel group
      user:
        name: ansible
        groups: wheel
        append: yes

    - name: Create SSH directory for ansible user
      file:
        path: /home/ansible/.ssh
        state: directory
        owner: ansible
        group: ansible
        mode: "0700"

    - name: Deploy SSH public key
      authorized_key:
        user: ansible
        state: present
        key: "{{ control_node_key }}"
        exclusive: no

    - name: Set correct permissions on authorized_keys
      file:
        path: /home/ansible/.ssh/authorized_keys
        owner: ansible
        group: ansible
        mode: "0600"

    - name: Configure passwordless sudo
      copy:
        content: "ansible ALL=(ALL) NOPASSWD: ALL"
        dest: /etc/sudoers.d/ansible
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"

    - name: Ensure SSH is running
      systemd:
        name: sshd
        state: started
        enabled: yes
```

### Verifying SSH Key Distribution

```bash
# Test SSH connectivity
ssh ansible@servera.lab.example.com "whoami"

# Test Ansible ping
ansible all -m ping

# Test privilege escalation
ansible all -m ping -b

# Test with verbose output
ansible all -m ping -vvv

# Check authorized keys on managed node
ssh ansible@servera.lab.example.com "cat ~/.ssh/authorized_keys"

# Check sudoers configuration
ssh ansible@servera.lab.example.com "sudo -l"
```

## Configuring Privilege Escalation on Managed Nodes

### Sudoers Configuration

The Ansible user must have passwordless sudo access on every managed node.

```bash
# Manual configuration on managed node
echo "ansible ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/ansible
sudo visudo -c -f /etc/sudoers.d/ansible
sudo chmod 0440 /etc/sudoers.d/ansible
```

### Automated Privilege Escalation Configuration

```yaml
---
- name: Configure privilege escalation
  hosts: all
  become: true
  tasks:
    - name: Ensure ansible user is in wheel group
      user:
        name: ansible
        groups: wheel
        append: yes
        state: present

    - name: Configure NOPASSWD sudo for ansible user
      copy:
        content: "ansible ALL=(ALL) NOPASSWD: ALL"
        dest: /etc/sudoers.d/ansible
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"

    - name: Configure NOPASSWD sudo for wheel group
      lineinfile:
        path: /etc/sudoers
        regexp: "^%wheel"
        line: "%wheel ALL=(ALL) NOPASSWD: ALL"
        state: present
        validate: "visudo -cf %s"
        backup: yes

    - name: Disable password aging for ansible user
      user:
        name: ansible
        password_expire: never
```

### Privilege Escalation Options in `ansible.cfg`

```ini
[privilege_escalation]
# Enable privilege escalation by default
become = true

# Use sudo for escalation
become_method = sudo

# Escalate to root user
become_user = root

# Do not prompt for password
become_ask_pass = false

# Alternative: use su instead of sudo
# become_method = su

# Alternative: escalate to specific user
# become_user = apache
```

### Per-Task Privilege Escalation

```yaml
---
- name: Mixed privilege tasks
  hosts: all
  tasks:
    - name: Task requiring root
      become: true
      dnf:
        name: httpd
        state: present

    - name: Task as regular user
      become: no
      command: whoami
      register: current_user
      changed_when: false

    - name: Task as specific user
      become: true
      become_user: apache
      command: whoami
      register: apache_user
      changed_when: false

    - name: Display results
      debug:
        msg: "Root task user: {{ current_user.stdout }}, Apache task user: {{ apache_user.stdout }}"
```

### Verifying Privilege Escalation

```bash
# Test basic ping
ansible all -m ping

# Test with privilege escalation
ansible all -m ping -b

# Test sudo access
ansible all -m command -a "whoami" -b

# Test specific become user
ansible all -m command -a "whoami" -b --become-user=apache

# Verify sudoers file
ansible all -m command -a "sudo -l"
```

## Deploying Files to Managed Nodes

### Using the `copy` Module

```yaml
---
- name: Deploy files to managed nodes
  hosts: all
  become: true
  tasks:
    - name: Create target directory
      file:
        path: /etc/myapp
        state: directory
        owner: root
        group: root
        mode: "0755"

    - name: Deploy configuration file from control node
      copy:
        src: files/myapp.conf
        dest: /etc/myapp/myapp.conf
        owner: root
        group: root
        mode: "0644"
        backup: yes

    - name: Deploy file with inline content
      copy:
        content: |
          [application]
          name = myapp
          port = 8080
          log_level = info
        dest: /etc/myapp/settings.ini
        owner: root
        group: root
        mode: "0644"

    - name: Deploy multiple files
      copy:
        src: "{{ item.src }}"
        dest: "{{ item.dest }}"
        owner: root
        group: root
        mode: "0644"
      loop:
        - { src: files/config1.conf, dest: /etc/myapp/config1.conf }
        - { src: files/config2.conf, dest: /etc/myapp/config2.conf }
```

### Using the `template` Module

```yaml
---
- name: Deploy templated configuration files
  hosts: all
  become: true
  vars:
    app_port: 8080
    app_name: myapp
    log_level: info
  tasks:
    - name: Deploy application configuration
      template:
        src: templates/myapp.conf.j2
        dest: /etc/myapp/myapp.conf
        owner: root
        group: root
        mode: "0644"
        backup: yes
      notify: restart myapp

  handlers:
    - name: restart myapp
      systemd:
        name: myapp
        state: restarted
```

```yaml
# templates/myapp.conf.j2
# Application configuration for {{ inventory_hostname }}
# Managed by Ansible - Do not edit manually

[application]
name = {{ app_name }}
port = {{ app_port }}
log_level = {{ log_level }}
server_name = {{ inventory_hostname }}
environment = {{ environment | default('production') }}

[logging]
log_file = /var/log/{{ app_name }}/app.log
max_size = {{ log_max_size | default('100M') }}
rotate_count = {{ log_rotate_count | default(7) }}

{% if enable_debug | default(false) %}
[debug]
enabled = true
trace_requests = true
{% endif %}
```

### Using the `fetch` Module

```yaml
---
- name: Fetch files from managed nodes
  hosts: all
  become: true
  tasks:
    - name: Fetch log files for analysis
      fetch:
        src: /var/log/messages
        dest: /home/ansible/logs/{{ inventory_hostname }}/messages
        flat: no
        validate_checksum: yes

    - name: Fetch configuration backup
      fetch:
        src: /etc/myapp/myapp.conf
        dest: /home/ansible/backups/{{ inventory_hostname }}/myapp.conf
        flat: yes
```

### Deploying Scripts

```yaml
---
- name: Deploy and execute scripts
  hosts: all
  become: true
  tasks:
    - name: Deploy setup script
      copy:
        content: |
          #!/bin/bash
          set -euo pipefail
          echo "Setting up {{ inventory_hostname }}"
          dnf update -y
          echo "Setup complete"
        dest: /usr/local/bin/setup.sh
        owner: root
        group: root
        mode: "0755"

    - name: Execute setup script
      command: /usr/local/bin/setup.sh
      register: setup_result
      changed_when: false

    - name: Display script output
      debug:
        var: setup_result.stdout_lines
```

## Running Playbooks

### Running with `ansible-playbook`

```bash
# Basic playbook execution
ansible-playbook /home/ansible/playbooks/setup.yml

# Run with specific inventory
ansible-playbook -i /home/ansible/inventory playbooks/setup.yml

# Run with extra variables
ansible-playbook playbooks/setup.yml -e "app_name=myapp http_port=8080"

# Run with variable file
ansible-playbook playbooks/setup.yml -e "@vars/production.yml"

# Dry run (check mode)
ansible-playbook playbooks/setup.yml --check

# Dry run with diff output
ansible-playbook playbooks/setup.yml --check --diff

# Syntax validation only
ansible-playbook playbooks/setup.yml --syntax-check

# Verbose output
ansible-playbook playbooks/setup.yml -v
ansible-playbook playbooks/setup.yml -vv
ansible-playbook playbooks/setup.yml -vvv

# Limit to specific hosts
ansible-playbook playbooks/setup.yml --limit servera.lab.example.com
ansible-playbook playbooks/setup.yml --limit webservers

# Run specific tags
ansible-playbook playbooks/setup.yml --tags "packages,firewall"

# Skip specific tags
ansible-playbook playbooks/setup.yml --skip-tags "reboot"

# Start at specific task
ansible-playbook playbooks/setup.yml --start-at-task "Install packages"

# Run with specific forks
ansible-playbook playbooks/setup.yml -f 20

# Run specific play
ansible-playbook playbooks/setup.yml --limit webservers
```

### Running with `ansible-navigator`

```bash
# Basic playbook execution
ansible-navigator run /home/ansible/playbooks/setup.yml

# Run with inventory specification
ansible-navigator run playbooks/setup.yml --inventory /home/ansible/inventory

# Run with extra variables
ansible-navigator run playbooks/setup.yml --mode stdout -e "app_name=myapp"

# Run in interactive mode
ansible-navigator run playbooks/setup.yml --mode interactive

# Run with specific playbook directory
ansible-navigator run --playbook-dir /home/ansible/playbooks

# Open ansible-navigator without running a playbook
ansible-navigator

# Execute ad hoc command via navigator
ansible-navigator execute all -m ping

# View documentation
ansible-navigator documentation
```

### ansible-navigator Playbook Execution Workflow

```yaml
---
# Example playbook for ansible-navigator execution
- name: Configure web server
  hosts: webservers
  become: true
  vars:
    http_port: 8080
    app_name: myapp
  tasks:
    - name: Install httpd
      dnf:
        name:
          - httpd
          - mod_ssl
        state: present
      tags: packages
      notify: restart httpd

    - name: Deploy configuration
      template:
        src: httpd.conf.j2
        dest: /etc/httpd/conf.d/myapp.conf
        owner: root
        group: root
        mode: "0644"
      tags: config
      notify: restart httpd

    - name: Open firewall port
      firewalld:
        port: "{{ http_port }}/tcp"
        permanent: yes
        immediate: yes
        state: enabled
      tags: firewall

    - name: Start httpd
      systemd:
        name: httpd
        state: started
        enabled: yes
      tags: services

  handlers:
    - name: restart httpd
      systemd:
        name: httpd
        state: restarted
```

```bash
# Run full playbook
ansible-navigator run playbook.yml

# Run only package installation tasks
ansible-navigator run playbook.yml --tags packages

# Run only config and firewall tasks
ansible-navigator run playbook.yml --tags config,firewall

# Skip reboot tasks
ansible-navigator run playbook.yml --skip-tags reboot
```

### Using ansible-navigator to Find Modules

```bash
# Open ansible-navigator
ansible-navigator

# Navigate to documentation browser
# Use arrow keys to browse module documentation
# Search for modules by keyword:
#   - Type / to enter search mode
#   - Enter module name or keyword
#   - Press Enter to search

# From command line, search for module documentation
ansible-doc -l | grep -i firewall
ansible-doc firewalld
ansible-doc -s firewalld
```

### Playbook Execution Best Practices

```yaml
---
# Well-structured playbook for exam execution
- name: Configure system
  hosts: all
  become: true
  gather_facts: yes
  vars:
    app_name: myapp
  pre_tasks:
    - name: Validate required variables
      assert:
        that:
          - app_name is defined
        fail_msg: "app_name variable is required"
  tasks:
    - name: Install packages
      dnf:
        name: "{{ item }}"
        state: present
      loop:
        - httpd
        - firewalld
      tags: packages

    - name: Configure services
      systemd:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop:
        - httpd
        - firewalld
      tags: services

  post_tasks:
    - name: Verify httpd is responding
      uri:
        url: http://localhost:80
        status_code: 200
      ignore_errors: true
      tags: verify
```

## Explanation of Each Configuration

### `ansible.cfg` Explanation

The `ansible.cfg` file is the central configuration for Ansible. The `[defaults]` section sets global behavior. The `inventory` parameter tells Ansible where to find the host inventory. The `remote_user` parameter specifies which user to use for SSH connections. Setting `host_key_checking = false` disables SSH host key verification, which is necessary in exam environments where hosts may not have stable host keys. The `become = true` setting enables privilege escalation by default for all tasks.

The `[privilege_escalation]` section configures how privilege escalation works. `become_method = sudo` uses sudo for escalation. `become_user = root` escalates to the root user. `become_ask_pass = false` prevents password prompts, which is essential for non-interactive playbook execution.

The `[ssh_connection]` section optimizes SSH connectivity. `pipelining = true` reduces the number of SSH round trips by sending multiple commands in a single connection. The `ssh_args` parameter configures SSH connection multiplexing, which reuses existing connections for better performance.

### Inventory File Explanation

Inventory files define the infrastructure Ansible manages. INI format uses section headers in square brackets for groups. Hosts listed under a group header belong to that group. The `:children` suffix creates nested groups that inherit all hosts from child groups. The `:vars` suffix defines variables that apply to all hosts in a group.

Host variables can be defined inline after the hostname, separated by spaces. Group variables apply to all hosts in the group and are overridden by host-specific variables. Nested groups provide a way to organize hosts hierarchically without duplicating host entries.

### SSH Key Distribution Explanation

SSH key-based authentication eliminates password prompts during playbook execution. The `ssh-keygen` command generates a public/private key pair. The private key stays on the control node; the public key is distributed to managed nodes. The `authorized_key` module deploys the public key to the `~/.ssh/authorized_keys` file on managed nodes.

The `lookup('file', path)` plugin reads file contents from the control node, enabling you to embed the public key content in a playbook without storing it as a variable.

### Privilege Escalation Explanation

Privilege escalation allows non-root users to execute commands as root. The `NOPASSWD` tag in sudoers disables password prompts for sudo commands. The `validate: "visudo -cf %s"` parameter in the `copy` module runs a syntax check before writing the sudoers file, preventing syntax errors that could lock out sudo access entirely.

### File Deployment Explanation

The `copy` module transfers files from the control node to managed nodes. The `src` parameter specifies the source file path relative to the playbook directory. The `content` parameter writes inline text directly to a file. The `template` module works like `copy` but processes Jinja2 templates, enabling dynamic content based on variables and facts.

The `backup: yes` parameter creates a backup of the existing file before overwriting it. The `validate` parameter runs a command to check file syntax before deployment, preventing broken configurations from being deployed.

## Verification Procedures

### Verify ansible.cfg Configuration

```bash
# Check effective configuration
ansible-config dump --only-changed

# Verify inventory path
ansible-config dump | grep "^inventory"

# Verify remote user
ansible-config dump | grep "^remote_user"

# Verify become settings
ansible-config dump | grep "^become"
```

### Verify Inventory

```bash
# List all hosts
ansible-inventory --list

# Show group structure
ansible-inventory --graph

# Verify host variables
ansible-inventory --host servera.lab.example.com

# Test group targeting
ansible webservers --list-hosts
ansible dbservers --list-hosts
ansible production --list-hosts
```

### Verify SSH Connectivity

```bash
# Test basic connectivity
ansible all -m ping

# Test with verbose output
ansible all -m ping -vvv

# Verify SSH user
ansible all -m command -a "whoami"

# Verify privilege escalation
ansible all -m command -a "whoami" -b

# Verify sudoers configuration
ansible all -m command -a "sudo -l"
```

### Verify ansible-navigator

```bash
# Check ansible-navigator version
ansible-navigator --version

# Verify configuration file
ansible-navigator configuration

# Test playbook execution
ansible-navigator run playbook.yml --mode stdout
```

### Verify File Deployment

```bash
# Check deployed files
ansible all -m command -a "ls -la /etc/myapp/"

# Verify file content
ansible all -m command -a "cat /etc/myapp/myapp.conf"

# Verify file permissions
ansible all -m stat -a "path=/etc/myapp/myapp.conf"
```

## Troubleshooting

### ansible.cfg Issues

**Problem: Ansible cannot find inventory file**

```bash
# Check inventory path
ansible-config dump | grep inventory

# Verify inventory file exists
ls -la /home/ansible/inventory

# Solution: Correct the inventory path in ansible.cfg
# Or specify inventory on command line
ansible-playbook playbook.yml -i /correct/path/inventory
```

**Problem: Configuration not taking effect**

```bash
# Check which ansible.cfg is being used
ansible-config dump | grep "^config file"

# Solution: Ensure ansible.cfg is in the correct location
# Priority: ANSIBLE_CONFIG > ./ansible.cfg > ~/.ansible.cfg > /etc/ansible/ansible.cfg
```

### Inventory Issues

**Problem: Host not found in inventory**

```bash
# Check inventory parsing
ansible-inventory --list

# Verify hostname spelling
cat /home/ansible/inventory

# Solution: Check for typos in hostname
# Ensure DNS resolution works
ansible all -m ping -vvv
```

**Problem: Group not recognized**

```bash
# Check group structure
ansible-inventory --graph

# Solution: Verify group syntax in inventory file
# Ensure [groupname] syntax is correct
# Check for missing or extra brackets
```

### SSH Connectivity Issues

**Problem: SSH connection refused**

```bash
# Test SSH manually
ssh ansible@servera.lab.example.com

# Check SSH service on managed node
ansible servera -m command -a "systemctl status sshd"

# Solution: Ensure sshd is running on managed node
# Check firewall rules on managed node
```

**Problem: SSH authentication failed**

```bash
# Test key authentication
ssh -i /home/ansible/.ssh/id_ed25519 ansible@servera.lab.example.com

# Check authorized keys
ssh ansible@servera.lab.example.com "cat ~/.ssh/authorized_keys"

# Solution: Verify public key is deployed correctly
# Check key file permissions
ssh ansible@servera.lab.example.com "ls -la ~/.ssh/"
```

**Problem: SSH host key verification failure**

```bash
# Solution: Disable host key checking in ansible.cfg
# [defaults]
# host_key_checking = false

# Or add to SSH args
# [ssh_connection]
# ssh_args = -o StrictHostKeyChecking=no
```

### Privilege Escalation Issues

**Problem: sudo permission denied**

```bash
# Check sudoers configuration
ansible all -m command -a "sudo -l" -b

# Verify sudoers file exists
ansible all -m command -a "cat /etc/sudoers.d/ansible"

# Solution: Ensure NOPASSWD sudoers entry exists
# ansible ALL=(ALL) NOPASSWD: ALL
```

**Problem: become fails with password prompt**

```bash
# Check ansible.cfg become settings
ansible-config dump | grep become

# Solution: Set become_ask_pass = false in ansible.cfg
# Ensure sudoers has NOPASSWD entry
```

### Playbook Execution Issues

**Problem: Playbook fails on first task**

```bash
# Run with verbose output
ansible-playbook playbook.yml -vvv

# Check syntax first
ansible-playbook playbook.yml --syntax-check

# Run in check mode
ansible-playbook playbook.yml --check

# Solution: Review error message in verbose output
# Fix the specific failing task
```

**Problem: Task reports changed when it should not**

```bash
# Run with diff output
ansible-playbook playbook.yml --check --diff

# Solution: Add changed_when: false for read-only tasks
# Ensure module parameters are correct for idempotency
```

**Problem: Module not found**

```bash
# Check available modules
ansible-doc -l | grep -i module_name

# Check installed collections
ansible-galaxy collection list

# Solution: Install required collection
ansible-galaxy collection install collection_name
```

### ansible-navigator Issues

**Problem: ansible-navigator fails to start**

```bash
# Check installation
ansible-navigator --version

# Check configuration file
cat /home/ansible/ansible-navigator.yml

# Solution: Verify YAML syntax in configuration file
# Check execution environment settings
```

**Problem: Playbook does not execute in ansible-navigator**

```bash
# Run with stdout mode for visible output
ansible-navigator run playbook.yml --mode stdout

# Check inventory specification
ansible-navigator run playbook.yml --inventory /home/ansible/inventory

# Solution: Verify inventory path is correct
# Check that hosts are reachable
```

## Real-World Automation Examples

### Complete Control Node Setup

```yaml
---
- name: Set up Ansible control node
  hosts: localhost
  connection: local
  become: true
  vars:
    ansible_home: /home/ansible
  tasks:
    - name: Create Ansible working directory
      file:
        path: "{{ ansible_home }}/{{ item }}"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"
      loop:
        - .
        - inventory
        - playbooks
        - roles
        - files
        - templates
        - vars

    - name: Create ansible.cfg
      copy:
        content: |
          [defaults]
          inventory = {{ ansible_home }}/inventory
          remote_user = ansible
          host_key_checking = false
          log_path = {{ ansible_home }}/ansible.log
          forks = 10
          timeout = 30
          collections_paths = {{ ansible_home }}/.ansible/collections:/usr/share/ansible/collections

          [privilege_escalation]
          become = true
          become_method = sudo
          become_user = root
          become_ask_pass = false

          [ssh_connection]
          pipelining = true
          ssh_args = -o ControlMaster=auto -o ControlPersist=60s
        dest: "{{ ansible_home }}/ansible.cfg"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create ansible-navigator.yml
      copy:
        content: |
          ansible-navigator:
            execution-environment:
              enabled: false
            mode: stdout
            ansible:
              diff: true
              inventory:
                - {{ ansible_home }}/inventory
            logging:
              level: warning
        dest: "{{ ansible_home }}/ansible-navigator.yml"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create static inventory
      copy:
        content: |
          [webservers]
          servera.lab.example.com
          serverb.lab.example.com

          [dbservers]
          serverc.lab.example.com

          [production:children]
          webservers
          dbservers

          [webservers:vars]
          http_port=8080
          app_name=myapp

          [dbservers:vars]
          db_port=5432
        dest: "{{ ansible_home }}/inventory/production"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Generate SSH key if not exists
      openssh_keypair:
        path: "{{ ansible_home }}/.ssh/id_ed25519"
        type: ed25519
        state: present
      no_log: true
```

### Managed Node Bootstrap

```yaml
---
- name: Bootstrap managed nodes
  hosts: all
  become: true
  vars:
    control_key: "{{ lookup('file', '/home/ansible/.ssh/id_ed25519.pub') }}"
  tasks:
    - name: Create ansible user
      user:
        name: ansible
        state: present
        createhome: yes
        shell: /bin/bash
        groups: wheel

    - name: Configure SSH directory
      file:
        path: /home/ansible/.ssh
        state: directory
        owner: ansible
        group: ansible
        mode: "0700"

    - name: Deploy SSH key
      authorized_key:
        user: ansible
        state: present
        key: "{{ control_key }}"

    - name: Configure sudoers
      copy:
        content: "ansible ALL=(ALL) NOPASSWD: ALL"
        dest: /etc/sudoers.d/ansible
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"

    - name: Install required Python packages
      dnf:
        name:
          - python3
          - python3-libselinux
          - python3-firewall
          - python3-cryptography
        state: present

    - name: Ensure SSH is running
      systemd:
        name: sshd
        state: started
        enabled: yes

    - name: Ensure firewall allows SSH
      firewalld:
        port: "22/tcp"
        permanent: yes
        immediate: yes
        state: enabled

    - name: Verify connectivity
      ping:
```

## RHCE Exam Notes

### Critical Exam Points

1. **ansible.cfg is your first step**: Always verify or create `ansible.cfg` before running playbooks. The exam environment may not have a pre-configured file. Know the essential settings: `inventory`, `remote_user`, `host_key_checking`, `become`, and SSH connection settings.

2. **Inventory files must be accurate**: Hostnames must match exactly. Groups must be properly defined. Test your inventory with `ansible-inventory --list` before running playbooks.

3. **SSH keys must work**: If you cannot ping managed nodes, no playbook will work. Verify SSH connectivity early and fix authentication issues before proceeding.

4. **Privilege escalation is required**: Most exam tasks require root access. Ensure `become: true` is set and sudoers is configured correctly on managed nodes.

5. **ansible-navigator is the primary interface**: The exam expects you to use ansible-navigator for playbook execution and documentation lookup. Know how to run playbooks, search for modules, and view documentation within ansible-navigator.

6. **Use `--check` and `--diff`**: Preview changes before applying them. This helps catch errors before they cause problems.

7. **Configuration file locations matter**: Ansible searches for `ansible.cfg` in a specific order. Place it in your working directory to ensure it takes effect.

8. **Validate sudoers syntax**: Always use `validate: "visudo -cf %s"` when writing sudoers files to prevent locking yourself out.

### Exam Workflow

1. Create or verify `ansible.cfg`
2. Create or verify inventory file
3. Test SSH connectivity with `ansible all -m ping`
4. Verify privilege escalation with `ansible all -m ping -b`
5. Create playbooks for each task
6. Validate syntax with `--syntax-check`
7. Preview changes with `--check --diff`
8. Execute playbooks with ansible-navigator
9. Verify results on managed nodes
10. Commit changes to Git

### Time Management

- Spend no more than 5 minutes on initial configuration
- Test connectivity before writing playbooks
- Use `--syntax-check` to catch errors early
- Use ansible-navigator documentation browser when unsure about module parameters
- Move on from stuck tasks and return later

## Common Mistakes

### Mistake 1: Wrong Inventory Path in ansible.cfg

```ini
# WRONG - path does not exist
[defaults]
inventory = /etc/ansible/inventory

# CORRECT - path matches actual inventory location
[defaults]
inventory = /home/ansible/inventory
```

### Mistake 2: Missing `host_key_checking = false`

```ini
# WRONG - SSH host key checking enabled
[defaults]
# host_key_checking not set

# CORRECT - disable for exam environment
[defaults]
host_key_checking = false
```

### Mistake 3: Incorrect Sudoers Syntax

```yaml
# WRONG - missing validate parameter
- name: Configure sudoers
  copy:
    content: "ansible ALL=(ALL) NOPASSWD: ALL"
    dest: /etc/sudoers.d/ansible
    mode: "0440"

# CORRECT - validate syntax before writing
- name: Configure sudoers
  copy:
    content: "ansible ALL=(ALL) NOPASSWD: ALL"
    dest: /etc/sudoers.d/ansible
    mode: "0440"
    validate: "visudo -cf %s"
```

### Mistake 4: Wrong SSH Key Permissions

```bash
# WRONG - overly permissive key file
chmod 644 ~/.ssh/id_ed25519

# CORRECT - private key must be restricted
chmod 600 ~/.ssh/id_ed25519
chmod 700 ~/.ssh
```

### Mistake 5: Forgetting to Restart SSH After Key Deployment

```yaml
# WRONG - SSH may not pick up key changes
- name: Deploy SSH key
  authorized_key:
    user: ansible
    state: present
    key: "{{ ssh_key }}"

# CORRECT - ensure SSH is running and key is accessible
- name: Deploy SSH key
  authorized_key:
    user: ansible
    state: present
    key: "{{ ssh_key }}"
    manage_dir: yes

- name: Ensure SSH is running
  systemd:
    name: sshd
    state: started
    enabled: yes
```

### Mistake 6: Not Testing Inventory Before Playbook

```bash
# WRONG - jumping straight to playbook execution
ansible-playbook playbook.yml

# CORRECT - verify inventory first
ansible-inventory --list
ansible all -m ping
ansible-playbook playbook.yml
```

### Mistake 7: Mixing INI and YAML Inventory in Same File

```ini
# WRONG - mixing formats
[webservers]
servera
  hosts:
    serverb:    # YAML syntax in INI file

# CORRECT - use consistent format
[webservers]
servera
serverb
```

### Mistake 8: Not Setting `become_ask_pass = false`

```ini
# WRONG - will prompt for password
[privilege_escalation]
become = true
# become_ask_pass not set (defaults to true)

# CORRECT - disable password prompt
[privilege_escalation]
become = true
become_ask_pass = false
```

## Chapter Summary

This chapter covered the complete Ansible configuration workflow:

- **`ansible.cfg`** controls global Ansible behavior including inventory location, remote user, privilege escalation, SSH settings, and logging. Create it in your working directory for maximum priority.
- **`ansible-navigator.yml`** configures the TUI interface used in the exam. Key settings include execution environment, display mode, diff output, and logging level.
- **Static inventories** define managed hosts using INI or YAML format. Groups organize hosts logically, and nested groups provide hierarchy. Host and group variables customize per-target behavior.
- **Managed node configuration** requires SSH access, Python 3, and sudo privileges. The `authorized_key` module deploys SSH keys, and sudoers entries enable passwordless privilege escalation.
- **SSH key distribution** establishes passwordless authentication between the control node and managed nodes. The `lookup` plugin reads key files from the control node.
- **File deployment** uses `copy` for static files and `template` for dynamic content. The `validate` parameter ensures configuration file correctness before deployment.
- **Playbook execution** supports both `ansible-playbook` CLI and `ansible-navigator` TUI. Use `--check`, `--diff`, `--syntax-check`, and `-vvv` for debugging and validation.
- **ansible-navigator documentation** provides module lookup capabilities essential for the exam. Know how to search for modules and review their parameters.

## Quick Reference

| Task | Command | Purpose |
|---|---|---|
| View config | `ansible-config dump` | Show effective settings |
| View changed config | `ansible-config dump --only-changed` | Show non-default settings |
| List inventory | `ansible-inventory --list` | Show all hosts and groups |
| Graph inventory | `ansible-inventory --graph` | Show group hierarchy |
| Host variables | `ansible-inventory --host hostname` | Show host-specific vars |
| Ping hosts | `ansible all -m ping` | Test connectivity |
| Ping with become | `ansible all -m ping -b` | Test privilege escalation |
| Generate SSH key | `ssh-keygen -t ed25519` | Create key pair |
| Distribute key | `ssh-copy-id user@host` | Deploy public key |
| Run playbook | `ansible-playbook playbook.yml` | Execute playbook |
| Dry run | `ansible-playbook playbook.yml --check --diff` | Preview changes |
| Syntax check | `ansible-playbook playbook.yml --syntax-check` | Validate syntax |
| Verbose output | `ansible-playbook playbook.yml -vvv` | Debug output |
| Run in navigator | `ansible-navigator run playbook.yml` | Execute via TUI |
| Navigator docs | `ansible-navigator` | Open documentation |
| Module docs | `ansible-doc module_name` | View module help |
| Module snippet | `ansible-doc -s module_name` | Show parameters |
| Extra variables | `-e "key=value"` | Pass runtime variables |
| Limit hosts | `--limit hostname` | Target specific hosts |
| Run tags | `--tags tag_name` | Run tagged tasks only |
| Skip tags | `--skip-tags tag_name` | Skip tagged tasks |

## Review Questions

1. In what order does Ansible search for the `ansible.cfg` file, and which location has the highest priority?

2. What is the purpose of `host_key_checking = false` in `ansible.cfg`, and why is it important for the exam environment?

3. What three parameters are essential in the `[privilege_escalation]` section of `ansible.cfg`, and what should their values be for the exam?

4. How do you create a nested group in an INI-format inventory file, and what is its purpose?

5. What is the difference between the `copy` module and the `template` module when deploying files to managed nodes?

6. How do you use the `lookup` plugin to read a file from the control node and use its content in a playbook task?

7. What is the purpose of the `validate` parameter in the `copy` module, and when should you use it?

8. How do you run a playbook in check mode with diff output, and what information does this provide?

9. What command do you use to verify that your inventory file is correctly parsed, and what output should you expect?

10. How do you configure passwordless sudo for the Ansible user on a managed node using Ansible, and what safety measure should you include?

## Answers

1. Ansible searches for `ansible.cfg` in this order: (1) the path specified by the `ANSIBLE_CONFIG` environment variable (highest priority), (2) `ansible.cfg` in the current working directory, (3) `~/.ansible.cfg` in the user's home directory, (4) `/etc/ansible/ansible.cfg` system-wide (lowest priority). The first file found is used, and subsequent files are ignored.

2. `host_key_checking = false` disables SSH host key verification when connecting to managed nodes. In the exam environment, managed nodes may have changed host keys or may not have stable host keys registered. Without this setting, SSH will prompt to accept new host keys, which blocks non-interactive playbook execution. Setting it to `false` allows Ansible to connect without host key verification prompts.

3. The three essential parameters are: `become_method = sudo` (specifies sudo as the escalation method), `become_user = root` (escalates to the root user for system administration tasks), and `become_ask_pass = false` (prevents password prompts during non-interactive playbook execution). These three settings together enable seamless privilege escalation without user interaction.

4. Create a nested group using the `[parent_group:children]` syntax, then list child group names underneath. For example:
```ini
[production:children]
webservers
dbservers
```
The purpose is to organize hosts hierarchically. Hosts in child groups are automatically included in the parent group, allowing you to target multiple groups with a single group name without duplicating host entries.

5. The `copy` module transfers static files from the control node to managed nodes or writes inline content using the `content` parameter. The `template` module works similarly but processes Jinja2 templates, replacing variables, conditionals, and loops with actual values before writing the file. Use `copy` for static content and `template` for dynamic content that varies per host or depends on variables and facts.

6. Use the `lookup('file', path)` plugin to read file contents from the control node. For example: `ssh_key: "{{ lookup('file', '/home/ansible/.ssh/id_ed25519.pub') }}"`. The `lookup` plugin runs on the control node and returns the file contents as a string that can be used in playbook tasks. This is useful for embedding public keys, passwords, or other control-node-only data into playbook tasks.

7. The `validate` parameter runs a command on the managed node to check the syntax or validity of a file before it is deployed. The `%s` placeholder is replaced with the path to a temporary file containing the new content. Use it when deploying configuration files where syntax errors could cause service failures, such as sudoers files (`validate: "visudo -cf %s"`), Apache configs (`validate: "httpd -t -f %s"`), or SSH configs (`validate: "sshd -t -f %s"`).

8. Run with `ansible-playbook playbook.yml --check --diff`. Check mode (`--check`) simulates playbook execution without making actual changes, reporting what would change. The `--diff` flag shows the before/after content of files that would be modified. This combination lets you preview all changes before applying them, helping catch errors and verify expected behavior.

9. Use `ansible-inventory --list` to verify inventory parsing. The output is a JSON structure showing all groups, their hosts, and associated variables. You should see your defined groups, hosts listed under each group, and any host or group variables. Use `ansible-inventory --graph` to visualize the group hierarchy. If the output matches your expected structure, the inventory is correctly configured.

10. Use the `copy` module to write a sudoers file in `/etc/sudoers.d/` with content `ansible ALL=(ALL) NOPASSWD: ALL`. Set `mode: "0440"`, `owner: root`, and `group: root`. The critical safety measure is `validate: "visudo -cf %s"`, which checks sudoers syntax before writing the file. Without this, a syntax error in the sudoers file could lock out all sudo access, requiring single-user mode recovery to fix.


---

# Chapter 03 — Reviewed Edition Closing Checkpoint

Before you mark this chapter complete, verify that you can:

- explain the chapter topic in plain English;
- identify which EX294 objective group it supports;
- write a small playbook from scratch without copying;
- run syntax checks and dry-run checks;
- run the playbook twice and explain the result difference between `ok` and `changed`;
- verify the final managed-node state using Linux commands;
- troubleshoot at least one realistic failure related to this chapter;
- identify which examples depend on collections or environment-specific tools and verify them with `ansible-doc` or `ansible-navigator doc`.

**Personal rule:** do not move to the next chapter until this topic feels like a normal Linux administration workflow translated into Ansible state.
