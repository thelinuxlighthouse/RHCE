# Chapter 11: Create Ansible Plays and Playbooks


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


## Added Review: Playbooks Are State Documents

A playbook is not a shell script written in YAML. A shell script says:

```text
Run this command, then this command, then this command.
```

A good playbook says:

```text
Make this host group end in this state.
```

That is why modules matter. Modules understand state and can report whether they changed anything.

### Good task anatomy

```yaml
- name: Ensure httpd service is enabled and running
  ansible.builtin.systemd_service:
    name: httpd
    enabled: true
    state: started
```

Read it naturally:

```text
Task name: human goal.
Module: systemd service manager.
name: service unit.
enabled: persist after reboot.
state: current runtime state.
```


## Learning Objectives

By the end of this chapter, you will be able to:

- Create Ansible plays and playbooks from scratch
- Use commonly used Ansible modules for system configuration
- Use variables to retrieve and store task results
- Use conditionals to control play execution based on facts and variables
- Configure error handling with ignore_errors, failed_when, and changed_when
- Create playbooks that configure systems to a specified state (idempotent)
- Structure playbooks for complex automation scenarios
- Use Ansible facts to make dynamic decisions
- Organize playbooks with pre_tasks, post_tasks, and handlers
- Validate playbook syntax before execution

## Concept Overview

Ansible playbooks are YAML files that define automation tasks. A playbook contains one or more plays, where each play targets a group of hosts and executes a sequence of tasks. Plays organize automation into logical units that can be executed independently or as part of a larger workflow.

### Why Playbook Creation Matters

Without well-structured playbooks:

- Automation is not repeatable
- Configuration drift occurs
- Manual intervention is required
- No audit trail of changes
- Difficult to maintain and scale

### How Playbooks Configure Systems

```
Playbook Execution Flow:
─────────────────────────────────────────────────────────────────
1. Play Definition
   │
   ├── Hosts to target
   │
   ├── Become (privilege escalation)
   │
   └── Gather facts

2. Pre-tasks
   │
   └── Tasks before main tasks

3. Main Tasks
   │
   ├── With loops
   │
   ├── With conditionals
   │
   ├── With error handling
   │
   └── Register results

4. Post-tasks
   │
   └── Tasks after main tasks

5. Handlers
   │
   └── Run on play completion if notified

6. Converged State
   │
   └── System is in desired state
```

## Ansible Fundamentals Required

### Ansible Module Categories

| Category | Common Modules | Purpose |
|---|---|---|
| Packages | `ansible.builtin.dnf` / verified `ansible.builtin.dnf5`, and only environment-verified module-stream methods | Install/remove packages |
| Services | `systemd` | Start/stop/enable services |
| Files | `copy`, `template`, `file` | Deploy files |
| Users/Groups | `user`, `group` | Manage users/groups |
| Firewall | `firewalld` | Configure firewall rules |
| Storage | `mount`, `lvg`, `lvol` | Manage storage |
| Security | `selinux`, `seboolean` | Configure SELinux |
| Cron | `cron` | Schedule tasks |
| Archive | `archive`, `unarchive` | Create/extract archives |
| Network | `ping`, `uri`, `url` | Network operations |

### Play Structure

```yaml
---
- name: Play description
  hosts: target_group
  become: true
  gather_facts: yes
  vars:
    variable_name: value
  pre_tasks:
    - name: Pre-task
      module:
        parameter: value
  tasks:
    - name: Task
      module:
        parameter: value
      register: result_var
      when: condition
      failed_when: failure_condition
      changed_when: changed_condition
      ignore_errors: true
  post_tasks:
    - name: Post-task
      module:
        parameter: value
  handlers:
    - name: Handler
      module:
        parameter: value
```

## Commands and Tools

