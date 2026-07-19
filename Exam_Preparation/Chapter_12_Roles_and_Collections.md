# Chapter 12: Use Roles and Ansible Content Collections


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


## Added Review: Roles Are Folders with a Contract

A role is a reusable automation package. The most important files are:

```text
roles/webserver/
├── defaults/main.yml     # safe overridable defaults
├── vars/main.yml         # stronger internal variables, use carefully
├── tasks/main.yml        # main task list
├── handlers/main.yml     # restart/reload reactions
├── templates/           # Jinja2 templates
├── files/               # static files
├── meta/main.yml         # role metadata/dependencies
└── README.md            # how to use the role
```

Use `defaults/main.yml` for values users are expected to override. Use `vars/main.yml` sparingly because it has higher precedence and is harder to override cleanly.

A role should be understandable from its `README.md`, safe defaults, and clear variable names.


## Learning Objectives

By the end of this chapter, you will be able to

- Understand Ansible roles and their directory structure
- Create new Ansible roles from scratch
- Modify existing roles for specific requirements
- Use roles in playbooks to organize automation
- Install roles from Ansible Galaxy
- Install and manage Ansible Content Collections
- Use collection modules in playbooks
- Create custom collections
- Manage role dependencies
- Share and distribute roles and collections

## Concept Overview

Ansible roles and Content Collections are the standard ways to organize and share Ansible automation. Roles package related tasks, variables, files, and templates into reusable units. Content Collections are the modern, standardized way to distribute modules, plugins, and roles.

### Why Roles and Collections Matter

Without roles and collections:

- Playbooks become unmanageable
- No code reuse across projects
- Difficult to maintain automation
- No standardized distribution
- No dependency management
- No version control for automation

### How Roles and Collections Work

```
Role Structure:
─────────────────────────────────────────────────────────────────
roles/
└── webserver/
    ├── tasks/
    │   └── main.yml          # Main task list
    ├── handlers/
    │   └── main.yml          # Handlers
    ├── templates/
    │   └── httpd.conf.j2     # Jinja2 templates
    ├── files/
    │   └── index.html        # Static files
    ├── vars/
    │   └── main.yml          # Role variables
    ├── defaults/
    │   └── main.yml          # Default variables
    ├── meta/
    │   └── main.yml          # Role metadata
    └── tests/
        └── test.yml          # Role tests

Content Collections:
─────────────────────────────────────────────────────────────────
collections/
└── community.general/
    ├── plugins/
    │   ├── modules/
    │   │   └── sysctl.py     # Module files
    │   └── lookup/
    │       └── env.py        # Lookup plugins
    └── MANIFEST.json         # Collection manifest
```

## Ansible Fundamentals Required

### Role vs Collection

| Aspect | Role | Collection |
|---|---|---|
| Structure | Directory-based | Package-based |
| Distribution | Manual or Galaxy | Galaxy or local |
| Dependencies | `meta/main.yml` | `galaxy.yml` |
| Versioning | Git tags | Collection version |
| Modern standard | Legacy but still used | Current standard |

### Role Directory Structure

```
roles/
├── role_name/
│   ├── tasks/
│   │   └── main.yml
│   ├── handlers/
│   │   └── main.yml
│   ├── templates/
│   │   └── template.j2
│   ├── files/
│   │   └── static_file
│   ├── vars/
│   │   └── main.yml
│   ├── defaults/
│   │   └── main.yml
│   ├── meta/
│   │   └── main.yml
│   ├── README.md
│   └── tests/
│       └── test.yml
```

## Commands and Tools

```bash
# Create a role
ansible-galaxy init role_name

# Install a role from Galaxy
ansible-galaxy role install geerlingguy.httpd

# Install from requirements file
ansible-galaxy role install -r requirements.yml

# List installed roles
ansible-galaxy role list

# Search for roles
ansible-galaxy role search httpd

# Remove a role
ansible-galaxy role remove geerlingguy.httpd

# Install a collection
ansible-galaxy collection install community.general

# Install from requirements file
ansible-galaxy collection install -r requirements.yml

# List installed collections
ansible-galaxy collection list

# Search for collections
ansible-galaxy collection search community

# Remove a collection
ansible-galaxy collection remove community.general

# Validate collection
ansible-galaxy collection validate

# Check collection dependencies
ansible-galaxy collection list

# View collection documentation
ansible-doc -l | grep module_name

# View module examples
ansible-doc -s module_name
```

