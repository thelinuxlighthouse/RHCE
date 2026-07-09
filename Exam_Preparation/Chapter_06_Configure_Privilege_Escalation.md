# Chapter 6: Configure Privilege Escalation on Managed Nodes

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand privilege escalation and its necessity for Ansible
- Configure sudoers for the Ansible user on managed nodes
- Create and manage `/etc/sudoers.d/` files safely
- Use the `become` parameter in Ansible plays and tasks
- Configure privilege escalation in `ansible.cfg`
- Handle different sudoers syntax patterns
- Test and verify privilege escalation
- Troubleshoot privilege escalation failures
- Securely manage sensitive privilege escalation data

## Concept Overview

Privilege escalation allows the Ansible control node to execute tasks as root on managed nodes. Most system administration tasks require root privileges: installing packages, managing services, configuring firewalls, and modifying system files.

### Why Privilege Escalation Matters

Without privilege escalation:

- Ansible cannot install packages (requires root)
- Ansible cannot start/stop services (requires root)
- Ansible cannot modify system files (requires root)
- Ansible cannot configure firewalls (requires root)
- Most automation tasks fail

### How Privilege Escalation Works

```
Control Node                          Managed Node
─────────────                         ────────────
1. Ansible task requires root
   │
   └── become: yes
         │
         ├── Ansible executes as ansible user
         │
         └── sudo /bin/bash -c "command"
               │
               ├── sudo prompts for password (if configured)
               │
               └── executes as root

2. Module runs as root
   │
   ├── dnf, systemd, firewalld, etc.
   │
   └── Returns JSON result to Ansible
```

## Ansible Fundamentals Required

### Sudo Basics

Sudo (super-user do) allows users to run commands as another user (typically root):

```bash
# Regular user runs command as root
sudo command

# Sudo with password prompt
sudo -v  # Verify sudo access

# Sudo without password (NOPASSWD)
sudo -l  # List sudo privileges
```

### Sudoers File Format

```
# User/Group specification
username ALL=(ALL) NOPASSWD: ALL
%groupname ALL=(ALL) ALL

# Command restrictions
username ALL=(ALL) NOPASSWD: /usr/bin/systemctl
username ALL=(ALL) NOPASSWD: /usr/bin/dnf

# Hashed passwords
username ALL=(ALL) : (hashed password)
```

### Ansible Become Parameters

| Parameter | Purpose |
|---|---|
| `become: yes` | Enable privilege escalation |
| `become_method` | Method to use (sudo, su) |
| `become_user` | User to escalate to |
| `become_ask_pass` | Prompt for password |

## Commands and Tools

```bash
# Check sudo privileges
sudo -l

# Verify Ansible user can sudo
ansible all -m command -a "sudo whoami"

# Test privilege escalation
ansible all -m ping -b

# Check sudoers configuration
cat /etc/sudoers.d/ansible
visudo -cf /etc/sudoers.d/ansible

# View current sudoers
visudo -c

# List sudoers files
ls -la /etc/sudoers.d/

# Verify sudo password status
sudo -k  # Invalidate sudo cache
sudo -v  # Verify sudo access

# Check Ansible become settings
ansible-config dump | grep become

# Test become on specific task
ansible all -m command -a "whoami" -b

# Run playbook with become
ansible-playbook playbook.yml --become
```

## Playbook Examples

### Example 1: Configure Sudoers for Ansible User

```yaml
---
- name: Configure privilege escalation for Ansible user
  hosts: all
  become: yes
  gather_facts: yes
  vars:
    ansible_user_name: ansible
  tasks:
    - name: Ensure Ansible user exists
      user:
        name: "{{ ansible_user_name }}"
        state: present
        createhome: yes
        shell: /bin/bash
        groups: wheel

    - name: Add Ansible user to wheel group
      user:
        name: "{{ ansible_user_name }}"
        groups: wheel
        append: yes

    - name: Configure passwordless sudo for Ansible user
      copy:
        content: "{{ ansible_user_name }} ALL=(ALL) NOPASSWD: ALL"
        dest: /etc/sudoers.d/{{ ansible_user_name }}
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"
        backup: yes

    - name: Configure passwordless sudo for wheel group
      lineinfile:
        path: /etc/sudoers
        regexp: "^%wheel"
        line: "%wheel ALL=(ALL) NOPASSWD: ALL"
        state: present
        validate: "visudo -cf %s"
        backup: yes

    - name: Verify sudoers syntax
      command: visudo -cf /etc/sudoers.d/{{ ansible_user_name }}
      register: sudoers_test
      changed_when: false

    - name: Display sudoers configuration
      command: cat /etc/sudoers.d/{{ ansible_user_name }}
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
        when: sudo_test.rc == 0
```