```bash
# Create playbook
vim playbook.yml
nano playbook.yml
code playbook.yml

# Validate syntax
ansible-playbook playbook.yml --syntax-check

# Run playbook
ansible-playbook playbook.yml
ansible-playbook -i inventory playbook.yml

# Run with verbose output
ansible-playbook playbook.yml -vvv

# Run dry run
ansible-playbook playbook.yml --check

# Run dry run with diff
ansible-playbook playbook.yml --check --diff

# List available modules
ansible-doc -l

# Get module documentation
ansible-doc dnf
ansible-doc systemd
ansible-doc firewalld

# Search module parameters
ansible-doc -l | grep -i package

# View module examples
ansible-doc -s dnf
```

## Playbook Examples

### Example 1: Basic Playbook with Common Modules

```yaml
---
- name: Basic system configuration playbook
  hosts: all
  become: true
  gather_facts: yes
  tasks:
    # Package management
    - name: Update package cache
      dnf:
        name: "*"
        state: present
        update_cache: yes

    - name: Install common packages
      dnf:
        name:
          - vim-enhanced
          - net-tools
          - wget
          - curl
          - tree
        state: present

    # Service management
    - name: Ensure firewalld is running and enabled
      systemd:
        name: firewalld
        state: started
        enabled: yes

    - name: Ensure SSH is running and enabled
      systemd:
        name: sshd
        state: started
        enabled: yes

    # Firewall configuration
    - name: Open HTTP port
      firewalld:
        port: "80/tcp"
        permanent: yes
        immediate: yes
        state: enabled

    - name: Open HTTPS port
      firewalld:
        port: "443/tcp"
        permanent: yes
        immediate: yes
        state: enabled

    # User management
    - name: Create application user
      user:
        name: myapp
        system: yes
        shell: /sbin/nologin
        createhome: no

    - name: Create application group
      group:
        name: myapp
        system: yes

    # File management
    - name: Create application directory
      file:
        path: /etc/myapp
        state: directory
        owner: root
        group: root
        mode: "0755"

    - name: Deploy configuration file
      copy:
        content: |
          [application]
          name = myapp
          port = 8080
        dest: /etc/myapp/config.ini
        owner: root
        group: root
        mode: "0644"

    # Verification
    - name: Verify firewalld status
      command: systemctl is-active firewalld
      register: firewalld_status
      changed_when: false

    - name: Display firewalld status
      debug:
        msg: "Firewalld status: {{ firewalld_status.stdout }}"
```

### Example 2: Playbook with Variables and Results

```yaml
---
- name: Playbook with variables and results
  hosts: all
  become: true
  gather_facts: yes
  vars:
    app_name: myapp
    app_port: 8080
    log_level: info
    max_connections: 100
  tasks:
    # Install packages
    - name: Install web server packages
      dnf:
        name:
          - httpd
          - mod_ssl
        state: present
      register: httpd_install
      changed_when: false

    # Display installation results
    - name: Display httpd installation status
      debug:
        msg: "HTTPD installation {{ 'successful' if httpd_install.rc == 0 else 'failed' }}"
      when: httpd_install is defined

    # Start service
    - name: Start httpd service
      systemd:
        name: httpd
        state: started
        enabled: yes
      register: httpd_start
      changed_when: false

    # Display service start result
    - name: Display httpd start status
      debug:
        msg: "HTTPD start {{ 'successful' if httpd_start.rc == 0 else 'failed' }}"
      when: httpd_start is defined

    # Check if service is running
    - name: Check httpd status
      command: systemctl is-active httpd
      register: httpd_check
      changed_when: false

    # Display service status
    - name: Display httpd status
      debug:
        msg: "HTTPD status: {{ httpd_check.stdout }}"

    # Store package list
    - name: Get installed packages
      command: dnf list installed httpd
      register: httpd_packages
      changed_when: false

    # Display installed httpd version
    - name: Display httpd version
      debug:
        msg: "HTTPD packages: {{ httpd_packages.stdout_lines }}"
      when: httpd_packages is defined

    # Store service status
    - name: Get service status
      command: systemctl status httpd --no-pager
      register: httpd_service_status
      changed_when: false

    # Display service status (first 10 lines)
    - name: Display service status
      debug:
        msg: "{{ httpd_service_status.stdout_lines[:10] }}"
      when: httpd_service_status is defined
```

### Example 3: Playbook with Conditionals