## Playbook Examples

### Example 1: Create an Ansible Role

```yaml
---
- name: Create Ansible role structure
  hosts: localhost
  connection: local
  become: true
  tasks:
    - name: Create roles directory
      file:
        path: /home/ansible/roles
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create role directory
      file:
        path: "/home/ansible/roles/{{ role_name }}"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create tasks directory
      file:
        path: "/home/ansible/roles/{{ role_name }}/tasks"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create handlers directory
      file:
        path: "/home/ansible/roles/{{ role_name }}/handlers"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create templates directory
      file:
        path: "/home/ansible/roles/{{ role_name }}/templates"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create files directory
      file:
        path: "/home/ansible/roles/{{ role_name }}/files"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create vars directory
      file:
        path: "/home/ansible/roles/{{ role_name }}/vars"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create defaults directory
      file:
        path: "/home/ansible/roles/{{ role_name }}/defaults"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create meta directory
      file:
        path: "/home/ansible/roles/{{ role_name }}/meta"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create tests directory
      file:
        path: "/home/ansible/roles/{{ role_name }}/tests"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create tasks/main.yml
      copy:
        content: |
          ---
          - name: Install httpd packages
            dnf:
              name:
                - httpd
                - mod_ssl
              state: present
          - name: Create document root
            file:
              path: /var/www/html
              state: directory
              owner: apache
              group: apache
              mode: "0755"
          - name: Deploy default page
            copy:
              content: |
                <html>
                <head><title>Apache</title></head>
                <body><h1>Welcome to Apache</h1></body>
                </html>
              dest: /var/www/html/index.html
              owner: apache
              group: apache
              mode: "0644"
          - name: Start httpd
            systemd:
              name: httpd
              state: started
              enabled: yes
          - name: Open HTTP port
            firewalld:
              port: "80/tcp"
              permanent: yes
              immediate: yes
              state: enabled
        dest: "/home/ansible/roles/{{ role_name }}/tasks/main.yml"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create handlers/main.yml
      copy:
        content: |
          ---
          - name: restart httpd
            systemd:
              name: httpd
              state: restarted
        dest: "/home/ansible/roles/{{ role_name }}/handlers/main.yml"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create templates/httpd.conf.j2
      copy:
        content: |
          Listen {{ httpd_port | default(80) }}
          ServerName {{ inventory_hostname }}
          DocumentRoot /var/www/html
        dest: "/home/ansible/roles/{{ role_name }}/templates/httpd.conf.j2"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create defaults/main.yml
      copy:
        content: |
          ---
          httpd_port: 80
          app_name: myapp
          max_clients: 256
        dest: "/home/ansible/roles/{{ role_name }}/defaults/main.yml"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create vars/main.yml
      copy:
        content: |
          ---
          httpd_packages:
            - httpd
            - mod_ssl
        dest: "/home/ansible/roles/{{ role_name }}/vars/main.yml"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create meta/main.yml
      copy:
        content: |
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

          file_attributes:
            path: /var/www/html
            owner: apache
            group: apache
            mode: "0755"
        dest: "/home/ansible/roles/{{ role_name }}/meta/main.yml"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create README.md
      copy:
        content: |
          # {{ role_name }} Role

          Ansible role for {{ role_name }} configuration.

          ## Requirements

          - Ansible 2.14+
          - RHEL 9

          ## Role Variables

          Available variables are listed in vars/main.yml and defaults/main.yml.

          ## Dependencies

          None

          ## Example Playbook

          ```yaml
          - hosts: all
            become: true
            roles:
              - {{ role_name }}
          ```
        dest: "/home/ansible/roles/{{ role_name }}/README.md"
        owner: ansible
        group: ansible
        mode: "0644"
```