### Example 2: Configure Privilege Escalation in ansible.cfg

```yaml
---
- name: Configure privilege escalation in ansible.cfg
  hosts: localhost
  connection: local
  become: yes
  tasks:
    - name: Create ansible.cfg with become settings
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

    - name: Verify ansible.cfg settings
      command: ansible-config dump --only-changed
      register: ansible_config
      changed_when: false

    - name: Display effective become settings
      debug:
        var: ansible_config.stdout_lines
```

### Example 3: Task-Level Privilege Escalation

```yaml
---
- name: Task-level privilege escalation
  hosts: all
  gather_facts: yes
  tasks:
    - name: Task without privilege escalation
      command: whoami
      register: no_become
      changed_when: false

    - name: Task with privilege escalation
      command: whoami
      become: yes
      register: with_become
      changed_when: false

    - name: Task with specific become user
      command: whoami
      become: yes
      become_user: apache
      register: specific_user
      changed_when: false

    - name: Display results
      debug:
        msg: |
          Without become: {{ no_become.stdout }}
          With become: {{ with_become.stdout }}
          With specific user: {{ specific_user.stdout }}

    - name: Task with become_method
      command: whoami
      become: yes
      become_method: sudo
      register: sudo_method
      changed_when: false

    - name: Task with become_ask_pass disabled
      command: whoami
      become: yes
      become_ask_pass: false
      register: no_prompt
      changed_when: false
```

### Example 4: Conditional Privilege Escalation

```yaml
---
- name: Conditional privilege escalation
  hosts: all
  gather_facts: yes
  vars:
    require_root: true
    require_apache: false
  tasks:
    - name: Task always requires root
      command: whoami
      become: "{{ require_root }}"
      changed_when: false

    - name: Task with conditional become
      command: whoami
      become: "{{ require_root }}"
      changed_when: false

    - name: Task with group-based become
      command: whoami
      become: "{{ 'yes' if inventory_hostname in groups.webservers else 'no' }}"
      changed_when: false

    - name: Task with fact-based become
      command: whoami
      become: "{{ ansible_os_family == 'RedHat' }}"
      changed_when: false

    - name: Task with variable-based become
      command: whoami
      become: "{{ need_root }}"
      changed_when: false
      vars:
        need_root: true
```

### Example 5: Multiple Privilege Escalation Methods

```yaml
---
- name: Multiple privilege escalation methods
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
    - name: Default sudo escalation
      command: whoami
      become: yes
      become_method: sudo
      become_user: root
      register: sudo_root
      changed_when: false

    - name: Alternative su escalation
      command: whoami
      become: yes
      become_method: su
      become_user: root
      register: su_root
      changed_when: false

    - name: Escalate to specific user
      command: whoami
      become: yes
      become_user: apache
      register: apache_user
      changed_when: false

    - name: Escalate with sudo but different user
      command: whoami
      become: yes
      become_method: sudo
      become_user: apache
      register: sudo_apache
      changed_when: false

    - name: Display escalation results
      debug:
        msg: |
          Sudo to root: {{ sudo_root.stdout }}
          Su to root: {{ su_root.stdout }}
          To apache: {{ apache_user.stdout }}
          Sudo to apache: {{ sudo_apache.stdout }}
```

### Example 6: Secure Sudoers Configuration