```yaml
---
- name: Playbook with conditionals
  hosts: all
  become: true
  gather_facts: yes
  vars:
    enable_firewall: true
    enable_httpd: true
    http_port: 8080
    environment: production
  tasks:
    # Conditional package installation
    - name: Install httpd if enabled
      dnf:
        name: httpd
        state: present
      when: enable_httpd

    # Conditional service start
    - name: Start httpd if package installed
      systemd:
        name: httpd
        state: started
        enabled: yes
      when:
        - enable_httpd
        - ansible_facts.packages.httpd is defined
        - ansible_facts.packages.httpd.state == "present"

    # Conditional firewall configuration
    - name: Ensure firewalld is running
      systemd:
        name: firewalld
        state: started
        enabled: yes
      when: enable_firewall

    - name: Open HTTP port if firewall enabled
      firewalld:
        port: "{{ http_port }}/tcp"
        permanent: yes
        immediate: yes
        state: enabled
      when:
        - enable_firewall
        - ansible_facts.services.firewalld.service is defined

    # OS-specific configuration
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

    # Environment-specific configuration
    - name: Set log level for production
      lineinfile:
        path: /etc/myapp/config.ini
        regexp: "^log_level = "
        line: "log_level = {{ log_level | default('info') }}"
        state: present
      when: environment == "production"

    - name: Set debug mode for development
      lineinfile:
        path: /etc/myapp/config.ini
        regexp: "^debug = "
        line: "debug = true"
        state: present
      when: environment == "development"

    # Conditional based on fact
    - name: Install package if not present
      dnf:
        name: httpd
        state: present
      when:
        - ansible_facts.packages.httpd is defined
        - ansible_facts.packages.httpd.state != "present"

    - name: Skip if package already installed
      debug:
        msg: "HTTPD is already installed, skipping installation"
      when:
        - ansible_facts.packages.httpd is defined
        - ansible_facts.packages.httpd.state == "present"
```

### Example 4: Playbook with Error Handling

```yaml
---
- name: Playbook with error handling
  hosts: all
  become: true
  gather_facts: yes
  tasks:
    # Task with ignore_errors
    - name: Attempt to install optional package
      dnf:
        name: optional-package
        state: present
      ignore_errors: true
      register: optional_install
      changed_when: false

    # Check if installation succeeded
    - name: Check optional package installation
      debug:
        msg: "Optional package {{ 'installed' if optional_install.rc == 0 else 'not found' }}"
      when: optional_install is defined

    # Task with custom failed_when
    - name: Check for optional service
      command: systemctl is-active optional-service
      register: optional_service
      changed_when: false
      failed_when: false

    # Display service status
    - name: Display optional service status
      debug:
        msg: "Optional service status: {{ optional_service.stdout }}"
      when: optional_service is defined

    # Block with rescue and always
    - name: Configure with error handling
      block:
        - name: Install critical package
          dnf:
            name: vim-enhanced
            state: present

        - name: Deploy configuration
          copy:
            content: "critical config"
            dest: /etc/critical/config
          notify: restart critical-service

        - name: Start service
          systemd:
            name: critical-service
            state: started
            enabled: yes
      rescue:
        - name: Rollback on failure
          debug:
            msg: "Configuration failed, rolling back"

        - name: Remove critical config
          file:
            path: /etc/critical/config
            state: absent
          ignore_errors: true

        - name: Stop critical service
          systemd:
            name: critical-service
            state: stopped
          ignore_errors: true
      always:
        - name: Log completion
          debug:
            msg: "Configuration attempt completed"

    # Task with failed_when condition
    - name: Validate configuration
      command: httpd -t
      register: httpd_test
      changed_when: false
      failed_when: httpd_test.rc != 0

    # Display validation result
    - name: Display httpd validation
      debug:
        msg: "HTTPD configuration {{ 'valid' if httpd_test.rc == 0 else 'invalid' }}"
      when: httpd_test is defined
```

### Example 5: Playbook Configuring Systems to Specified State