### Example 2: Use Role in Playbook

```yaml
---
- name: Use Ansible role in playbook
  hosts: all
  become: true
  gather_facts: yes
  vars:
    role_name: webserver
  tasks:
    - name: Include role in play
      debug:
        msg: "Using role: {{ role_name }}"

    - name: Run role with defaults
      ansible.builtin.include_role:
        name: "{{ role_name }}"

    - name: Run role with parameters
      ansible.builtin.include_role:
        name: "{{ role_name }}"
        tasks_from: tasks
        vars_from: vars
      vars:
        httpd_port: 8080
        app_name: myapp

    - name: Run role with tags
      ansible.builtin.include_role:
        name: "{{ role_name }}"
      tags: webserver

    - name: Run role conditionally
      ansible.builtin.include_role:
        name: "{{ role_name }}"
      when:
        - ansible_os_family == "RedHat"
        - enable_webserver | default(true)

    - name: Display role structure
      command: ls -la /home/ansible/roles/{{ role_name }}/
      register: role_structure
      changed_when: false

    - name: Display role tasks
      command: cat /home/ansible/roles/{{ role_name }}/tasks/main.yml
      register: role_tasks
      changed_when: false

    - name: Display role defaults
      command: cat /home/ansible/roles/{{ role_name }}/defaults/main.yml
      register: role_defaults
      changed_when: false

    - name: Display role variables
      command: cat /home/ansible/roles/{{ role_name }}/vars/main.yml
      register: role_vars
      changed_when: false

    - name: Verify role exists
      stat:
        path: "/home/ansible/roles/{{ role_name }}/tasks/main.yml"
      register: role_stat

    - name: Display role verification
      debug:
        msg: "Role {{ role_name }} {{ 'exists' if role_stat.stat.exists else 'not found' }}"
```

### Example 3: Install Role from Galaxy

```yaml
---
- name: Install role from Ansible Galaxy
  hosts: localhost
  connection: local
  become: true
  tasks:
    - name: Create galaxy directory
      file:
        path: /home/ansible/.ansible
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create roles directory
      file:
        path: /home/ansible/roles
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Install geerlingguy.httpd role
      command: ansible-galaxy role install geerlingguy.httpd
      args:
        chdir: /home/ansible

    - name: Install from requirements file
      command: ansible-galaxy role install -r requirements.yml
      args:
        chdir: /home/ansible
      changed_when: false

    - name: List installed roles
      command: ansible-galaxy role list
      args:
        chdir: /home/ansible
      register: installed_roles
      changed_when: false

    - name: Search for httpd roles
      command: ansible-galaxy role search httpd
      register: role_search
      changed_when: false

    - name: Display installed roles
      debug:
        var: installed_roles.stdout_lines

    - name: Display role search results
      debug:
        var: role_search.stdout_lines

    - name: Create requirements.yml
      copy:
        content: |
          ---
          roles:
            - name: geerlingguy.httpd
              version: "3.0.0"
            - name: geerlingguy.firewall
              version: "2.0.0"
        dest: /home/ansible/requirements.yml
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Display requirements file
      command: cat /home/ansible/requirements.yml
      register: requirements
      changed_when: false

    - name: Verify role installation
      stat:
        path: "/home/ansible/roles/geerlingguy.httpd/tasks/main.yml"
      register: role_installed

    - name: Display installation status
      debug:
        msg: "HTTPD role {{ 'installed' if role_installed.stat.exists else 'not installed' }}"
```

### Example 4: Use Collection in Playbook

