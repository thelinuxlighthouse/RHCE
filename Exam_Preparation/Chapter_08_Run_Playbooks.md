# Chapter 8: Run Playbooks


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


## Added Review: Playbook Execution Workflow

Before running a playbook for real:

```bash
ansible-playbook --syntax-check site.yml
ansible-playbook --check --diff site.yml
ansible-playbook site.yml
ansible-playbook site.yml
```

The second real run is important. A good playbook usually reports fewer or no changes on the second run.

For navigator stdout mode:

```bash
ansible-navigator run site.yml -m stdout
ansible-navigator run site.yml -m stdout --ee false
```

For execution-environment mode, verify which image and mounts are active:

```bash
ansible-navigator config -m stdout
ansible-navigator images -m stdout
```

Use navigator interactive mode when you need to inspect task results deeply or replay artifacts.


## Learning Objectives

By the end of this chapter, you will be able to:

- Run playbooks with `ansible-playbook` command
- Run playbooks with `ansible-navigator` interface
- Use playbooks with various execution options and flags
- Use ansible-navigator to discover and use new modules
- Configure ansible-navigator for optimal workflow
- Create and manage inventories using ansible-navigator
- Perform basic Git operations for version control
- Clone Git repositories for Ansible playbooks
- Add files to Git repositories
- Create playbooks and push them to Git
- Configure ansible-navigator in VS Code
- Run playbooks using Ansible development containers

## Concept Overview

Running playbooks is the core Ansible workflow. Playbooks define automation tasks that are executed against managed nodes. Understanding how to run playbooks with different options, troubleshoot execution issues, and integrate with version control is essential for effective automation.

### Why Playbook Execution Matters

Without proper playbook execution:

- Automation tasks are not performed
- Configuration changes are not applied
- Deployments fail
- No audit trail of changes
- No version control for automation

### How Playbook Execution Works

```
Playbook Execution Flow:
─────────────────────────────────────────────────────────────────
1. Ansible reads playbook YAML
   │
   └── Parses tasks, plays, handlers

2. Reads inventory
   │
   └── Identifies target hosts

3. Gathers facts (if enabled)
   │
   └── Collects system information

4. Executes tasks
   │
   ├── Connects via SSH
   │
   ├── Runs modules on managed nodes
   │
   └── Collects results

5. Reports results
   │
   ├── Success/failure status
   │
   ├── Changed/not changed
   │
   └── Output to terminal or logs
```

## Ansible Fundamentals Required

### Playbook Structure

```yaml
---
- name: Play description
  hosts: target_group
  become: true
  tasks:
    - name: Task description
      module:
        parameter: value
  handlers:
    - name: Handler description
      module:
        parameter: value
```

### ansible-playbook vs ansible-navigator

| Tool | Use Case | Interface |
|---|---|---|
| `ansible-playbook` | CLI execution, scripting | Command line |
| `ansible-navigator` | Interactive execution, exploration | TUI (Terminal User Interface) |

### Execution Modes

```bash
# Dry run (no changes)
ansible-playbook playbook.yml --check

# Dry run with diff
ansible-playbook playbook.yml --check --diff

# Syntax check only
ansible-playbook playbook.yml --syntax-check

# Verbose output
ansible-playbook playbook.yml -vvv

# Limit to hosts
ansible-playbook playbook.yml --limit servera

# Run specific tags
ansible-playbook playbook.yml --tags packages

# Skip specific tags
ansible-playbook playbook.yml --skip-tags reboot
```

## Commands and Tools