```yaml
---
- name: Configure system to specified state
  hosts: all
  become: true
  gather_facts: yes
  vars:
    app_name: myapp
    app_port: 8080
    app_user: myapp
    app_group: myapp
    log_max_size: 100M
    log_rotate_count: 7
  tasks:
    # Package state: installed
    - name: Ensure httpd is installed
      dnf:
        name: httpd
        state: present
      register: httpd_installed
      changed_when: false

    # Service state: running and enabled
    - name: Ensure httpd is running
      systemd:
        name: httpd
        state: started
        enabled: yes
      register: httpd_running
      changed_when: false

    # User state: exists with specific attributes
    - name: Ensure myapp user exists
      user:
        name: "{{ app_user }}"
        system: yes
        shell: /sbin/nologin
        home: /var/lib/{{ app_user }}
        createhome: no
      register: myapp_user
      changed_when: false

    # Group state: exists
    - name: Ensure myapp group exists
      group:
        name: "{{ app_group }}"
        system: yes
      register: myapp_group
      changed_when: false

    # Directory state: exists with permissions
    - name: Ensure application directory exists
      file:
        path: /etc/{{ app_name }}
        state: directory
        owner: root
        group: root
        mode: "0755"
      register: app_dir
      changed_when: false

    # File state: exists with content and permissions
    - name: Ensure configuration file exists
      copy:
        content: |
          [application]
          name = {{ app_name }}
          port = {{ app_port }}
          log_max_size = {{ log_max_size }}
          log_rotate_count = {{ log_rotate_count }}
        dest: /etc/{{ app_name }}/config.ini
        owner: root
        group: root
        mode: "0644"
      register: app_config
      changed_when: false

    # Firewall state: port opened
    - name: Ensure HTTP port is open
      firewalld:
        port: "{{ app_port }}/tcp"
        permanent: yes
        immediate: yes
        state: enabled
      register: http_port_open
      when: ansible_facts.services.firewalld.service is defined
      changed_when: false

    # SELinux state: boolean set
    - name: Ensure SELinux boolean is set
      seboolean:
        name: httpd_can_network_connect
        state: yes
        persistent: yes
      when: ansible_selinux.status == "enabled"
      register: selinux_boolean
      changed_when: false

    # Verification tasks
    - name: Verify httpd is installed
      debug:
        msg: "HTTPD installed: {{ httpd_installed.rc == 0 }}"
      when: httpd_installed is defined

    - name: Verify httpd is running
      debug:
        msg: "HTTPD running: {{ httpd_running.rc == 0 }}"
      when: httpd_running is defined

    - name: Verify myapp user exists
      debug:
        msg: "User exists: {{ myapp_user is defined }}"
      when: myapp_user is defined

    - name: Verify configuration file exists
      stat:
        path: /etc/{{ app_name }}/config.ini
      register: config_file
      changed_when: false

    - name: Display configuration status
      debug:
        msg: "Configuration file: {{ 'exists' if config_file.stat.exists else 'missing' }}"
```

### Example 6: Complete Production Playbook