```yaml
---
- name: Use Ansible collection in playbook
  hosts: all
  become: true
  gather_facts: yes
  tasks:
    # Use fully qualified collection name
    - name: Set sysctl parameter using collection module
      community.general.sysctl:
        name: vm.swappiness
        value: "10"
        state: present
        sysctl_set: yes
      register: sysctl_result
      changed_when: false

    - name: Display sysctl result
      debug:
        msg: "Sysctl {{ 'changed' if sysctl_result.changed else 'unchanged' }}"
      when: sysctl_result is defined

    # Use collection module for package installation
    - name: Install package using collection
      community.general.dnf:
        name:
          - vim-enhanced
          - net-tools
        state: present
      register: package_result
      changed_when: false

    - name: Display package result
      debug:
        msg: "Packages {{ 'installed' if package_result.changed else 'already present' }}"
      when: package_result is defined

    # Use collection module for service management
    - name: Start service using collection
      community.general.systemd:
        name: firewalld
        state: started
        enabled: yes
      register: service_result
      changed_when: false

    - name: Display service result
      debug:
        msg: "Service {{ 'started' if service_result.changed else 'already running' }}"
      when: service_result is defined

    # Use collection module for user management
    - name: Create user using collection
      community.general.user:
        name: testuser
        state: present
        create_home: yes
        shell: /bin/bash
      register: user_result
      changed_when: false

    - name: Display user result
      debug:
        msg: "User {{ 'created' if user_result.changed else 'already exists' }}"
      when: user_result is defined

    # Use collection module for cron jobs
    - name: Create cron job using collection
      community.general.cron:
        name: "Test cron job"
        minute: "0"
        hour: "2"
        job: "/usr/local/bin/backup.sh"
        user: root
      register: cron_result
      changed_when: false

    - name: Display cron result
      debug:
        msg: "Cron job {{ 'created' if cron_result.changed else 'already exists' }}"
      when: cron_result is defined

    # Verify collection is available
    - name: Check if collection is installed
      command: ansible-galaxy collection list
      register: collection_list
      changed_when: false

    - name: Display installed collections
      debug:
        msg: "{{ collection_list.stdout_lines }}"
```

### Example 5: Complete Role and Collection Playbook

```yaml
---
- name: Complete role and collection automation
  hosts: all
  become: true
  gather_facts: yes
  vars:
    app_name: myapp
    app_port: 8080
    log_level: info
  tasks:
    # Use role for web server
    - name: Install web server using role
      include_role:
        name: geerlingguy.httpd
      vars:
        httpd_packages:
          - httpd
          - mod_ssl
        httpd_port: "{{ app_port }}"
      tags: webserver

    # Use collection for system configuration
    - name: Configure system using collection
      community.general.sysctl:
        name: "{{ item }}"
        value: "{{ item_value }}"
        state: present
        sysctl_set: yes
      loop:
        - { name: vm.swappiness, value: "10" }
        - { name: vm.overcommit_memory, value: "1" }
      loop_control:
        loop_var: sysctl_item
      tags: system

    # Use collection for package management
    - name: Install additional packages
      community.general.dnf:
        name:
          - vim-enhanced
          - net-tools
          - wget
          - curl
        state: present
      register: package_install
      changed_when: false
      tags: packages

    # Use collection for service management
    - name: Start services
      community.general.systemd:
        name: "{{ item }}"
        state: started
        enabled: yes
      loop:
        - firewalld
        - sshd
      loop_control:
        label: "{{ item }}"
      tags: services

    # Use collection for user management
    - name: Create application user
      community.general.user:
        name: "{{ app_name }}"
        system: yes
        shell: /sbin/nologin
        create_home: no
      register: user_created
      changed_when: false
      tags: users

    # Use collection for group management
    - name: Create application group
      community.general.group:
        name: "{{ app_name }}"
        system: yes
      register: group_created
      changed_when: false
      tags: users

    # Use collection for file management
    - name: Create application directory
      community.general.file:
        path: /etc/{{ app_name }}
        state: directory
        owner: root
        group: root
        mode: "0755"
      tags: files

    # Use collection for cron management
    - name: Create backup cron job
      community.general.cron:
        name: "Daily backup"
        minute: "0"
        hour: "2"
        job: "/usr/local/bin/backup.sh >> /var/log/backup.log 2>&1"
        user: root
      tags: cron

    # Use collection for SELinux
    - name: Set SELinux boolean
      community.general.seboolean:
        name: httpd_can_network_connect
        state: yes
        persistent: yes
      when: ansible_selinux.status == "enabled"
      ignore_errors: true
      tags: security

    # Use collection for archive management
    - name: Create backup archive
      community.general.archive:
        path: /etc/{{ app_name }}
        dest: /backup/{{ app_name }}-{{ ansible_date_time.date }}.tar.gz
        format: gz
      when: ansible_date_time is defined
      tags: backup

    # Verification
    - name: Verify httpd is running
      community.general.systemd:
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
      community.general.systemd:
        name: httpd
        state: restarted

    - name: reload firewalld
      community.general.systemd:
        name: firewalld
        state: reloaded
```

