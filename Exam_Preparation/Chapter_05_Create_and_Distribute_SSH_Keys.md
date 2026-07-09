# Chapter 5: Create and Distribute SSH Keys to Managed Nodes

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand SSH key-based authentication and its importance
- Generate SSH key pairs on the control node
- Distribute SSH public keys to managed nodes
- Configure SSH server settings on managed nodes
- Create and manage `authorized_keys` files
- Configure passwordless sudo for Ansible user
- Automate SSH key distribution with Ansible
- Troubleshoot SSH authentication issues
- Verify SSH connectivity to managed nodes

## Concept Overview

SSH (Secure Shell) is the primary communication protocol between Ansible control nodes and managed nodes. Ansible uses SSH to execute modules remotely. Passwordless SSH key-based authentication is required for non-interactive playbook execution.

### Why SSH Key Distribution Matters

Without SSH key-based authentication:

- Playbooks cannot run without password prompts
- Ansible cannot execute modules on managed nodes
- Automated configurations fail
- Remote management is impossible

SSH key distribution establishes trust between the control node and managed nodes, enabling passwordless, secure remote execution.

### How SSH Key Authentication Works

```
Control Node                          Managed Node
─────────────                         ────────────
1. Generate key pair (ed25519/rsa)
   │
   ├── Private key: stays on control node
   │
   └── Public key: distributed to managed node
         │
         ├── Stored in ~/.ssh/authorized_keys
         │
         └── Used to authenticate SSH connections

2. SSH connection established
   │
   ├── Control node presents private key
   │
   ├── Managed node verifies with public key
   │
   └── Authentication successful, session opened
```

## Ansible Fundamentals Required

### SSH Protocol Basics

SSH uses asymmetric cryptography with key pairs:

- **Private key**: Secret key kept on the control node. Never share this.
- **Public key**: Key distributed to managed nodes. Can be shared safely.

### SSH Directory Structure

```
~/.ssh/
├── id_ed25519              # Private key (control node)
├── id_ed25519.pub          # Public key (control node)
└── authorized_keys         # Public keys (managed node)
```

### SSH Key Types

| Key Type | Bits | Recommendation |
|---|---|---|
| ed25519 | 256 | Recommended (modern, fast, secure) |
| rsa | 4096 | Legacy alternative |
| ecdsa | 256 | Rarely used |

## Commands and Tools

```bash
# Generate SSH key pair (Ed25519 recommended)
ssh-keygen -t ed25519 -C "ansible-control" -N "" -f ~/.ssh/id_ed25519

# Generate RSA key pair
ssh-keygen -t rsa -b 4096 -C "ansible-control" -N "" -f ~/.ssh/id_rsa

# List SSH keys
ls -la ~/.ssh/

# Copy public key to managed node
ssh-copy-id -i ~/.ssh/id_ed25519.pub ansible@servera.lab.example.com

# Test SSH connectivity
ssh ansible@servera.lab.example.com "whoami"
ssh ansible@servera.lab.example.com "hostname"

# Display public key
cat ~/.ssh/id_ed25519.pub

# Check authorized_keys on managed node
ssh ansible@servera.lab.example.com "cat ~/.ssh/authorized_keys"

# Test Ansible connectivity
ansible all -m ping

# Test verbose SSH output
ansible all -m ping -vvv

# Check SSH server status on managed node
ansible all -m command -a "systemctl status sshd"

# Check SSH configuration
ansible all -m command -a "cat /etc/ssh/sshd_config"

# Check sudo access
ansible all -m command -a "sudo -l"
```

## Playbook Examples

### Example 1: Generate SSH Keys on Control Node

```yaml
---
- name: Generate SSH keys on control node
  hosts: localhost
  connection: local
  become: yes
  tasks:
    - name: Create .ssh directory
      file:
        path: /home/ansible/.ssh
        state: directory
        owner: ansible
        group: ansible
        mode: "0700"

    - name: Generate Ed25519 SSH key pair
      openssh_keypair:
        path: /home/ansible/.ssh/id_ed25519
        type: ed25519
        state: present
      no_log: true

    - name: Generate RSA SSH key pair (backup)
      openssh_keypair:
        path: /home/ansible/.ssh/id_rsa
        type: rsa
        size: 4096
        state: present
      no_log: true

    - name: Display public key content
      debug:
        var: openssh_keypair_result
      when: openssh_keypair_result is defined

    - name: Set correct permissions on private key
      file:
        path: /home/ansible/.ssh/id_ed25519
        mode: "0600"
        owner: ansible
        group: ansible

    - name: Set correct permissions on public key
      file:
        path: /home/ansible/.ssh/id_ed25519.pub
        mode: "0644"
        owner: ansible
        group: ansible

    - name: Set correct permissions on .ssh directory
      file:
        path: /home/ansible/.ssh
        mode: "0700"
        owner: ansible
        group: ansible
```