```yaml
---
- name: Complete production system configuration
  hosts: all
  become: true
  gather_facts: yes
  vars:
    app_name: myapp
    app_port: 8080
    app_user: myapp
    app_group: myapp
    environment: production
    log_level: info
    max_connections: 100
  pre_tasks:
    # Validate prerequisites
    - name: Validate required variables
      assert:
        that:
          - app_name is defined
          - app_port is defined
          - app_user is defined
        fail_msg: "Required variables are not defined"
        success_msg: "All required variables are defined"
      tags: always

    # Update packages
    - name: Update all packages to latest
      dnf:
        name: "*"
        state: present
        update_cache: yes
      tags: packages

  tasks:
    # Package installation
    - name: Install web server packages
      dnf:
        name:
          - httpd
          - mod_ssl
          - firewalld
          - crontabs
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

    - name: Ensure SSH is running
      systemd:
        name: sshd
        state: started
        enabled: yes
      notify: restart sshd
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

    - name: Open custom application port
      firewalld:
        port: "{{ app_port }}/tcp"
        permanent: yes
        immediate: yes
        state: enabled
      notify: reload firewalld
      tags: firewall

    # User and group management
    - name: Create application group
      group:
        name: "{{ app_group }}"
        system: yes
      tags: users

    - name: Create application user
      user:
        name: "{{ app_user }}"
        system: yes
        shell: /sbin/nologin
        home: /var/lib/{{ app_user }}
        createhome: no
      tags: users

    # File system configuration
    - name: Create application configuration directory
      file:
        path: /etc/{{ app_name }}
        state: directory
        owner: root
        group: root
        mode: "0755"
      tags: files

    - name: Create web content directory
      file:
        path: /var/www/{{ app_name }}
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0755"
      tags: files

    - name: Create log directory
      file:
        path: /var/log/{{ app_name }}
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0755"
      tags: files

    # Configuration deployment
    - name: Deploy main configuration
      copy:
        content: |
          [application]
          name = {{ app_name }}
          port = {{ app_port }}
          log_level = {{ log_level }}
          max_connections = {{ max_connections }}
          environment = {{ environment }}
        dest: /etc/{{ app_name }}/config.ini
        owner: root
        group: root
        mode: "0644"
      tags: files

    - name: Deploy httpd configuration
      copy:
        content: |
          Listen {{ app_port }}
          ServerName {{ inventory_hostname }}
          DocumentRoot /var/www/{{ app_name }}
        dest: /etc/httpd/conf.d/{{ app_name }}.conf
        owner: root
        group: root
        mode: "0644"
      tags: files
      notify: restart httpd

    # SELinux configuration
    - name: Set SELinux file context
      sefcontext:
        target: "/var/www/{{ app_name }}(/.*)?"
        setype: httpd_sys_content_t
        state: present
      when: ansible_selinux.status == "enabled"
      ignore_errors: true
      tags: security

    - name: Apply SELinux contexts
      command: restorecon -Rv /var/www/{{ app_name }}
      changed_when: false
      when: ansible_selinux.status == "enabled"
      ignore_errors: true
      tags: security

    # Cron job configuration
    - name: Create backup cron job
      cron:
        name: "Daily {{ app_name }} backup"
        minute: "0"
        hour: "2"
        job: "/usr/local/bin/backup-{{ app_name }}.sh >> /var/log/backup-{{ app_name }}.log 2>&1"
        user: root
      tags: cron

    # Verification
    - name: Verify httpd is running
      systemd:
        name: httpd
        state: started
      tags: verify

    - name: Verify httpd is listening
      uri:
        url: http://localhost:{{ app_port }}
        status_code: 200
      register: httpd_check
      tags: verify

    - name: Display verification results
      debug:
        msg: "HTTPD status: {{ httpd_check.status | default('unknown') }}"
      tags: always

  handlers:
    - name: restart httpd
      systemd:
        name: httpd
        state: restarted

    - name: reload firewalld
      systemd:
        name: firewalld
        state: reloaded

    - name: restart sshd
      systemd:
        name: sshd
        state: restarted
```

## Explanation of Each Playbook

### Playbook 1: Basic System Configuration

This playbook demonstrates fundamental Ansible modules across different categories: packages (dnf), services (systemd), firewall (firewalld), users (user, group), and files (copy). The `register` variable captures task output for later reference. The `changed_when: false` parameter prevents tasks from reporting changes when they only check status.

### Playbook 2: Variables and Results

This playbook shows how to use the `vars` section to define variables at the play level. The `register` keyword captures task output into variables like `httpd_install`, `httpd_start`, and `httpd_check`. The `when` clause uses these registered variables to conditionally display results. The `stdout_lines` attribute extracts individual lines from command output.

### Playbook 3: Conditionals

This playbook demonstrates multiple conditional patterns:
- Simple `when: variable` for boolean conditions
- Multiple conditions with `when:` list
- OS-specific tasks using `ansible_os_family` fact
- Environment-specific tasks using variables
- Fact-based conditionals checking if packages/services exist
- `when` with `default()` filter for optional variables

### Playbook 4: Error Handling