```bash
# Run playbook with ansible-playbook
ansible-playbook /path/to/playbook.yml
ansible-playbook -i inventory playbook.yml
ansible-playbook playbook.yml -e "var=value"

# Run with ansible-navigator
ansible-navigator run playbook.yml
ansible-navigator run playbook.yml --mode stdout
ansible-navigator run playbook.yml --inventory inventory

# Syntax validation
ansible-playbook playbook.yml --syntax-check
ansible-playbook playbook.yml --check

# Verbose execution
ansible-playbook playbook.yml -v
ansible-playbook playbook.yml -vv
ansible-playbook playbook.yml -vvv

# List available modules
ansible-doc -l
ansible-doc -l | grep firewall
ansible-doc firewalld

# View module documentation
ansible-doc -s firewalld
ansible-doc dnf
ansible-doc systemd

# Search module parameters
ansible-doc -l | grep -i package

# Create inventory with ansible-inventory
ansible-inventory --list
ansible-inventory --graph
ansible-inventory --host servera

# Git operations
git clone https://github.com/user/repo.git
git add .
git commit -m "message"
git push origin main

# VS Code ansible-navigator
code .
# Open ansible-navigator from VS Code
# Run playbook in development container
```

## Playbook Examples

### Example 1: Basic Playbook Execution

```yaml
---
- name: Basic playbook execution
  hosts: all
  become: true
  gather_facts: yes
  tasks:
    - name: Display hostname
      debug:
        msg: "Running on {{ inventory_hostname }}"

    - name: Display OS information
      debug:
        msg: "{{ ansible_distribution }} {{ ansible_distribution_version }}"

    - name: Ping all hosts
      ping:

    - name: Show current user
      command: whoami
      register: current_user
      changed_when: false

    - name: Display current user
      debug:
        msg: "Current user: {{ current_user.stdout }}"

    - name: Show Python version
      command: python3 --version
      register: python_version
      changed_when: false

    - name: Display Python version
      debug:
        msg: "Python version: {{ python_version.stdout }}"
```

**Execution:**
```bash
# Basic execution
ansible-playbook basic_playbook.yml

# With verbose output
ansible-playbook basic_playbook.yml -vvv

# Limit to specific hosts
ansible-playbook basic_playbook.yml --limit servera.lab.example.com

# Dry run
ansible-playbook basic_playbook.yml --check
```

### Example 2: Playbook with Tags

```yaml
---
- name: Playbook with tags
  hosts: all
  become: true
  gather_facts: yes
  tasks:
    - name: Update package cache
      dnf:
        name: "*"
        state: present
        update_cache: yes
      tags:
        - packages
        - cache

    - name: Install common packages
      dnf:
        name:
          - vim-enhanced
          - net-tools
          - wget
          - curl
        state: present
      tags: packages

    - name: Configure firewall
      firewalld:
        port: "80/tcp"
        permanent: yes
        immediate: yes
        state: enabled
      tags: firewall

    - name: Start services
      systemd:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop:
        - firewalld
        - sshd
      tags: services

    - name: Verify configuration
      command: hostname
      register: hostname_check
      changed_when: false
      tags: always
```

**Execution:**
```bash
# Run only package tasks
ansible-playbook tagged_playbook.yml --tags packages

# Run only firewall tasks
ansible-playbook tagged_playbook.yml --tags firewall

# Run specific tags
ansible-playbook tagged_playbook.yml --tags "packages,firewall"

# Skip reboot tasks
ansible-playbook tagged_playbook.yml --skip-tags reboot

# Run all tasks
ansible-playbook tagged_playbook.yml
```

### Example 3: Playbook with Error Handling

```yaml
---
- name: Playbook with error handling
  hosts: all
  become: true
  gather_facts: yes
  tasks:
    - name: Install httpd package
      dnf:
        name: httpd
        state: present
      ignore_errors: true
      register: httpd_install
      changed_when: false

    - name: Check if httpd installation succeeded
      debug:
        msg: "HTTPD installed successfully"
      when: httpd_install.rc == 0

    - name: Start httpd service
      systemd:
        name: httpd
        state: started
        enabled: yes
      when: httpd_install.rc == 0

    - name: Alternative task if httpd failed
      debug:
        msg: "HTTPD installation failed, using alternative"
      when: httpd_install.rc != 0

    - name: Verify httpd status
      command: systemctl is-active httpd
      register: httpd_status
      changed_when: false
      failed_when: false

    - name: Display httpd status
      debug:
        msg: "HTTPD status: {{ httpd_status.stdout }}"

    - name: Block with error handling
      block:
        - name: Install critical package
          dnf:
            name: vim-enhanced
            state: present

        - name: Deploy configuration
          copy:
            content: "config"
            dest: /etc/config
      rescue:
        - name: Rollback on failure
          debug:
            msg: "Configuration failed, rolling back"
        - name: Remove configuration
          file:
            path: /etc/config
            state: absent
      always:
        - name: Log completion
          debug:
            msg: "Block execution completed"
```