### Example 2: Distribute SSH Keys to Managed Nodes

```yaml
---
- name: Distribute SSH keys to managed nodes
  hosts: all
  become: yes
  vars:
    ansible_user_name: ansible
    ssh_key_path: /home/ansible/.ssh/id_ed25519.pub
    ssh_key_file: /home/ansible/.ssh/id_ed25519
  tasks:
    - name: Create Ansible user if not exists
      user:
        name: "{{ ansible_user_name }}"
        state: present
        createhome: yes
        shell: /bin/bash
        groups: wheel

    - name: Create .ssh directory on managed node
      file:
        path: "/home/{{ ansible_user_name }}/.ssh"
        state: directory
        owner: "{{ ansible_user_name }}"
        group: "{{ ansible_user_name }}"
        mode: "0700"

    - name: Deploy SSH public key
      authorized_key:
        user: "{{ ansible_user_name }}"
        state: present
        key: "{{ lookup('file', ssh_key_path) }}"
        manage_dir: yes

    - name: Set correct permissions on authorized_keys
      file:
        path: "/home/{{ ansible_user_name }}/.ssh/authorized_keys"
        owner: "{{ ansible_user_name }}"
        group: "{{ ansible_user_name }}"
        mode: "0600"

    - name: Verify SSH key deployment
      command: cat /home/{{ ansible_user_name }}/.ssh/authorized_keys
      register: key_check
      changed_when: false

    - name: Test SSH connectivity
      command: ssh -i "{{ ssh_key_file }}" -o BatchMode=yes "{{ ansible_user_name }}@{{ inventory_hostname }}" "echo connection successful"
      register: ssh_test
      changed_when: false
      failed_when: false

    - name: Display SSH test result
      debug:
        msg: "SSH connection {{ 'successful' if ssh_test.rc == 0 else 'failed' }}"
```

### Example 3: Configure SSH Server on Managed Nodes

```yaml
---
- name: Configure SSH server on managed nodes
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
    - name: Ensure SSH server is installed
      dnf:
        name: openssh-server
        state: present

    - name: Ensure SSH daemon is running
      systemd:
        name: sshd
        state: started
        enabled: yes

    - name: Configure SSH to allow passwordless authentication
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^{{ item.regexp }}"
        line: "{{ item.line }}"
        state: present
        backup: yes
      loop:
        - { regexp: "^#?PubkeyAuthentication", line: "PubkeyAuthentication yes" }
        - { regexp: "^#?PasswordAuthentication", line: "PasswordAuthentication no" }
        - { regexp: "^#?PermitRootLogin", line: "PermitRootLogin prohibit-password" }
        - { regexp: "^#?X11Forwarding", line: "X11Forwarding no" }
      notify: restart sshd

    - name: Configure SSH to allow specific users
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^#?AllowUsers"
        line: "AllowUsers ansible admin root"
        state: present
        backup: yes
      notify: restart sshd

    - name: Set SSH login grace time
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^#?ClientAliveInterval"
        line: "ClientAliveInterval 300"
        state: present
        backup: yes
      notify: restart sshd

    - name: Validate SSH configuration
      command: sshd -t
      register: sshd_test
      changed_when: false

    - name: Restart SSH if configuration changed
      systemd:
        name: sshd
        state: restarted
      when: sshd_test.rc != 0
      notify: restart sshd

  handlers:
    - name: restart sshd
      systemd:
        name: sshd
        state: restarted
```

### Example 4: Configure Sudoers for Ansible User