This playbook demonstrates error handling strategies:
- `ignore_errors: true` continues execution even if task fails
- Custom `failed_when` condition for specific failure criteria
- `block/rescue/always` for structured error handling
- `changed_when: false` for read-only verification tasks
- `register` with conditional display based on task success

### Playbook 5: Specified State

This playbook demonstrates idempotent configuration by ensuring systems reach a specific state:
- `state: present` ensures resources exist
- `state: started` and `state: enabled` for services
- `state: directory` with permissions
- `state: mounted` for filesystems
- Verification tasks confirm the desired state

### Playbook 6: Complete Production Playbook

This comprehensive playbook demonstrates all playbook elements:
- `pre_tasks` for validation and setup
- Multiple module categories working together
- `notify` handlers for service restarts
- `tags` for selective execution
- `handlers` for service management
- `assert` for validation
- Verification tasks for confirmation

## Variables and Templates

### Using Variables for Task Results

```yaml
---
- name: Store task results in variables
  hosts: all
  become: true
  tasks:
    # Register task output
    - name: Install package
      dnf:
        name: httpd
        state: present
      register: package_result

    # Register command output
    - name: Run command
      command: rpm -q httpd
      register: command_result
      changed_when: false

    # Access registered variables
    - name: Display package result
      debug:
        msg: "Return code: {{ package_result.rc }}"
        msg: "Output: {{ package_result.stdout }}"
        msg: "Stderr: {{ package_result.stderr }}"
        msg: "Changed: {{ package_result.changed }}"
        msg: "Failed: {{ package_result.failed }}"

    - name: Display command result
      debug:
        msg: "Command output: {{ command_result.stdout }}"
        msg: "Lines: {{ command_result.stdout_lines }}"
        msg: "RC: {{ command_result.rc }}"
```

### Using Jinja2 Templates in Playbooks

```yaml
---
- name: Playbook with template variables
  hosts: all
  become: true
  vars:
    app_name: myapp
    app_port: 8080
    log_level: info
  tasks:
    - name: Deploy configuration template
      template:
        src: templates/config.j2
        dest: /etc/{{ app_name }}/config.ini
        owner: root
        group: root
        mode: "0644"
        validate: "grep -q ^\[main\] %s"
```

```yaml
# Template: templates/config.j2
[application]
name = {{ app_name }}
port = {{ app_port }}
log_level = {{ log_level }}
environment = {{ environment | default('production') }}
```

## Verification Procedures

### Verify Playbook Syntax

```bash
# Syntax check
ansible-playbook playbook.yml --syntax-check

# Validate YAML
python3 -c "import yaml; yaml.safe_load(open('playbook.yml'))"
```

### Verify Playbook Execution

```bash
# Run with verbose output
ansible-playbook playbook.yml -vvv

# Dry run
ansible-playbook playbook.yml --check

# Dry run with diff
ansible-playbook playbook.yml --check --diff
```

### Verify Task Results

```bash
# Check specific task
ansible-playbook playbook.yml -vvv 2>&1 | grep -A 5 "TASK"

# View task output
ansible-playbook playbook.yml 2>&1 | grep "changed\|failed\|ok"
```

## Troubleshooting

### Playbook Syntax Errors

**Problem: YAML syntax error**

```bash
# Solution: Check syntax
ansible-playbook playbook.yml --syntax-check

# Common errors:
# - Missing indentation
# - Missing colons
# - Missing quotes
# - Trailing commas
```

### Module Execution Failures

**Problem: Module returns error**

```bash
# Solution: Run with verbose output
ansible-playbook playbook.yml -vvv

# Solution: Check module documentation
ansible-doc module_name

# Solution: Verify module parameters
ansible-doc -s module_name
```

### Conditional Not Working

**Problem: Task not executing as expected**

```bash
# Solution: Check condition
ansible-playbook playbook.yml -vvv 2>&1 | grep "when"

# Solution: Verify variable values
ansible all -m debug -a "msg={{ variable_name }}"

# Solution: Check fact gathering
ansible all -m setup -a "filter=ansible_os_family"
```

### Error Handling Not Working

**Problem: Block not catching errors**