## Explanation of Each Playbook

### Playbook 1: Create Role

This playbook creates a complete Ansible role from scratch, including all required directories and files. It creates tasks, handlers, templates, defaults, vars, meta, and README files. The role demonstrates web server configuration with Apache HTTPD.

### Playbook 2: Use Role in Playbook

This playbook shows how to include and use roles in playbooks. It demonstrates `include_role` with different options: using defaults, passing parameters, using tags, and conditional execution. It also verifies the role structure and displays role contents.

### Playbook 3: Install Role from Galaxy

This playbook demonstrates installing roles from Ansible Galaxy. It uses `ansible-galaxy role install` to install roles, creates a requirements.yml file for dependency management, and searches for roles by keyword.

### Playbook 4: Use Collection in Playbook

This playbook demonstrates using Ansible Content Collections. It uses collection modules like `community.general.sysctl`, `community.general.dnf`, `community.general.systemd`, and `community.general.user` to configure systems. It shows how to use fully qualified collection names.

### Playbook 5: Complete Automation

This comprehensive playbook combines roles and collections for complete system automation. It uses geerlingguy.httpd role for web server configuration and community.general collection for system configuration, package management, service management, user management, and more.

## Variables and Templates

### Role Variables

```yaml
# roles/webserver/defaults/main.yml
---
httpd_port: 80
app_name: myapp
max_clients: 256
log_level: info
```

```yaml
# roles/webserver/vars/main.yml
---
httpd_packages:
  - httpd
  - mod_ssl
  - firewalld
```

### Using Role Variables in Playbook

```yaml
---
- name: Use role variables
  hosts: all
  become: true
  roles:
    - role: webserver
      httpd_port: 8080
      app_name: myapp
```

### Collection Variables

```yaml
# collections/ansible.posix/collections/ansible_collections/ansible/posix/collection_galaxy_info.yml
galaxy_info:
  author: Ansible
  description: POSIX utilities
  version: 1.0.0
```

## Verification Procedures

### Verify Role Structure

```bash
# List role directories
ls -la roles/

# Check role structure
tree roles/role_name/

# Verify tasks file exists
ls -la roles/role_name/tasks/main.yml

# Verify handlers file exists
ls -la roles/role_name/handlers/main.yml
```

### Verify Collection Installation

```bash
# List installed collections
ansible-galaxy collection list

# Check collection version
ansible-galaxy collection list | grep community.general

# Validate collection
ansible-galaxy collection validate
```

### Verify Role Usage

```bash
# Run role with syntax check
ansible-playbook playbook.yml --syntax-check

# Run role with verbose output
ansible-playbook playbook.yml -vvv
```

## Troubleshooting

### Role Not Found

**Problem: Role not found error**

```bash
# Solution: Check role directory
ls -la roles/

# Solution: Verify role path
ansible-playbook playbook.yml -vvv

# Solution: Install role from Galaxy
ansible-galaxy role install role_name
```

### Collection Not Found

**Problem: Collection not found error**

```bash
# Solution: Check collections path
ansible-galaxy collection list

# Solution: Install collection
ansible-galaxy collection install collection_name

# Solution: Add collections path to ansible.cfg
collections_paths = /home/ansible/.ansible/collections:/usr/share/ansible/collections
```

### Role Parameters Not Working

**Problem: Role parameters not applied**

```yaml
# Solution: Use correct parameter name
- name: Include role
  include_role:
    name: role_name
    vars:
      httpd_port: 8080
```