```yaml
---
- name: Secure sudoers configuration
  hosts: all
  become: yes
  gather_facts: yes
  vars:
    ansible_user_name: ansible
    admin_group: wheel
  tasks:
    - name: Create sudoers.d directory if not exists
      file:
        path: /etc/sudoers.d
        state: directory
        owner: root
        group: root
        mode: "0750"

    - name: Configure Ansible user sudoers
      copy:
        content: |
          # Ansible automation user
          {{ ansible_user_name }} ALL=(ALL) NOPASSWD: ALL
        dest: /etc/sudoers.d/{{ ansible_user_name }}
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"
        backup: yes

    - name: Configure admin group sudoers
      copy:
        content: |
          # Admin group sudoers
          {{ admin_group }} ALL=(ALL) NOPASSWD: ALL
        dest: /etc/sudoers.d/{{ admin_group }}
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"
        backup: yes

    - name: Configure restricted sudo for services
      copy:
        content: |
          # Service accounts with restricted sudo
          myapp ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp, /usr/bin/systemctl status myapp
        dest: /etc/sudoers.d/myapp
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"
        backup: yes

    - name: Configure backup user sudoers
      copy:
        content: |
          # Backup user with restricted sudo
          backup ALL=(ALL) NOPASSWD: /usr/bin/tar, /usr/bin/rsync, /usr/bin/ssh
        dest: /etc/sudoers.d/backup
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"
        backup: yes

    - name: Verify all sudoers files
      command: visudo -cf /etc/sudoers.d/{{ item }}
      loop:
        - "{{ ansible_user_name }}"
        - "{{ admin_group }}"
        - myapp
        - backup
      register: sudoers_tests
      changed_when: false
      failed_when: false

    - name: Display sudoers verification results
      debug:
        msg: "{{ 'Syntax OK' if item.item.rc == 0 else 'Syntax Error' }}"
      loop: "{{ sudoers_tests.results }}"
      loop_control:
        label: "{{ item.item.dest | basename }}"
```

### Example 7: Complete Privilege Escalation Setup

```yaml
---
- name: Complete privilege escalation setup
  hosts: all
  become: yes
  gather_facts: yes
  vars:
    ansible_user_name: ansible
    ansible_user_password: "{{ lookup('password', '/dev/null length=20 chars=ascii_letters,digits') }}"
  tasks:
    # Create Ansible user
    - name: Create Ansible user
      user:
        name: "{{ ansible_user_name }}"
        state: present
        createhome: yes
        shell: /bin/bash
        groups: wheel
        password: "{{ ansible_user_password }}"
        password_update: no

    # Configure SSH for Ansible user
    - name: Ensure SSH service is running
      systemd:
        name: sshd
        state: started
        enabled: yes

    # Configure sudoers
    - name: Configure passwordless sudo
      copy:
        content: "{{ ansible_user_name }} ALL=(ALL) NOPASSWD: ALL"
        dest: /etc/sudoers.d/{{ ansible_user_name }}
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"
        backup: yes

    # Test privilege escalation
    - name: Test privilege escalation
      command: whoami
      become: yes
      register: become_test
      changed_when: false

    - name: Test sudo access
      command: sudo -l
      register: sudo_test
      changed_when: false
      failed_when: false

    - name: Verify Ansible user can sudo
      command: ansible all -m ping -b
      register: ansible_ping
      changed_when: false
      failed_when: false

    # Display results
    - name: Display privilege escalation status
      debug:
        msg: |
          Ansible user: {{ ansible_user_name }}
          Sudo test: {{ 'PASSED' if become_test.rc == 0 else 'FAILED' }}
          Sudo access: {{ 'CONFIGURED' if sudo_test.rc == 0 else 'NOT CONFIGURED' }}
          Ansible ping: {{ 'OK' if ansible_ping.rc == 0 else 'FAILED' }}
          Become user: {{ become_test.stdout }}
```

## Explanation of Each Playbook

### Playbook 1: Configure Sudoers

This playbook sets up passwordless sudo for the Ansible user. It creates the Ansible user if it doesn't exist, adds the user to the wheel group, and writes a sudoers file with the `NOPASSWD` tag. The `validate: "visudo -cf %s"` parameter is critical for preventing syntax errors that could lock out sudo access.

### Playbook 2: ansible.cfg Configuration

This playbook creates `ansible.cfg` with the `[privilege_escalation]` section. The `become = true` setting enables privilege escalation by default for all tasks. The `become_method = sudo` specifies sudo as the escalation method. The `become_user = root` escalates to root. The `become_ask_pass = false` prevents password prompts.

### Playbook 3: Task-Level Escalation

This playbook demonstrates different levels of privilege escalation. The first task runs without escalation (as the Ansible user). The second task uses `become: yes` to escalate to root. The third task escalates to a specific user (apache). The `become_method` parameter specifies the escalation method, and `become_ask_pass` controls whether to prompt for a password.

### Playbook 4: Conditional Escalation

This playbook shows how to conditionally apply privilege escalation. The `when` parameter with `become` allows different tasks to require root based on variables, facts, or host groups. This is useful when some tasks need root and others don't.

### Playbook 5: Multiple Escalation Methods