```yaml
---
- name: Configure sudoers for Ansible user
  hosts: all
  become: yes
  tasks:
    - name: Ensure Ansible user exists
      user:
        name: ansible
        state: present
        createhome: yes
        shell: /bin/bash
        groups: wheel

    - name: Add Ansible user to wheel group
      user:
        name: ansible
        groups: wheel
        append: yes

    - name: Configure passwordless sudo for Ansible user
      copy:
        content: "ansible ALL=(ALL) NOPASSWD: ALL"
        dest: /etc/sudoers.d/ansible
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"

    - name: Configure passwordless sudo for wheel group
      lineinfile:
        path: /etc/sudoers
        regexp: "^%wheel"
        line: "%wheel ALL=(ALL) NOPASSWD: ALL"
        state: present
        validate: "visudo -cf %s"
        backup: yes

    - name: Verify sudoers syntax
      command: visudo -cf /etc/sudoers.d/ansible
      register: sudoers_test
      changed_when: false

    - name: Display sudoers configuration
      command: cat /etc/sudoers.d/ansible
      register: sudoers_config
      changed_when: false

    - name: Test sudo access
      command: sudo -l
      register: sudo_test
      changed_when: false
      failed_when: false

    - name: Display sudo test result
      debug:
        msg: "Sudo access {{ 'configured' if sudo_test.rc == 0 else 'not configured' }}"
```

### Example 5: Complete SSH Key Distribution and Verification

```yaml
---
- name: Complete SSH key distribution and verification
  hosts: all
  become: yes
  gather_facts: yes
  vars:
    ansible_user_name: ansible
    ssh_key_path: /home/ansible/.ssh/id_ed25519.pub
    ssh_key_file: /home/ansible/.ssh/id_ed25519
  tasks:
    # Create Ansible user
    - name: Create Ansible user
      user:
        name: "{{ ansible_user_name }}"
        state: present
        createhome: yes
        shell: /bin/bash
        groups: wheel
        comment: "Ansible automation user"

    # Configure SSH directory
    - name: Create .ssh directory
      file:
        path: "/home/{{ ansible_user_name }}/.ssh"
        state: directory
        owner: "{{ ansible_user_name }}"
        group: "{{ ansible_user_name }}"
        mode: "0700"

    # Deploy SSH key
    - name: Deploy SSH public key
      authorized_key:
        user: "{{ ansible_user_name }}"
        state: present
        key: "{{ lookup('file', ssh_key_path) }}"
        manage_dir: yes
        exclusive: no

    # Set permissions
    - name: Set authorized_keys permissions
      file:
        path: "/home/{{ ansible_user_name }}/.ssh/authorized_keys"
        owner: "{{ ansible_user_name }}"
        group: "{{ ansible_user_name }}"
        mode: "0600"

    # Configure sudoers
    - name: Configure passwordless sudo
      copy:
        content: "{{ ansible_user_name }} ALL=(ALL) NOPASSWD: ALL"
        dest: /etc/sudoers.d/{{ ansible_user_name }}
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"

    # Ensure SSH is running
    - name: Ensure SSH server is running
      systemd:
        name: sshd
        state: started
        enabled: yes

    # Verify SSH key deployment
    - name: Verify SSH key exists
      stat:
        path: "/home/{{ ansible_user_name }}/.ssh/authorized_keys"
      register: key_stat

    - name: Verify SSH key content
      command: cat /home/{{ ansible_user_name }}/.ssh/authorized_keys
      register: key_content
      changed_when: false

    - name: Verify sudoers file
      stat:
        path: /etc/sudoers.d/{{ ansible_user_name }}
      register: sudoers_stat

    - name: Verify sudoers content
      command: cat /etc/sudoers.d/{{ ansible_user_name }}
      register: sudoers_content
      changed_when: false

    # Test connectivity
    - name: Test SSH connectivity
      command: ssh -i "{{ ssh_key_file }}" -o BatchMode=yes -o StrictHostKeyChecking=no "{{ ansible_user_name }}@{{ inventory_hostname }}" "echo SSH connection successful"
      register: ssh_test
      changed_when: false

    - name: Test Ansible ping
      ansible.builtin.ping:

    - name: Test privilege escalation
      command: whoami
      register: whoami_test
      changed_when: false
      failed_when: false

    # Display verification results
    - name: Display SSH key content
      debug:
        msg: "SSH key deployed: {{ key_content.stdout }}"

    - name: Display sudoers configuration
      debug:
        msg: "Sudoers configured: {{ sudoers_content.stdout }}"

    - name: Display connectivity test result
      debug:
        msg: "SSH connectivity {{ 'successful' if ssh_test.rc == 0 else 'failed' }}"

    - name: Display privilege escalation test
      debug:
        msg: "Privilege escalation {{ 'working' if whoami_test.rc == 0 else 'not working' }} as {{ whoami_test.stdout }}"
```

## Explanation of Each Playbook

### Playbook 1: Generate SSH Keys