```bash
# Solution: Verify block/rescue structure
# Ensure tasks are inside block:
block:
  - name: Task
    module:
rescue:
  - name: Recovery
    module:
```

## Real-World Automation Examples

### Web Server Provisioning Playbook

```yaml
---
- name: Provision web server
  hosts: webservers
  become: true
  gather_facts: yes
  vars:
    app_name: myapp
    app_port: 8080
  pre_tasks:
    - name: Validate inventory
      assert:
        that:
          - 'webservers' in groups
        fail_msg: "No webservers group found"

  tasks:
    # Installation
    - name: Install httpd
      dnf:
        name: httpd
        state: present
      tags: packages

    # Configuration
    - name: Deploy httpd config
      copy:
        content: |
          Listen {{ app_port }}
          ServerName {{ inventory_hostname }}
        dest: /etc/httpd/conf.d/{{ app_name }}.conf
      notify: restart httpd
      tags: config

    # Firewall
    - name: Open port
      firewalld:
        port: "{{ app_port }}/tcp"
        permanent: yes
        immediate: yes
        state: enabled
      notify: reload firewalld
      tags: firewall

  handlers:
    - name: restart httpd
      systemd:
        name: httpd
        state: restarted

    - name: reload firewalld
      systemd:
        name: firewalld
        state: reloaded
```

## RHCE Exam Notes

### Critical Exam Points

1. **Play structure is essential**: Know how to write plays with hosts, become, tasks, handlers.

2. **Common modules**: Master dnf, systemd, firewalld, copy, template, user, group, file.

3. **Variables and registration**: Use `vars` for play-level variables and `register` for task output.

4. **Conditionals**: Use `when` for task execution control.

5. **Error handling**: Know `ignore_errors`, `failed_when`, `changed_when`, and `block/rescue/always`.

6. **Handlers**: Use `notify` to trigger handlers after task changes.

7. **Idempotency**: Configure systems to a specified state, not actions.

8. **Tags**: Use `tags` for selective task execution.

### Exam Workflow

1. Write playbook structure
2. Add tasks with modules
3. Register important results
4. Add conditionals
5. Add error handling
6. Add handlers if needed
7. Validate with `--syntax-check`
8. Test with `--check`
9. Execute

### Time Management

- 10 minutes: Write playbook
- 5 minutes: Syntax check
- 10 minutes: Test execution
- 5 minutes: Verify results

## Common Mistakes

### Mistake 1: Not Using `register` for Important Results

```yaml
# WRONG - cannot check task result
- name: Install package
  dnf:
    name: httpd
    state: present

# CORRECT - capture result
- name: Install package
  dnf:
    name: httpd
    state: present
  register: httpd_install
```

### Mistake 2: Using `state: present` for Actions

```yaml
# WRONG - always runs, not idempotent
- name: Install package
  command: dnf install -y httpd

# CORRECT - idempotent state
- name: Install package
  dnf:
    name: httpd
    state: present
```

### Mistake 3: Forgetting `become: true`

```yaml
# WRONG - tasks run as non-root
- name: Install package
  dnf:
    name: httpd
    state: present

# CORRECT - escalate to root
- name: Configure servers
  hosts: all
  become: true
  tasks:
    - name: Install package
      dnf:
        name: httpd
        state: present
```

### Mistake 4: Not Using Handlers