## Real-World Automation Examples

### Complete Infrastructure with Roles and Collections

```yaml
---
- name: Complete infrastructure automation
  hosts: all
  become: true
  gather_facts: yes
  vars:
    environments:
      - name: production
        httpd_port: 80
      - name: staging
        httpd_port: 8080
  pre_tasks:
    - name: Validate environment
      assert:
        that:
          - environment in environments
        fail_msg: "Invalid environment: {{ environment }}"

  tasks:
    # Install base packages
    - name: Install base packages
      community.general.dnf:
        name:
          - vim-enhanced
          - net-tools
          - wget
          - curl
        state: present
      tags: packages

    # Use webserver role
    - name: Configure web server
      include_role:
        name: geerlingguy.httpd
      vars:
        httpd_port: "{{ environments[0].httpd_port }}"
      tags: webserver

    # Use firewall role
    - name: Configure firewall
      include_role:
        name: geerlingguy.firewall
      vars:
        firewall_services:
          - http
          - https
          - ssh
      tags: firewall

    # Use database role
    - name: Configure database
      include_role:
        name: geerlingguy.postgresql
      when: role_database | default(false)
      tags: database

    # System configuration
    - name: Configure system
      community.general.sysctl:
        name: vm.swappiness
        value: "10"
        state: present
        sysctl_set: yes
      tags: system

    # User management
    - name: Create users
      community.general.user:
        name: "{{ item }}"
        state: present
        create_home: yes
      loop:
        - admin
        - developer
      tags: users

  roles:
    - role: custom_role
      when: custom_role | default(false)
```

## RHCE Exam Notes

### Critical Exam Points

1. **Roles organize automation**: Know role directory structure and how to use roles in playbooks.

2. **Ansible Galaxy**: Know how to search and install roles from Galaxy.

3. **Collections are the future**: Understand collection structure and how to use collection modules.

4. **Fully qualified names**: Use `collection_name.module_name` for modules.

5. **Requirements files**: Use `requirements.yml` for role/collection dependencies.

6. **Role parameters**: Pass variables to roles using `vars:` in `include_role`.

7. **galaxy_info**: Know the purpose of `galaxy_info` in role meta/main.yml.

8. **Collection vs Role**: Understand when to use roles vs collections.

### Exam Workflow

1. Create role with `ansible-galaxy init`
2. Add tasks to tasks/main.yml
3. Add handlers to handlers/main.yml
4. Add defaults to defaults/main.yml
5. Use role in playbook
6. Install collections if needed
7. Test playbook

### Time Management

- 5 minutes: Create role structure
- 10 minutes: Add tasks and handlers
- 5 minutes: Test role
- 5 minutes: Add to playbook

## Common Mistakes

### Mistake 1: Not Using include_role

```yaml
# WRONG - role not executed
- name: Use webserver role
  include_role:
    name: webserver

# CORRECT - role executed
- hosts: all
  become: true
  roles:
    - webserver
```

### Mistake 2: Wrong Collection Name

```yaml
# WRONG - incorrect collection name
- name: Set sysctl
  community.general.sysctl:  # Correct
    name: vm.swappiness
    value: "10"

# WRONG - missing collection prefix
- name: Set sysctl
  sysctl:  # Must use full name
    name: vm.swappiness
    value: "10"
```

### Mistake 3: Not Installing Collection

```bash
# WRONG - collection not found
ansible-playbook playbook.yml

# CORRECT - install collection first
ansible-galaxy collection install community.general
ansible-playbook playbook.yml
```

## Chapter Summary

This chapter covered Ansible roles and Content Collections:

- **Roles** package related tasks, variables, files, and templates into reusable units.
- **Role structure** includes tasks, handlers, templates, files, vars, defaults, meta, and tests directories.
- **Create roles** using `ansible-galaxy init` or manually creating directory structure.
- **Use roles** with `include_role` or in the `roles:` list of a play.
- **Install roles** from Ansible Galaxy using `ansible-galaxy role install`.
- **Collections** are the modern standard for distributing modules, plugins, and roles.
- **Install collections** using `ansible-galaxy collection install`.
- **Use collections** with fully qualified names like `community.general.sysctl`.
- **Requirements files** manage role and collection dependencies.
- **galaxy_info** provides role/collection metadata for Galaxy distribution.