This playbook demonstrates different privilege escalation methods. The `become_method: sudo` uses sudo, while `become_method: su` uses su. Escalating to different users (root, apache) shows how to target specific accounts. The `become_user` parameter specifies the target user.

### Playbook 6: Secure Sudoers Configuration

This playbook creates multiple sudoers files in `/etc/sudoers.d/` for different users and groups. It demonstrates how to restrict sudo access to specific commands (like `/usr/bin/systemctl restart myapp`). The `validate` parameter ensures each sudoers file has correct syntax.

### Playbook 7: Complete Setup

This comprehensive playbook creates the Ansible user, configures sudoers, ensures SSH is running, and tests privilege escalation. It uses the `lookup('password')` plugin to generate a random password for the Ansible user.

## Variables and Templates

### Using Variables for Sudoers Configuration

```yaml
---
- name: Sudoers with variables
  hosts: all
  become: yes
  vars:
    ansible_user_name: ansible
    admin_group: wheel
  tasks:
    - name: Configure sudoers
      copy:
        content: "{{ ansible_user_name }} ALL=(ALL) NOPASSWD: ALL"
        dest: /etc/sudoers.d/{{ ansible_user_name }}
        mode: "0440"
        validate: "visudo -cf %s"
```

### Using Jinja2 Templates for Sudoers

```yaml
# Template: sudoers.d/custom.conf.j2
# Custom sudoers configuration for {{ inventory_hostname }}

# Web server users
{% for user in web_users | default([]) %}
{{ user }} ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart {{ item.service }}
{% endfor %}

# Admin group
{% if admin_group | default(wheel) %}
{{ admin_group }} ALL=(ALL) NOPASSWD: ALL
{% endif %}

# Restricted sudo for backup
backup ALL=(ALL) NOPASSWD: /usr/bin/tar, /usr/bin/rsync
```

```yaml
# Playbook using template
---
- name: Deploy custom sudoers
  hosts: all
  become: yes
  vars:
    web_users:
      - www-data
      - apache
    admin_group: wheel
  tasks:
    - name: Deploy custom sudoers
      template:
        src: templates/sudoers.d/custom.conf.j2
        dest: /etc/sudoers.d/custom
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"
      notify: reload sudoers
```

### Using Facts for Conditional Sudoers

```yaml
---
- name: Sudoers with facts
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
    - name: Configure sudoers for Red Hat systems
      copy:
        content: "ansible ALL=(ALL) NOPASSWD: ALL"
        dest: /etc/sudoers.d/ansible
        mode: "0440"
        validate: "visudo -cf %s"
      when: ansible_os_family == "RedHat"
```

## Verification Procedures

### Verify Sudoers Configuration

```bash
# Check sudoers file exists
ls -la /etc/sudoers.d/ansible

# Check sudoers content
cat /etc/sudoers.d/ansible

# Verify sudoers syntax
visudo -cf /etc/sudoers.d/ansible

# Check file permissions
stat /etc/sudoers.d/ansible

# Test sudo access
ansible all -m command -a "sudo whoami"

# Display sudo privileges
ansible all -m command -a "sudo -l"

# Check Ansible user in wheel group
ansible all -m command -a "groups ansible"
```

### Verify Privilege Escalation

```bash
# Test become mode
ansible all -m ping -b

# Test specific task with become
ansible all -m command -a "whoami" -b

# Run playbook with verbose become
ansible-playbook playbook.yml -vvv
ansible-playbook playbook.yml --become --ask-become-pass

# Check ansible.cfg become settings
ansible-config dump | grep become

# Verify sudoers validation
visudo -cf /etc/sudoers.d/ansible
```

### Verify Ansible User

```bash
# Check Ansible user exists
id ansible

# Check Ansible user groups
groups ansible

# Check Ansible user home directory
ls -la /home/ansible

# Verify SSH access
ssh ansible@servera.lab.example.com "whoami"
```

## Troubleshooting

### Sudoers Syntax Errors

**Problem: visudo reports syntax error**

```bash
# Solution: Check syntax before writing
visudo -cf /etc/sudoers.d/ansible

# Common errors:
# - Missing space around operators
# - Invalid user/group names
# - Wrong file permissions (must be 0440)
# - Missing newline at end of file
```

**Ansible fix:**
```yaml
- name: Configure sudoers
  copy:
    content: "ansible ALL=(ALL) NOPASSWD: ALL"
    dest: /etc/sudoers.d/ansible
    mode: "0440"
    validate: "visudo -cf %s"
```