**Execution:**
```bash
# Run with error handling
ansible-playbook error_handling_playbook.yml

# View full error details
ansible-playbook error_handling_playbook.yml -vvv
```

### Example 4: Playbook with Loops and Conditionals

```yaml
---
- name: Playbook with loops and conditionals
  hosts: all
  become: true
  gather_facts: yes
  vars:
    packages_to_install:
      - vim-enhanced
      - net-tools
      - wget
      - curl
      - tree
    services_to_start:
      - firewalld
      - sshd
      - httpd
  tasks:
    - name: Install packages
      dnf:
        name: "{{ item }}"
        state: present
      loop: "{{ packages_to_install }}"
      loop_control:
        label: "{{ item }}"
      tags: packages

    - name: Start services based on OS
      systemd:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop: "{{ services_to_start }}"
      when: ansible_os_family == "RedHat"
      tags: services

    - name: Open firewall ports
      firewalld:
        port: "{{ item }}"
        permanent: yes
        immediate: yes
        state: enabled
      loop:
        - "80/tcp"
        - "443/tcp"
        - "8080/tcp"
      when: ansible_facts.services.firewalld.service is defined
      tags: firewall

    - name: Conditional task based on fact
      debug:
        msg: "Running on {{ ansible_distribution }}"
      when: ansible_distribution == "RedHat"

    - name: Conditional task based on variable
      debug:
        msg: "Debug mode enabled"
      when: debug_mode | default(false)

    - name: Install package if not present
      dnf:
        name: httpd
        state: present
      when:
        - ansible_facts.packages.httpd is defined
        - ansible_facts.packages.httpd.state != "present"
```

**Execution:**
```bash
# Run with specific tags
ansible-playbook loops_conditional_playbook.yml --tags packages,firewall

# Run with verbose output
ansible-playbook loops_conditional_playbook.yml -vvv

# Run dry run
ansible-playbook loops_conditional_playbook.yml --check
```

### Example 5: Playbook Using ansible-navigator

```yaml
---
- name: Playbook for ansible-navigator
  hosts: all
  become: true
  gather_facts: yes
  tasks:
    - name: Display system information
      debug:
        msg: "Host: {{ inventory_hostname }}"

    - name: Install packages
      dnf:
        name:
          - vim-enhanced
          - net-tools
        state: present
      tags: packages

    - name: Configure firewall
      firewalld:
        port: "80/tcp"
        permanent: yes
        immediate: yes
        state: enabled
      tags: firewall

    - name: Start services
      systemd:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop:
        - firewalld
        - sshd
      tags: services

    - name: Verify configuration
      ping:
      tags: verify
```

**ansible-navigator Execution:**
```bash
# Run playbook in ansible-navigator
ansible-navigator run playbook.yml

# Run with specific mode
ansible-navigator run playbook.yml --mode stdout

# Run with inventory
ansible-navigator run playbook.yml --inventory /path/to/inventory

# Execute ad hoc command
ansible-navigator execute all -m ping

# View documentation
ansible-navigator documentation

# Open interactive TUI
ansible-navigator
```

## Explanation of Each Playbook

### Playbook 1: Basic Execution

This playbook demonstrates basic playbook execution. It uses the `debug` module to display information, the `command` module to run shell commands, and the `ping` module to test connectivity. The `register` variable captures command output for later use. The `changed_when: false` parameter prevents tasks from reporting changes when they don't modify system state.

### Playbook 2: Tagged Execution