```yaml
# WRONG - service doesn't reload config
- name: Deploy config
  copy:
    src: httpd.conf
    dest: /etc/httpd/conf/httpd.conf

# CORRECT - notify handler
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

## Chapter Summary

This chapter covered creating Ansible plays and playbooks:

- **Play structure**: Defines hosts, become, tasks, handlers, pre_tasks, post_tasks.
- **Common modules**: dnf, systemd, firewalld, copy, template, user, group, file, cron, selinux.
- **Variables**: Use `vars` for play-level variables and `register` for task results.
- **Conditionals**: Use `when` to control task execution based on variables and facts.
- **Error handling**: Use `ignore_errors`, `failed_when`, `changed_when`, and `block/rescue/always`.
- **Handlers**: Use `notify` to trigger handlers after task changes.
- **Idempotency**: Configure systems to a specified state, not perform actions.
- **Tags**: Use `tags` for selective task execution.
- **Verification**: Use `--syntax-check`, `--check`, and `--diff` for validation.

## Quick Reference

| Playbook Element | Purpose | Example |
|---|---|---|
| `hosts` | Target hosts/groups | `hosts: all` |
| `become` | Privilege escalation | `become: true` |
| `gather_facts` | Collect system facts | `gather_facts: yes` |
| `vars` | Play-level variables | `vars: { app: myapp }` |
| `tasks` | Task list | `tasks: [...]` |
| `handlers` | Notification handlers | `handlers: [...]` |
| `pre_tasks` | Pre-execution tasks | `pre_tasks: [...]` |
| `post_tasks` | Post-execution tasks | `post_tasks: [...]` |
| `register` | Store task output | `register: result` |
| `when` | Conditional execution | `when: condition` |
| `ignore_errors` | Continue on failure | `ignore_errors: true` |
| `failed_when` | Custom failure | `failed_when: result.rc > 1` |
| `changed_when` | Suppress change | `changed_when: false` |
| `notify` | Trigger handler | `notify: restart httpd` |
| `tags` | Selective execution | `tags: packages` |

## Review Questions

1. What is the difference between `vars` and `register` in a playbook?

2. How do you use the `when` parameter to conditionally execute a task?

3. What is the purpose of the `failed_when` parameter?

4. How do handlers work in Ansible playbooks?

5. What is the difference between `ignore_errors: true` and `failed_when: false`?

6. How do you configure a playbook to run tasks only on specific hosts?

7. What is the purpose of `pre_tasks` and `post_tasks`?

8. How do you make a playbook task idempotent?

9. What is the purpose of the `notify` directive?

10. How do you use tags to selectively execute tasks?

## Answers

1. `vars` defines variables at the play level that are available to all tasks in that play. `register` captures the output of a task into a variable for later use. `vars` is for configuration, `register` is for task results.

2. Use the `when` parameter to add a condition: `when: condition`. The task only executes if the condition evaluates to true. Conditions can be variables, facts, registered outputs, or boolean expressions like `when: enable_httpd` or `when: ansible_os_family == "RedHat"`.

3. The `failed_when` parameter customizes when a task is considered failed. By default, a task fails if the module returns a non-zero return code. With `failed_when: false`, the task never fails. With `failed_when: result.rc > 1`, the task fails only if the return code is greater than 1.

4. Handlers are tasks that run once at the end of a play if a task in the play notifies them. They are used for actions that should only happen when configuration changes, like restarting services. Use `notify: handler_name` in a task to trigger a handler.

5. `ignore_errors: true` continues playbook execution even if a task fails, treating the failure as non-fatal. `failed_when: false` customizes when a task is considered failed, allowing you to define failure beyond just a non-zero return code. `ignore_errors` always runs the task; `failed_when` controls success/failure determination.

6. Use the `hosts` parameter to specify which hosts or groups to target: `hosts: webservers` targets the webservers group, `hosts: all` targets all managed hosts. You can also use `--limit` when running the playbook.

7. `pre_tasks` are tasks that run before the main tasks in a play. `post_tasks` run after the main tasks. They are useful for setup (pre_tasks) or cleanup/verification (post_tasks).

8. Make a task idempotent by using modules that manage state (like `state: present`) rather than performing actions. The task should produce the same result regardless of how many times it runs. Use `register` to capture results and `when` to conditionally run tasks only when needed.

9. The `notify` directive tells Ansible to run a named handler after the task completes. Handlers only run if at least one task in the play notifies them. This is used for actions like restarting services that should only happen when configuration changes.

10. Use the `tags` parameter to assign tags to tasks. When running the playbook, use `--tags tag_name` to run only tasks with that tag, or `--skip-tags tag_name` to skip tasks with that tag. Tags allow selective execution without modifying the playbook.


---

# Chapter 11 — Reviewed Edition Closing Checkpoint

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