### Ansible User Not in Wheel Group

**Problem: Sudo fails because user not in wheel**

```bash
# Solution: Add user to wheel group
usermod -aG wheel ansible

# Solution: Configure sudoers for specific user
ansible ALL=(ALL) NOPASSWD: ALL
```

**Ansible playbook:**
```yaml
- name: Add Ansible user to wheel group
  user:
    name: ansible
    groups: wheel
    append: yes
```

### Become Not Working

**Problem: Tasks fail with become**

```bash
# Solution: Check ansible.cfg settings
ansible-config dump | grep become

# Solution: Add become to task
- name: Task with become
  command: whoami
  become: yes
  become_user: root

# Solution: Add become to play
- name: Play with become
  hosts: all
  become: yes
  tasks:
    - name: Task
      command: whoami
```

### Password Prompt Issues

**Problem: Ansible prompts for sudo password**

```bash
# Solution: Set become_ask_pass = false in ansible.cfg
[privilege_escalation]
become_ask_pass = false

# Solution: Add --become-user flag
ansible-playbook playbook.yml --become-user root

# Solution: Configure sudoers with NOPASSWD
ansible ALL=(ALL) NOPASSWD: ALL
```

### Sudo Caching Issues

**Problem: Sudo access lost after timeout**

```bash
# Solution: Invalidate sudo cache
sudo -k

# Solution: Verify sudo access
sudo -v

# Solution: Add sudo -v to playbook tasks
- name: Refresh sudo access
  command: sudo -v
```

### Multiple Sudoers Files Conflict

**Problem: Multiple sudoers files with conflicting entries**

```bash
# Solution: List all sudoers files
ls -la /etc/sudoers.d/

# Solution: Check each file
cat /etc/sudoers.d/*

# Solution: Use specific sudoers file per user
# /etc/sudoers.d/ansible
# /etc/sudoers.d/wheel
# /etc/sudoers.d/admins
```

## Real-World Automation Examples

### Complete Privilege Escalation Infrastructure

```yaml
---
- name: Set up complete privilege escalation infrastructure
  hosts: all
  become: yes
  gather_facts: yes
  vars:
    ansible_user_name: ansible
    admin_group: wheel
  tasks:
    # Create sudoers.d directory
    - name: Ensure sudoers.d directory exists
      file:
        path: /etc/sudoers.d
        state: directory
        owner: root
        group: root
        mode: "0750"

    # Create Ansible user
    - name: Create Ansible automation user
      user:
        name: "{{ ansible_user_name }}"
        state: present
        createhome: yes
        shell: /bin/bash
        groups: "{{ admin_group }}"
        comment: "Ansible automation user"

    # Configure Ansible user sudoers
    - name: Configure Ansible user sudoers
      copy:
        content: |
          # Ansible automation user
          # Managed by Ansible - Do not edit manually
          {{ ansible_user_name }} ALL=(ALL) NOPASSWD: ALL
        dest: /etc/sudoers.d/{{ ansible_user_name }}
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"
        backup: yes

    # Configure admin group sudoers
    - name: Configure admin group sudoers
      copy:
        content: |
          # Admin group sudoers
          # Managed by Ansible - Do not edit manually
          {{ admin_group }} ALL=(ALL) NOPASSWD: ALL
        dest: /etc/sudoers.d/{{ admin_group }}
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"
        backup: yes

    # Configure restricted sudo for backup
    - name: Configure backup user sudoers
      copy:
        content: |
          # Backup user with restricted sudo
          # Managed by Ansible - Do not edit manually
          backup ALL=(ALL) NOPASSWD: /usr/bin/tar, /usr/bin/rsync, /usr/bin/ssh
        dest: /etc/sudoers.d/backup
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"
        backup: yes

    # Configure service account sudoers
    - name: Configure service account sudoers
      copy:
        content: |
          # Service account with restricted sudo
          # Managed by Ansible - Do not edit manually
          myapp ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp, /usr/bin/systemctl status myapp
        dest: /etc/sudoers.d/myapp
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"
        backup: yes

    # Ensure SSH is running
    - name: Ensure SSH service is running
      systemd:
        name: sshd
        state: started
        enabled: yes

    # Verify sudoers syntax
    - name: Verify all sudoers files
      command: visudo -cf /etc/sudoers.d/{{ item }}
      loop:
        - "{{ ansible_user_name }}"
        - "{{ admin_group }}"
        - backup
        - myapp
      register: sudoers_tests
      changed_when: false
      failed_when: false

    # Test privilege escalation
    - name: Test privilege escalation
      command: whoami
      become: yes
      register: become_test
      changed_when: false

    # Display results
    - name: Display privilege escalation status
      debug:
        msg: |
          Ansible user: {{ ansible_user_name }}
          Sudoers: {{ 'CONFIGURED' if become_test.rc == 0 else 'NOT CONFIGURED' }}
          Become test: {{ 'PASSED' if become_test.rc == 0 else 'FAILED' }}
```