This playbook creates SSH key pairs on the control node using the `openssh_keypair` module. The Ed25519 algorithm is recommended for its security and performance. The `no_log: true` parameter prevents the private key content from being logged. File permissions are set to `0600` for the private key (read/write for owner only) and `0644` for the public key.

### Playbook 2: Distribute SSH Keys

This playbook deploys SSH public keys to managed nodes using the `authorized_key` module. The `lookup('file', path)` plugin reads the public key content from the control node. The `manage_dir: yes` parameter automatically creates the `.ssh` directory if it doesn't exist. File permissions are set to `0600` on `authorized_keys` for security.

### Playbook 3: Configure SSH Server

This playbook configures the SSH daemon on managed nodes. It ensures the `openssh-server` package is installed and the `sshd` service is running. The `lineinfile` module modifies `/etc/ssh/sshd_config` to enable key-based authentication and disable password authentication. The `notify` directive triggers handlers to restart SSH after configuration changes.

### Playbook 4: Configure Sudoers

This playbook sets up passwordless sudo access for the Ansible user. It creates the Ansible user if it doesn't exist, adds the user to the wheel group, and writes a sudoers file with the `NOPASSWD` tag. The `validate: "visudo -cf %s"` parameter is critical—it checks sudoers syntax before writing to prevent locking out sudo access.

### Playbook 5: Complete Distribution and Verification

This comprehensive playbook combines all previous steps and verifies each configuration. It creates the user, deploys the key, configures sudoers, ensures SSH is running, and tests connectivity. The `stat` module checks for file existence, and the `command` module reads file contents for verification.

## Variables and Templates

### Using Variables for SSH Configuration

```yaml
---
- name: SSH configuration with variables
  hosts: all
  become: yes
  vars:
    ansible_user_name: ansible
    ssh_key_path: /home/ansible/.ssh/id_ed25519.pub
    ssh_key_file: /home/ansible/.ssh/id_ed25519
  tasks:
    - name: Deploy SSH key
      authorized_key:
        user: "{{ ansible_user_name }}"
        state: present
        key: "{{ lookup('file', ssh_key_path) }}"
```

### Using Jinja2 Templates for SSH Configuration

```yaml
# Template: sshd_config.d/myapp.conf.j2
# SSH configuration for {{ inventory_hostname }}
{% if custom_ssh_port | default(22) %}
Port {{ custom_ssh_port }}
{% endif %}
{% if custom_ssh_port | default(22) %}
Port {{ custom_ssh_port }}
{% endif %}
```

```yaml
# Playbook using template
---
- name: Deploy custom SSH configuration
  hosts: all
  become: yes
  vars:
    custom_ssh_port: 2222
  tasks:
    - name: Deploy custom SSH config
      template:
        src: templates/sshd_config.d/myapp.conf.j2
        dest: /etc/ssh/sshd_config.d/myapp.conf
        owner: root
        group: root
        mode: "0600"
        validate: "sshd -t -f %s"
      notify: restart sshd
```

### Using Facts for SSH Configuration

```yaml
---
- name: SSH configuration with facts
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
    - name: Configure based on OS family
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^{{ item.regexp }}"
        line: "{{ item.line }}"
        state: present
      when: ansible_os_family == "RedHat"
      loop:
        - { regexp: "^#?PermitRootLogin", line: "PermitRootLogin prohibit-password" }
```

## Verification Procedures

### Verify SSH Key Generation

```bash
# Check private key exists
ls -la ~/.ssh/id_ed25519

# Check public key exists
ls -la ~/.ssh/id_ed25519.pub

# View private key (first 5 lines)
head -5 ~/.ssh/id_ed25519

# View public key
cat ~/.ssh/id_ed25519.pub

# Check key permissions
stat ~/.ssh/id_ed25519
```

### Verify SSH Key Distribution

```bash
# Check authorized_keys on managed node
ssh ansible@servera.lab.example.com "cat ~/.ssh/authorized_keys"

# Check authorized_keys permissions
ssh ansible@servera.lab.example.com "ls -la ~/.ssh/authorized_keys"

# Check .ssh directory permissions
ssh ansible@servera.lab.example.com "ls -la ~/.ssh/"

# Verify key matches control node
ssh ansible@servera.lab.example.com "cat ~/.ssh/authorized_keys" | diff - ~/.ssh/id_ed25519.pub
```

### Verify SSH Connectivity