This playbook demonstrates the use of tags to control task execution. Tags allow selective task execution without modifying the playbook. The `--tags` parameter runs only tasks with matching tags. The `--skip-tags` parameter skips tasks with matching tags. The `tags: always` ensures a task runs regardless of other tag filters.

### Playbook 3: Error Handling

This playbook demonstrates error handling with `ignore_errors`, `block/rescue/always`, and conditional execution. The `ignore_errors: true` parameter continues execution even if a task fails. The `block/rescue/always` structure provides structured error handling similar to try-catch-finally. The `when` parameter controls task execution based on conditions.

### Playbook 4: Loops and Conditionals

This playbook demonstrates `loop` for iterating over lists and `when` for conditional execution. The `loop_control.label` parameter provides better output labels. The `loop` module eliminates task duplication. The `when` parameter enables intelligent task execution based on facts, variables, and registered outputs.

### Playbook 5: ansible-navigator Execution

This playbook is optimized for ansible-navigator execution. It uses tags for selective execution and includes verification tasks. The `ansible-navigator` interface provides real-time execution output, documentation browsing, and inventory management.

## Variables and Templates

### Using Variables in Playbook Execution

```yaml
---
- name: Playbook with variables
  hosts: all
  become: true
  vars:
    app_name: myapp
    app_port: 8080
  tasks:
    - name: Deploy configuration
      copy:
        content: |
          [application]
          name = {{ app_name }}
          port = {{ app_port }}
        dest: /etc/{{ app_name }}/config.ini
```

**Execution with extra variables:**
```bash
ansible-playbook playbook.yml -e "app_name=myapp app_port=8080"
ansible-playbook playbook.yml -e "@vars/production.yml"
```

### Using Facts in Playbook Execution

```yaml
---
- name: Playbook using facts
  hosts: all
  become: true
  gather_facts: yes
  tasks:
    - name: Display facts
      debug:
        msg: "Distribution: {{ ansible_distribution }}"
        msg: "Version: {{ ansible_distribution_version }}"
        msg: "CPU Cores: {{ ansible_processor_vcpus }}"
        msg: "Memory: {{ ansible_memtotal_mb }} MB"

    - name: Conditional based on fact
      dnf:
        name: httpd
        state: present
      when: ansible_os_family == "RedHat"

    - name: Alternative for Debian
      apt:
        name: httpd
        state: present
      when: ansible_os_family == "Debian"
```

### Using Templates in Playbook Execution

```yaml
---
- name: Playbook with templates
  hosts: all
  become: true
  vars:
    app_name: myapp
    app_port: 8080
  tasks:
    - name: Deploy configuration template
      template:
        src: templates/config.j2
        dest: /etc/{{ app_name }}/config.ini
        owner: root
        group: root
        mode: "0644"
        validate: "grep -q ^\[main\] %s"

    - name: Deploy systemd service template
      template:
        src: templates/systemd/service.j2
        dest: /etc/systemd/system/{{ app_name }}.service
        owner: root
        group: root
        mode: "0644"
        validate: "systemd-analyze verify %s"
      notify: daemon reload
```

## Verification Procedures

### Verify Playbook Syntax

```bash
# Syntax check
ansible-playbook playbook.yml --syntax-check

# Validate YAML
ansible-playbook playbook.yml --check

# Check with diff
ansible-playbook playbook.yml --check --diff
```

### Verify Playbook Execution

```bash
# Basic execution
ansible-playbook playbook.yml

# Verbose execution
ansible-playbook playbook.yml -vvv

# Dry run
ansible-playbook playbook.yml --check

# Dry run with diff
ansible-playbook playbook.yml --check --diff

# Limit to hosts
ansible-playbook playbook.yml --limit servera
```

### Verify Results

```bash
# Check if playbook ran successfully
ansible-playbook playbook.yml 2>&1 | grep -i "ok\|failed"

# Check specific task results
ansible-playbook playbook.yml -vvv 2>&1 | grep -A 5 "TASK"

# Verify changes
ansible-playbook playbook.yml --check --diff

# Check managed nodes status
ansible all -m ping
```