## Quick Reference

| Command | Purpose |
|---|---|
| `ansible-galaxy init role_name` | Create new role |
| `ansible-galaxy role install name` | Install role from Galaxy |
| `ansible-galaxy role list` | List installed roles |
| `ansible-galaxy role search keyword` | Search roles |
| `ansible-galaxy role remove name` | Remove role |
| `ansible-galaxy collection install name` | Install collection |
| `ansible-galaxy collection list` | List collections |
| `ansible-galaxy collection search name` | Search collections |
| `ansible-galaxy collection validate` | Validate collection |
| `ansible-galaxy collection remove name` | Remove collection |
| `include_role: name: role_name` | Include role in playbook |
| `community.general.module_name` | Use collection module |
| `requirements.yml` | Manage dependencies |

## Review Questions

1. What is the purpose of the `tasks/main.yml` file in an Ansible role?

2. How do you create a new Ansible role using ansible-galaxy?

3. What is the difference between `defaults/main.yml` and `vars/main.yml` in a role?

4. How do you include a role in a playbook?

5. What is a Content Collection and how does it differ from a role?

6. How do you use a collection module in a playbook?

7. What is the purpose of `galaxy_info` in a role's meta/main.yml?

8. How do you install a role from Ansible Galaxy?

9. What is a requirements.yml file and what is it used for?

10. How do you pass variables to a role when including it?

## Answers

1. The `tasks/main.yml` file in an Ansible role contains the main task list that is executed when the role is included in a playbook. It defines the automation steps that the role performs, such as installing packages, configuring services, or deploying files.

2. Use `ansible-galaxy init role_name` to create a new Ansible role. This command creates a directory with the standard role structure including tasks, handlers, templates, files, vars, defaults, meta, and tests directories, along with a README.md file.

3. `defaults/main.yml` contains variables that can be overridden by play variables or role parameters. They have low precedence in the variable hierarchy. `vars/main.yml` contains variables that cannot be overridden and have higher precedence. Variables in vars are always used, while defaults are only used if not overridden elsewhere.

4. You can include a role in a playbook using `include_role` within tasks or by listing the role in the `roles:` section of a play. For example: `roles: - webserver` or `include_role: { name: webserver }`.

5. A Content Collection is a packaged, versioned distribution of Ansible content including modules, plugins, roles, and collections. Unlike roles which are directory-based and often manually distributed, collections are standardized packages with metadata, dependency management, and are the modern standard for sharing Ansible content. Collections support better versioning and dependency resolution.

6. Use a fully qualified collection name to use a collection module. For example: `community.general.sysctl: { name: vm.swappiness, value: "10" }`. The module name must be prefixed with the collection name, like `collection_name.module_name`.

7. `galaxy_info` in a role's meta/main.yml contains metadata about the role for Ansible Galaxy. It includes information like author, description, license, minimum Ansible version, supported platforms, tags, and dependencies. This metadata is used when publishing roles to Galaxy and helps users discover and select roles.

8. Use `ansible-galaxy role install role_name` to install a role from Ansible Galaxy. For example: `ansible-galaxy role install geerlingguy.httpd`. You can also install from a requirements.yml file using `ansible-galaxy role install -r requirements.yml`.

9. A requirements.yml file is a YAML file that lists roles and/or collections with their versions. It is used to manage dependencies for multiple projects and can be shared with other developers. It ensures consistent environments across different systems and teams.

10. You pass variables to a role using the `vars:` parameter in `include_role` or as inline variables in the roles list. For example: `include_role: { name: webserver, vars: { httpd_port: 8080 } }` or `roles: - { role: webserver, httpd_port: 8080 }`.


---

# Chapter 12 — Reviewed Edition Closing Checkpoint

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