```bash
# Test basic SSH
ssh ansible@servera.lab.example.com "whoami"

# Test with verbose output
ssh -v ansible@servera.lab.example.com "whoami"

# Test with Ansible
ansible all -m ping

# Test with verbose Ansible output
ansible all -m ping -vvv

# Test privilege escalation
ansible all -m ping -b

# Test sudo access
ansible all -m command -a "sudo whoami"
```

### Verify Sudoers Configuration

```bash
# Check sudoers file exists
ls -la /etc/sudoers.d/ansible

# Check sudoers content
cat /etc/sudoers.d/ansible

# Verify sudoers syntax
visudo -cf /etc/sudoers.d/ansible

# Test sudo access
ansible all -m command -a "sudo whoami"

# Display sudo privileges
ansible all -m command -a "sudo -l"
```

### Verify SSH Service

```bash
# Check SSH service status
ansible all -m command -a "systemctl is-active sshd"

# Check SSH service enabled
ansible all -m command -a "systemctl is-enabled sshd"

# Check SSH configuration
ansible all -m command -a "sshd -t"

# View SSH logs
ansible all -m command -a "journalctl -u sshd -n 20"
```

## Troubleshooting

### SSH Connection Refused

**Problem: Connection refused error**

```bash
# Solution: Check SSH service on managed node
ansible servera -m command -a "systemctl status sshd"

# Solution: Start SSH service
ansible servera -m systemd -a "name=sshd state=started enabled=yes"
```

**Ansible playbook:**
```yaml
- name: Ensure SSH is running
  systemd:
    name: sshd
    state: started
    enabled: yes
```

### SSH Authentication Failed

**Problem: Permission denied (publickey)**

```bash
# Solution: Check authorized_keys file
ssh ansible@servera.lab.example.com "cat ~/.ssh/authorized_keys"

# Solution: Check key permissions
ssh ansible@servera.lab.example.com "ls -la ~/.ssh/authorized_keys"

# Solution: Check .ssh directory permissions
ssh ansible@servera.lab.example.com "ls -la ~/.ssh/"
```

**Required permissions:**
```bash
# .ssh directory must be 700
chmod 700 ~/.ssh

# authorized_keys must be 600
chmod 600 ~/.ssh/authorized_keys
```

**Ansible playbook:**
```yaml
- name: Set .ssh directory permissions
  file:
    path: "/home/{{ ansible_user_name }}/.ssh"
    mode: "0700"

- name: Set authorized_keys permissions
  file:
    path: "/home/{{ ansible_user_name }}/.ssh/authorized_keys"
    mode: "0600"
```

### SSH Key Not Matching

**Problem: Public key on managed node doesn't match control node**

```bash
# Solution: Check public key on control node
cat ~/.ssh/id_ed25519.pub

# Solution: Check authorized_keys on managed node
ssh ansible@servera.lab.example.com "cat ~/.ssh/authorized_keys"

# Solution: Re-deploy key
ssh-copy-id -i ~/.ssh/id_ed25519.pub ansible@servera.lab.example.com
```

**Ansible playbook:**
```yaml
- name: Deploy SSH public key
  authorized_key:
    user: ansible
    state: present
    key: "{{ lookup('file', '/home/ansible/.ssh/id_ed25519.pub') }}"
    manage_dir: yes
```

### Sudoers Syntax Error

**Problem: sudoers file has syntax errors**

```bash
# Solution: Check syntax before writing
visudo -cf /etc/sudoers.d/ansible

# Solution: Use validate parameter
copy:
  content: "ansible ALL=(ALL) NOPASSWD: ALL"
  dest: /etc/sudoers.d/ansible
  mode: "0440"
  validate: "visudo -cf %s"
```

**Common syntax errors:**
- Missing spaces around operators
- Incorrect file permissions (must be 0440)
- Invalid user/group names

### SSH Host Key Verification Failure

**Problem: SSH prompts to accept host key**

```bash
# Solution: Disable host key checking in ansible.cfg
[defaults]
host_key_checking = false

# Or use SSH arguments
[ssh_connection]
ssh_args = -o StrictHostKeyChecking=no
```

### Authorized Keys File Not Found

**Problem: authorized_keys file doesn't exist**

```bash
# Solution: Check .ssh directory exists
ssh ansible@servera.lab.example.com "ls -la ~/.ssh/"

# Solution: Create .ssh directory
ansible all -m file -a "path=/home/ansible/.ssh state=directory mode=0700"

# Solution: Deploy key with manage_dir=yes
authorized_key:
  user: ansible
  state: present
  key: "{{ ssh_key }}"
  manage_dir: yes
```