### Verify ansible-navigator

```bash
# Check ansible-navigator version
ansible-navigator --version

# Verify configuration
ansible-navigator configuration

# Run playbook
ansible-navigator run playbook.yml

# Execute ad hoc command
ansible-navigator execute all -m ping

# View documentation
ansible-navigator documentation
```

## Troubleshooting

### Playbook Syntax Errors

**Problem: YAML syntax error**

```bash
# Solution: Check YAML syntax
ansible-playbook playbook.yml --syntax-check

# Solution: Use text editor for validation
# VS Code, Vim, or Python: python3 -c "import yaml; yaml.safe_load(open('playbook.yml'))"

# Common errors:
# - Missing indentation
# - Missing colons
# - Missing quotes
# - Trailing commas
```

### Playbook Execution Errors

**Problem: Task fails during execution**

```bash
# Solution: Run with verbose output
ansible-playbook playbook.yml -vvv

# Solution: Check task details
ansible-playbook playbook.yml -vvv 2>&1 | grep -A 10 "TASK"

# Solution: Run specific task
ansible-playbook playbook.yml --start-at-task "Task name"
```

**Common errors:**
- SSH connection failed
- Privilege escalation failed
- Module not found
- Variable not defined
- File not found

### ansible-navigator Issues

**Problem: ansible-navigator fails to start**

```bash
# Solution: Check installation
ansible-navigator --version

# Solution: Check configuration
ansible-navigator configuration

# Solution: Run with stdout mode
ansible-navigator run playbook.yml --mode stdout
```

**Problem: Documentation not found**

```bash
# Solution: Check collections
ansible-galaxy collection list

# Solution: Install required collections
ansible-galaxy collection install collection_name

# Solution: Search for modules
ansible-doc -l | grep keyword
```

### Git Integration Issues

**Problem: Git authentication fails**

```bash
# Solution: Configure Git credentials
git config --global credential.helper store

# Solution: Use SSH keys
git config --global url."ssh://git@github.com/".insteadOf "https://github.com/"

# Solution: Check SSH key
ssh -T git@github.com
```

## Real-World Automation Examples

### Complete Playbook Execution Workflow

```yaml
---
- name: Complete automation playbook
  hosts: all
  become: true
  gather_facts: yes
  vars:
    app_name: myapp
    app_port: 8080
    log_level: info
  tasks:
    # Pre-tasks
    - name: Validate required variables
      assert:
        that:
          - app_name is defined
          - app_port is defined
        fail_msg: "Required variables app_name and app_port are not defined"
        success_msg: "Required variables are defined"
      tags: always

    # Package installation
    - name: Update package cache
      dnf:
        name: "*"
        state: present
        update_cache: yes
      tags: packages

    - name: Install required packages
      dnf:
        name:
          - httpd
          - mod_ssl
          - firewalld
        state: present
      tags: packages

    # Service configuration
    - name: Ensure firewalld is running
      systemd:
        name: firewalld
        state: started
        enabled: yes
      notify: reload firewalld
      tags: services

    - name: Ensure httpd is running
      systemd:
        name: httpd
        state: started
        enabled: yes
      notify: restart httpd
      tags: services

    # Firewall configuration
    - name: Open HTTP port
      firewalld:
        port: "80/tcp"
        permanent: yes
        immediate: yes
        state: enabled
      notify: reload firewalld
      tags: firewall

    - name: Open HTTPS port
      firewalld:
        port: "443/tcp"
        permanent: yes
        immediate: yes
        state: enabled
      notify: reload firewalld
      tags: firewall

    # File deployment
    - name: Create application directory
      file:
        path: /etc/{{ app_name }}
        state: directory
        owner: root
        group: root
        mode: "0755"
      tags: files

    - name: Deploy configuration
      template:
        src: templates/config.j2
        dest: /etc/{{ app_name }}/config.ini
        owner: root
        group: root
        mode: "0644"
        validate: "grep -q ^\[main\] %s"
      tags: files
      notify: restart httpd

    # Verification
    - name: Verify httpd is running
      systemd:
        name: httpd
        state: started
      tags: verify

    - name: Verify httpd is listening
      uri:
        url: http://localhost:80
        status_code: 200
      register: httpd_check
      tags: verify

    - name: Display verification results
      debug:
        msg: "HTTPD status: {{ httpd_check.status | default('unknown') }}"
      tags: always

  handlers:
    - name: reload firewalld
      systemd:
        name: firewalld
        state: reloaded

    - name: restart httpd
      systemd:
        name: httpd
        state: restarted
```