## RHCE Exam Notes

### Critical Exam Points

1. **Sudoers validation is mandatory**: Always use `validate: "visudo -cf %s"` when writing sudoers files. A syntax error locks out sudo entirely.

2. **File permissions are critical**: Sudoers files must have `0440` permissions (readable by owner and group, not by others).

3. **Ansible user must be in wheel group**: For proper sudo functionality, the Ansible user must be in the wheel group.

4. **`become: yes` at play level**: Adding `become: yes` to a play enables privilege escalation for all tasks in that play by default.

5. **Task-level `become` overrides play-level**: A task can explicitly disable `become` even if the play has `become: yes`.

6. **`become_ask_pass = false`**: Set this in `ansible.cfg` to prevent password prompts during non-interactive playbook execution.

7. **`sudo -l` testing**: Use `sudo -l` to verify sudo privileges are configured correctly on managed nodes.

8. **`ansible all -m ping -b`**: This is the standard test for privilege escalation. If it returns success, sudo is working.

### Exam Workflow

1. Create or verify `ansible.cfg` with `become: yes` and `become_ask_pass: false`
2. Configure sudoers on managed nodes with validation
3. Test with `ansible all -m ping -b`
4. Proceed with other automation tasks

### Common Exam Scenarios

- **Sudo fails**: Check sudoers file exists, has correct permissions, and valid syntax
- **Password prompt**: Check `become_ask_pass` setting and sudoers NOPASSWD configuration
- **Not in wheel group**: Add user to wheel group
- **Syntax error**: Use `validate: "visudo -cf %s"` parameter

## Common Mistakes

### Mistake 1: Wrong Sudoers File Permissions

```yaml
# WRONG - too permissive
- name: Configure sudoers
  copy:
    content: "ansible ALL=(ALL) NOPASSWD: ALL"
    dest: /etc/sudoers.d/ansible
    mode: "0644"

# CORRECT - must be 0440
- name: Configure sudoers
  copy:
    content: "ansible ALL=(ALL) NOPASSWD: ALL"
    dest: /etc/sudoers.d/ansible
    mode: "0440"
```

### Mistake 2: Not Validating Sudoers Syntax

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

### Mistake 3: Forgetting to Add User to Wheel Group

```yaml
# WRONG - user may not have sudo access
- name: Create Ansible user
  user:
    name: ansible
    state: present

# CORRECT - add to wheel group
- name: Create Ansible user
  user:
    name: ansible
    state: present
    groups: wheel
```

### Mistake 4: Not Setting become in Play

```yaml
# WRONG - tasks won't escalate
- name: Configure servers
  hosts: all
  tasks:
    - name: Install package
      dnf:
        name: httpd
        state: present

# CORRECT - enable become
- name: Configure servers
  hosts: all
  become: yes
  tasks:
    - name: Install package
      dnf:
        name: httpd
        state: present
```

### Mistake 5: Using Wrong Sudoers File Path

```yaml
# WRONG - editing /etc/sudoers directly is dangerous
- name: Configure sudoers
  lineinfile:
    path: /etc/sudoers
    line: "ansible ALL=(ALL) NOPASSWD: ALL"
    validate: "visudo -cf %s"

# CORRECT - use /etc/sudoers.d/
- name: Configure sudoers
  copy:
    content: "ansible ALL=(ALL) NOPASSWD: ALL"
    dest: /etc/sudoers.d/ansible
    validate: "visudo -cf %s"
```

## Chapter Summary

This chapter covered privilege escalation configuration on managed nodes:

- **Sudoers configuration** creates passwordless sudo access for the Ansible user using the `copy` module with `validate: "visudo -cf %s"`.
- **ansible.cfg settings** control default privilege escalation behavior with `become`, `become_method`, `become_user`, and `become_ask_pass`.
- **Task-level escalation** allows individual tasks to enable or disable privilege escalation.
- **Conditional escalation** uses variables and facts to determine when to escalate.
- **Multiple escalation methods** support both `sudo` and `su` as escalation mechanisms.
- **Secure sudoers** demonstrates how to restrict sudo access to specific commands for service accounts.
- **Verification procedures** test sudoers syntax, sudo access, and privilege escalation.
- **Troubleshooting** addresses syntax errors, missing wheel group membership, and password prompt issues.

## Quick Reference

| Parameter | Purpose | Example |
|---|---|---|
| `become: yes` | Enable privilege escalation | `become: yes` |
| `become_method` | Escalation method | `become_method: sudo` |
| `become_user` | Target user | `become_user: root` |
| `become_ask_pass` | Prompt for password | `become_ask_pass: false` |
| `validate` | Validate before writing | `validate: "visudo -cf %s"` |
| `dest` | Sudoers file path | `dest: /etc/sudoers.d/ansible` |
| `mode` | File permissions | `mode: "0440"` |
| `content` | Sudoers content | `content: "ansible ALL=(ALL) NOPASSWD: ALL"` |
| `backup` | Create backup | `backup: yes` |

## Review Questions

1. What is the purpose of the `validate` parameter when writing sudoers files, and why is it critical?

2. What file permissions should sudoers files have, and why?

3. How does the `become: yes` parameter in a play affect individual tasks?

4. What is the difference between `become_method: sudo` and `become_method: su`?

5. Why must the Ansible user be in the wheel group for sudo to work properly?

6. How do you test if privilege escalation is working on managed nodes?

7. What parameter in `ansible.cfg` prevents password prompts during playbook execution?

8. What is the purpose of the `backup: yes` parameter when writing sudoers files?

9. How would you configure sudoers to allow a user to run only specific commands?

10. What command would you use to check the syntax of a sudoers file?

## Answers

1. The `validate` parameter runs a command to check the syntax of a file before it is written. For sudoers files, `validate: "visudo -cf %s"` checks the syntax without modifying the file. The `%s` placeholder is replaced with the path to a temporary file. This is critical because a syntax error in sudoers can lock out all sudo access, requiring single-user mode recovery.

2. Sudoers files must have `0440` permissions (readable by owner and group, not by others, writable only by owner). This ensures that only root can modify sudoers files while allowing the owner (root) to read them. The permissions are enforced by the `mode: "0440"` parameter in the `copy` module.

3. The `become: yes` parameter in a play enables privilege escalation by default for all tasks in that play. However, individual tasks can override this by explicitly setting `become: no` to disable escalation for that specific task. A task with `become: no` in a play with `become: yes` will run without privilege escalation.

4. `become_method: sudo` uses the sudo command to escalate privileges, which is the standard method on most Linux systems. `become_method: su` uses the su command, which switches to a different user account. Sudo is preferred because it allows running specific commands as root without giving the user full root login access.

5. The Ansible user must be in the wheel group because, by default, only members of the wheel group can use sudo without a password. The sudoers configuration `ansible ALL=(ALL) NOPASSWD: ALL` works correctly only if the user is in wheel, or the sudoers entry must explicitly allow the user. Adding the user to wheel ensures proper sudo functionality across different systems.

6. Use `ansible all -m ping -b` to test privilege escalation. The `-b` flag enables become mode. If this command returns success with the message "OK", privilege escalation is working correctly. You can also use `ansible all -m command -a "whoami" -b` to verify that tasks run as root.

7. The parameter `become_ask_pass = false` in `ansible.cfg` prevents password prompts during non-interactive playbook execution. When set to `true` (the default), Ansible will prompt for the sudo password if the user doesn't have NOPASSWD configured in sudoers.

8. The `backup: yes` parameter creates a backup of the existing sudoers file before overwriting it. This is important for sudoers files because a syntax error could lock out sudo access. The backup allows recovery if the new configuration is incorrect.

9. To restrict sudo access to specific commands, configure sudoers like: `username ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp, /usr/bin/systemctl status myapp`. This allows the user to run only the specified commands as root. Use multiple commands separated by commas in the same sudoers entry.

10. The command `visudo -cf /path/to/sudoers/file` checks the syntax of a sudoers file without editing it. The `-c` flag checks syntax only, and `-f` specifies the file path. This is used in the `validate` parameter to prevent syntax errors from locking out sudo access.