## Real-World Automation Examples

### Complete Ansible Control Node Setup

```yaml
---
- name: Set up complete Ansible environment
  hosts: localhost
  connection: local
  become: yes
  tasks:
    - name: Create Ansible working directory
      file:
        path: /home/ansible
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create .ssh directory
      file:
        path: /home/ansible/.ssh
        state: directory
        owner: ansible
        group: ansible
        mode: "0700"

    - name: Generate SSH key pair
      openssh_keypair:
        path: /home/ansible/.ssh/id_ed25519
        type: ed25519
        state: present
      no_log: true

    - name: Set SSH key permissions
      file:
        path: "{{ item.path }}"
        mode: "{{ item.mode }}"
        owner: ansible
        group: ansible
      loop:
        - { path: /home/ansible/.ssh/id_ed25519, mode: "0600" }
        - { path: /home/ansible/.ssh/id_ed25519.pub, mode: "0644" }
        - { path: /home/ansible/.ssh, mode: "0700" }

    - name: Create ansible.cfg
      copy:
        content: |
          [defaults]
          inventory = /home/ansible/inventory
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
          ssh_args = -o ControlMaster=auto -o ControlPersist=60s
        dest: /home/ansible/ansible.cfg
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create inventory file
      copy:
        content: |
          [managed_nodes]
          servera.lab.example.com
          serverb.lab.example.com

          [all:vars]
          ansible_user=ansible
        dest: /home/ansible/inventory
        owner: ansible
        group: ansible
        mode: "0644"
```

### Managed Node Bootstrap Playbook

```yaml
---
- name: Bootstrap managed nodes for Ansible
  hosts: all
  become: yes
  gather_facts: yes
  vars:
    ansible_user_name: ansible
    ssh_key_path: /home/ansible/.ssh/id_ed25519.pub
    ssh_key_file: /home/ansible/.ssh/id_ed25519
  tasks:
    # Create Ansible user
    - name: Create Ansible user
      user:
        name: "{{ ansible_user_name }}"
        state: present
        createhome: yes
        shell: /bin/bash
        groups: wheel
        comment: "Ansible automation user"

    # Configure SSH directory
    - name: Create .ssh directory
      file:
        path: "/home/{{ ansible_user_name }}/.ssh"
        state: directory
        owner: "{{ ansible_user_name }}"
        group: "{{ ansible_user_name }}"
        mode: "0700"

    # Deploy SSH key
    - name: Deploy SSH public key
      authorized_key:
        user: "{{ ansible_user_name }}"
        state: present
        key: "{{ lookup('file', ssh_key_path) }}"
        manage_dir: yes
        exclusive: no

    # Set permissions
    - name: Set authorized_keys permissions
      file:
        path: "/home/{{ ansible_user_name }}/.ssh/authorized_keys"
        owner: "{{ ansible_user_name }}"
        group: "{{ ansible_user_name }}"
        mode: "0600"

    # Configure sudoers
    - name: Configure passwordless sudo
      copy:
        content: "{{ ansible_user_name }} ALL=(ALL) NOPASSWD: ALL"
        dest: /etc/sudoers.d/{{ ansible_user_name }}
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"

    # Ensure SSH is running
    - name: Ensure SSH server is running
      systemd:
        name: sshd
        state: started
        enabled: yes

    # Verify configuration
    - name: Verify SSH key deployment
      stat:
        path: "/home/{{ ansible_user_name }}/.ssh/authorized_keys"
      register: key_stat

    - name: Verify sudoers file
      stat:
        path: /etc/sudoers.d/{{ ansible_user_name }}
      register: sudoers_stat

    - name: Test Ansible connectivity
      ansible.builtin.ping:

    - name: Test privilege escalation
      command: whoami
      register: whoami_test
      changed_when: false
      failed_when: false

    # Display results
    - name: Display configuration status
      debug:
        msg: |
          SSH key: {{ 'deployed' if key_stat.stat.exists else 'missing' }}
          Sudoers: {{ 'configured' if sudoers_stat.stat.exists else 'missing' }}
          Connectivity: {{ 'OK' if whoami_test.rc == 0 else 'FAILED' }}
          User: {{ whoami_test.stdout if whoami_test.rc == 0 else 'N/A' }}
```

## RHCE Exam Notes

### Critical Exam Points

1. **SSH keys must work before any playbook runs**: This is the most common point of failure. Test connectivity early with `ansible all -m ping`.