```bash
# Execute playbook
ansible-playbook automation_playbook.yml -vvv

# Execute with ansible-navigator
ansible-navigator run automation_playbook.yml

# Execute with specific tags
ansible-playbook automation_playbook.yml --tags packages,firewall

# Execute dry run
ansible-playbook automation_playbook.yml --check --diff
```

## RHCE Exam Notes

### Critical Exam Points

1. **ansible-playbook is the primary CLI tool**: Know all execution options and flags.

2. **ansible-navigator is the primary interface**: The exam expects you to use ansible-navigator for playbook execution and documentation lookup.

3. **Tags control task execution**: Use `--tags` and `--skip-tags` for selective execution.

4. **Syntax check before execution**: Always run `--syntax-check` before executing playbooks.

5. **Dry run with diff**: Use `--check --diff` to preview changes before applying.

6. **Git version control**: Know basic Git operations for playbook management.

7. **VS Code integration**: Configure ansible-navigator in VS Code for development workflow.

8. **Development containers**: Use Ansible development containers for isolated execution.

### Exam Workflow

1. Create or verify `ansible.cfg`
2. Create or verify inventory
3. Test connectivity with `ansible all -m ping`
4. Create playbook
5. Run syntax check: `ansible-playbook playbook.yml --syntax-check`
6. Run dry run: `ansible-playbook playbook.yml --check --diff`
7. Execute with ansible-navigator: `ansible-navigator run playbook.yml`
8. Verify results
9. Commit to Git

### Time Management

- 5 minutes: Initial configuration
- 10 minutes: Playbook creation
- 5 minutes: Syntax check and dry run
- 15 minutes: Execution and verification
- 5 minutes: Git commit

## Common Mistakes

### Mistake 1: Not Using `--syntax-check`

```bash
# WRONG - may have syntax errors
ansible-playbook playbook.yml

# CORRECT - check syntax first
ansible-playbook playbook.yml --syntax-check
```

### Mistake 2: Not Using `--check` for Preview

```bash
# WRONG - applies changes without preview
ansible-playbook playbook.yml

# CORRECT - preview changes first
ansible-playbook playbook.yml --check --diff
```

### Mistake 3: Forgetting Tags

```yaml
# WRONG - all tasks run every time
- name: Install packages
  dnf:
    name: httpd
    state: present

# CORRECT - use tags for selective execution
- name: Install packages
  dnf:
    name: httpd
    state: present
  tags: packages
```

### Mistake 4: Not Using ansible-navigator

```bash
# WRONG - using only CLI
ansible-playbook playbook.yml

# CORRECT - use ansible-navigator for better UX
ansible-navigator run playbook.yml
```

## Chapter Summary

This chapter covered playbook execution:

- **ansible-playbook** is the CLI tool for executing playbooks with various options and flags.
- **ansible-navigator** provides an interactive TUI for playbook execution, documentation browsing, and inventory management.
- **Tags** control selective task execution with `--tags` and `--skip-tags` parameters.
- **Syntax check** with `--syntax-check` validates playbooks before execution.
- **Dry run** with `--check --diff` previews changes without applying them.
- **Git operations** enable version control for playbooks with clone, add, commit, and push commands.
- **VS Code integration** configures ansible-navigator for development workflow.
- **Development containers** provide isolated Ansible execution environments.

## Quick Reference

| Command | Purpose |
|---|---|
| `ansible-playbook playbook.yml` | Execute playbook |
| `ansible-playbook --syntax-check` | Validate syntax |
| `ansible-playbook --check` | Dry run |
| `ansible-playbook --check --diff` | Dry run with diff |
| `ansible-playbook -vvv` | Verbose output |
| `ansible-playbook --limit host` | Limit to hosts |
| `ansible-playbook --tags tag` | Run tagged tasks |
| `ansible-playbook --skip-tags tag` | Skip tagged tasks |
| `ansible-navigator run playbook.yml` | Execute via TUI |
| `ansible-navigator documentation` | Browse docs |
| `ansible-navigator execute` | Run ad hoc |
| `git clone repo` | Clone repository |
| `git add .` | Add files |
| `git commit -m "msg"` | Commit changes |
| `git push origin main` | Push to remote |

## Review Questions

1. What is the difference between `ansible-playbook` and `ansible-navigator`?

2. What does the `--check` flag do when running a playbook?

3. How do you use tags to selectively execute tasks in a playbook?

4. What is the purpose of the `--syntax-check` flag?

5. How would you run a playbook with verbose output showing all details?

6. What is the difference between `--tags` and `--skip-tags`?

7. How do you use ansible-navigator to browse module documentation?

8. What is the purpose of the `--limit` parameter in playbook execution?

9. How do you commit changes to a Git repository?

10. What is an Ansible development container and how is it used?

## Answers

1. `ansible-playbook` is the command-line tool for executing playbooks. It runs playbooks in text mode with command-line output. `ansible-navigator` is a text-based user interface (TUI) that provides an interactive experience for playbook execution, documentation browsing, inventory management, and configuration editing. ansible-navigator wraps ansible-playbook and adds additional features.

2. The `--check` flag (also known as dry-run mode) executes the playbook without making any actual changes to the managed nodes. It shows what changes would be made if the playbook were run for real. This is useful for testing playbooks before applying changes to production systems.

3. Tags are keywords assigned to tasks in a playbook. To selectively execute tasks, use `--tags tag_name` to run only tasks with that tag, or `--skip-tags tag_name` to skip tasks with that tag. You can specify multiple tags: `--tags "packages,firewall"`. Tasks without tags run by default unless skipped.

4. The `--syntax-check` flag validates the playbook YAML syntax without executing it. It checks for YAML errors, missing parameters, and structural issues. Running this before execution helps catch errors early without wasting time on failed executions.

5. Use `ansible-playbook playbook.yml -vvv` for maximum verbosity. The `-v` flag increases verbosity level: `-v` shows task names, `-vv` shows task details, `-vvv` shows module parameters and additional debugging information.

6. `--tags tag_name` runs only tasks that have the specified tag. `--skip-tags tag_name` skips tasks that have the specified tag. For example, `--tags packages` runs only package installation tasks, while `--skip-tags reboot` skips all tasks with the reboot tag. You can combine both: `--tags packages --skip-tags verify`.

7. Use `ansible-navigator` to open the interactive documentation browser. Type `/` to enter search mode, then enter the module name (e.g., `firewalld`). Press Enter to search and browse the module documentation, including parameters, examples, and return values.

8. The `--limit` parameter restricts playbook execution to specific hosts or groups. For example, `--limit servera` runs only on servera, while `--limit webservers` runs on all hosts in the webservers group. This is useful for testing on specific hosts or updating individual servers.

9. To commit changes to a Git repository: (1) `git add .` to stage all changes, (2) `git commit -m "descriptive message"` to commit with a message, (3) optionally `git push origin main` to push to the remote repository. Always write clear commit messages describing the changes.

10. An Ansible development container is a Docker container pre-configured with Ansible and its dependencies. It provides an isolated environment for playbook execution, ensuring consistent behavior across different systems. In VS Code, you can use the Dev Containers extension to run playbooks inside a container, avoiding dependency conflicts on your host system.


---

# Chapter 08 — Reviewed Edition Closing Checkpoint

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