2. **Know the `authorized_key` module**: This is the only module for deploying SSH keys. Know its parameters: `user`, `key`, `state`, `manage_dir`, `exclusive`.

3. **Sudoers validation is mandatory**: Always use `validate: "visudo -cf %s"` when writing sudoers files. A syntax error locks out sudo entirely.

4. **File permissions are critical**: `.ssh` directory must be `0700`, `authorized_keys` must be `0600`, private key must be `0600`.

5. **Ansible user must be in wheel group**: For sudo to work properly, the Ansible user must be in the wheel group.

6. **SSH service must be running**: Configure `sshd` to start and enable on all managed nodes.

7. **Use `lookup('file', path)` for keys**: This reads the public key from the control node without storing it as a variable.

8. **Host key checking**: Set `host_key_checking = false` in `ansible.cfg` for exam environments.

### Exam Workflow

1. Verify or create `ansible.cfg` with correct settings
2. Verify or generate SSH key pair on control node
3. Create inventory file
4. Run SSH key distribution playbook
5. Test connectivity with `ansible all -m ping`
6. Test privilege escalation with `ansible all -m ping -b`
7. Proceed with other automation tasks

### Common Exam Scenarios

- **No SSH access**: Create Ansible user, deploy key, configure sudoers
- **Permission denied**: Check file permissions on `.ssh` and `authorized_keys`
- **Sudo fails**: Verify sudoers file exists and has correct syntax
- **Host key failure**: Add `host_key_checking = false` to `ansible.cfg`

## Common Mistakes

### Mistake 1: Wrong File Permissions

```yaml
# WRONG - authorized_keys too permissive
- name: Deploy SSH key
  authorized_key:
    user: ansible
    state: present
    key: "{{ ssh_key }}"

# CORRECT - set permissions after deployment
- name: Deploy SSH key
  authorized_key:
    user: ansible
    state: present
    key: "{{ ssh_key }}"

- name: Set authorized_keys permissions
  file:
    path: /home/ansible/.ssh/authorized_keys
    mode: "0600"
```

### Mistake 2: Not Creating .ssh Directory

```yaml
# WRONG - authorized_key won't work without .ssh directory
- name: Deploy SSH key
  authorized_key:
    user: ansible
    state: present
    key: "{{ ssh_key }}"

# CORRECT - create .ssh directory first
- name: Create .ssh directory
  file:
    path: /home/ansible/.ssh
    state: directory
    mode: "0700"

- name: Deploy SSH key
  authorized_key:
    user: ansible
    state: present
    key: "{{ ssh_key }}"
    manage_dir: yes
```

### Mistake 3: Forgetting `manage_dir: yes`

```yaml
# WRONG - fails if .ssh directory doesn't exist
- name: Deploy SSH key
  authorized_key:
    user: ansible
    state: present
    key: "{{ ssh_key }}"

# CORRECT - automatically creates .ssh directory
- name: Deploy SSH key
  authorized_key:
    user: ansible
    state: present
    key: "{{ ssh_key }}"
    manage_dir: yes
```

### Mistake 4: Not Validating Sudoers Syntax

```yaml
# WRONG - may cause syntax errors
- name: Configure sudoers
  copy:
    content: "ansible ALL=(ALL) NOPASSWD: ALL"
    dest: /etc/sudoers.d/ansible
    mode: "0440"

# CORRECT - validate before writing
- name: Configure sudoers
  copy:
    content: "ansible ALL=(ALL) NOPASSWD: ALL"
    dest: /etc/sudoers.d/ansible
    mode: "0440"
    validate: "visudo -cf %s"
```

### Mistake 5: Adding to Wrong Sudoers File

```yaml
# WRONG - editing /etc/sudoers directly is dangerous
- name: Configure sudoers
  lineinfile:
    path: /etc/sudoers
    line: "ansible ALL=(ALL) NOPASSWD: ALL"
    validate: "visudo -cf %s"

# CORRECT - use /etc/sudoers.d/ directory
- name: Configure sudoers
  copy:
    content: "ansible ALL=(ALL) NOPASSWD: ALL"
    dest: /etc/sudoers.d/ansible
    validate: "visudo -cf %s"
```

## Chapter Summary

This chapter covered SSH key distribution to managed nodes:

- **SSH key generation** creates asymmetric key pairs for authentication. Ed25519 is the recommended algorithm.
- **SSH key distribution** uses the `authorized_key` module to deploy public keys to managed nodes.
- **SSH server configuration** ensures `sshd` is installed, running, and properly configured.
- **Sudoers configuration** enables passwordless sudo for the Ansible user using the `copy` module with `validate`.
- **Verification procedures** test SSH connectivity, key deployment, sudoers syntax, and privilege escalation.
- **Troubleshooting** addresses common issues: connection refused, authentication failed, key mismatches, sudoers syntax errors, and host key verification failures.
- **Real-world automation** includes complete control node setup and managed node bootstrap playbooks.

## Quick Reference

| Task | Module | Key Parameters |
|---|---|---|
| Generate SSH key | `openssh_keypair` | `path`, `type: ed25519` |
| Deploy SSH key | `authorized_key` | `user`, `key`, `state: present` |
| Set key permissions | `file` | `path`, `mode: "0600"`, `0644` |
| Create .ssh dir | `file` | `path`, `mode: "0700"` |
| Configure sudoers | `copy` | `dest`, `validate: "visudo -cf %s"` |
| Test SSH | `command` | `ssh -i key user@host` |
| Test Ansible | `ping` | `ansible all -m ping` |
| Test sudo | `command` | `sudo -l` |
| Check SSH service | `systemd` | `name: sshd state: started` |
| View authorized_keys | `command` | `cat ~/.ssh/authorized_keys` |

## Review Questions

1. What is the difference between the private key and public key in SSH authentication?

2. Which parameter in the `authorized_key` module automatically creates the `.ssh` directory if it doesn't exist?

3. What file permissions should be set on the SSH private key, public key, `.ssh` directory, and `authorized_keys` file?

4. Why is the `validate: "visudo -cf %s"` parameter critical when writing sudoers files?

5. What module is used to check if a file exists, and how would you check if `authorized_keys` exists?

6. What parameter in `ansible.cfg` disables SSH host key verification?

7. How do you test SSH connectivity to a managed node using Ansible?

8. What is the purpose of the `lookup('file', path)` plugin when deploying SSH keys?

9. Why must the Ansible user be in the wheel group for sudo to work?

10. What command would you use to check the syntax of a sudoers file before writing it?

## Answers

1. The private key is a secret key kept on the control node that proves identity. The public key is distributed to managed nodes and is used to verify the private key. During authentication, the control node presents the private key, and the managed node verifies it against the stored public key.

2. The `manage_dir: yes` parameter in the `authorized_key` module automatically creates the `.ssh` directory on the managed node if it doesn't exist. This ensures the module can deploy the key without manual directory creation.

3. The required file permissions are: SSH private key (`0600` - read/write for owner only), SSH public key (`0644` - readable by all), `.ssh` directory (`0700` - full access for owner only), and `authorized_keys` file (`0600` - read/write for owner only). These permissions ensure SSH accepts the keys while maintaining security.

4. The `validate: "visudo -cf %s"` parameter runs the `visudo -cf` command with the sudoers file path substituted for `%s` before writing the file. This checks the syntax without actually modifying `/etc/sudoers.d/ansible`. Without this validation, a syntax error could lock out all sudo access, requiring single-user mode recovery.

5. The `stat` module checks if a file exists. To check if `authorized_keys` exists, use: `stat: { path: "/home/ansible/.ssh/authorized_keys" }` and register the result to check `stat.stat.exists`.

6. The parameter `host_key_checking = false` in `ansible.cfg` disables SSH host key verification. This prevents SSH from prompting to accept new host keys, which would block non-interactive playbook execution in exam environments.

7. Use `ansible all -m ping` to test SSH connectivity to all managed nodes. If this returns `{ "ansible_facts": { "ansible_ssh_host": "hostname" }, "changed": false, "failed": false, "invocation": {}, "msg": "OK", "rc": 0, "stdout": "", "stdout_lines": [] }`, SSH connectivity is working.

8. The `lookup('file', path)` plugin reads file contents from the control node and returns them as a string. When deploying SSH keys, it reads the public key file from the control node without storing the key content as a playbook variable, which would be logged.

9. The Ansible user must be in the wheel group because, by default, only members of the wheel group can use sudo without a password. The sudoers configuration `ansible ALL=(ALL) NOPASSWD: ALL` works correctly only if the user is in wheel, or the sudoers entry must explicitly allow the user.

10. The command `visudo -cf /path/to/sudoers/file` checks the syntax of a sudoers file without editing it. The `-c` flag checks syntax only, and `-f` specifies the file path. This is used in the `validate` parameter to prevent syntax errors from locking out sudo access.
