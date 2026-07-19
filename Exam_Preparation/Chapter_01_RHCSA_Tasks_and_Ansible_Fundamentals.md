# Chapter 1: Be Able to Perform All Tasks Expected of a Red Hat Certified System Administrator


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


## Added Review: Turning RHCSA Skills into Ansible Habits

RHCE-style automation is not separate from RHCSA. It is RHCSA performed repeatedly, safely, and verifiably across many systems.

Manual RHCSA thinking asks:

```text
What command fixes this server?
```

Ansible thinking asks:

```text
What final state must every matching server have, and which module can enforce that state every time?
```

Examples:

| Manual RHCSA action | Automation habit |
|---|---|
| `dnf install httpd` | `ansible.builtin.dnf: name=httpd state=present` |
| `systemctl enable --now httpd` | `ansible.builtin.systemd_service: name=httpd enabled=true state=started` |
| `firewall-cmd --add-service=http --permanent` | `ansible.posix.firewalld: service=http permanent=true immediate=true state=enabled` when the collection exists |
| `useradd appuser` | `ansible.builtin.user: name=appuser state=present` |
| edit `/etc/ssh/sshd_config` | `ansible.builtin.lineinfile`, `blockinfile`, or `template` with validation |

The most important skill is not memorizing YAML. It is learning to translate a manual state into an idempotent Ansible state.

### Important package-module correction

Do not depend on a module named `dnf_module` unless your local environment proves it exists with `ansible-doc dnf_module` or navigator documentation. For normal package work, use `ansible.builtin.dnf` or, in environments that require it and provide it, `ansible.builtin.dnf5`. For RHEL 10 study, verify the package manager module available in the execution environment before writing final exam playbooks.


## Learning Objectives

By the end of this chapter, you will be able to:

- Understand the relationship between RHCSA competencies and RHCE automation tasks
- Automate essential system administration tasks using Ansible modules
- Manage software packages and repositories through Ansible
- Configure system services and firewalls via automation
- Automate user and group management
- Manage file systems, storage, and LVM using Ansible
- Automate security configurations including SELinux and firewall rules
- Schedule tasks and manage cron jobs with Ansible
- Archive and deploy files to managed nodes
- Analyze and automate shell script workflows
- Use Ansible facts to make decisions based on system state

## Concept Overview

The RHCE EX294 exam assumes RHCSA-level competency. You are not tested on RHCSA tasks directly, but every automation scenario builds upon system administration fundamentals. If you cannot configure a service manually, you cannot automate its configuration reliably.

### Why Automation Matters

Manual system administration does not scale. Consider a fleet of 50 servers requiring the same configuration:

- Manual changes introduce human error
- Inconsistencies accumulate over time
- Auditing and compliance become impossible to verify
- Disaster recovery lacks repeatability

Ansible solves this by treating system state as code. You declare the desired outcome, and Ansible converges every node to that state idempotently.

### How Ansible Solves RHCSA Problems

Every RHCSA task maps to one or more Ansible modules:

| RHCSA Task | Ansible Module(s) |
|---|---|
| Install packages | `ansible.builtin.dnf` / verified `ansible.builtin.dnf5`, and only environment-verified module-stream methods |
| Manage services | `systemd` |
| Configure firewall | `firewalld` |
| Create users | `user`, `group` |
| Manage storage | `lvg`, `lvol`, `mount` |
| Configure SELinux | `selinux`, `seboolean` |
| Schedule tasks | `ansible.builtin.cron` and verified systemd timer unit management |
| Manage files | `copy`, `template`, `lineinfile` |
| Archive files | `archive`, `unarchive` |

## Ansible Fundamentals Required

Before automating RHCSA tasks, understand these core concepts:

### Inventories

An inventory defines which hosts Ansible manages. Static inventories use INI or YAML format.

```ini
# /etc/ansible/hosts
[webservers]
servera.lab.example.com
serverb.lab.example.com

[dbservers]
serverc.lab.example.com

[production:children]
webservers
dbservers
```

### Modules

Modules are the units of work Ansible executes on managed nodes. Each module accepts parameters and returns structured JSON. Ansible ships with hundreds of modules; additional modules live in content collections.

### Variables and Facts

Variables inject dynamic values into playbooks. Facts are variables Ansible collects automatically from managed nodes — CPU count, IP addresses, installed packages, and more.

### Plays and Playbooks

A play targets a group of hosts and executes a list of tasks. A playbook is one or more plays saved in YAML format.

### Privilege Escalation

Most RHCSA tasks require root. Use `become: true` in plays or individual tasks to escalate privileges.

## Commands and Tools

```bash
# Test connectivity to managed nodes
ansible all -m ping

# Gather facts about a host
ansible servera -m setup

# Run a module ad hoc
ansible all -m dnf -a "name=vim state=present"

# Run a playbook
ansible-playbook /path/to/playbook.yml

# Run a playbook via ansible-navigator
ansible-navigator run /path/to/playbook.yml

# List available modules
ansible-doc -l

# Get module documentation
ansible-doc dnf
ansible-doc systemd
ansible-doc firewalld
```

## Playbook Examples

### Example 1: Software Package and Repository Management

```yaml
---
- name: Configure software packages and repositories
  hosts: all
  become: true
  tasks:
    - name: Ensure Crb repository is enabled
      dnf:
        name: "@Crb"
        state: present
        disable_gpg_check: true

    - name: Install required packages
      dnf:
        name:
          - vim-enhanced
          - net-tools
          - wget
          - curl
          - bash-completion
          - tree
          - httpd
        state: present

    - name: Install latest version of specific package
      dnf:
        name: firewalld
        state: latest

    - name: Remove unwanted packages
      dnf:
        name:
          - telnet
          - rsyslog
        state: absent

    - name: Install DNF module for PostgreSQL
      ansible.builtin.command: dnf module list postgresql
      register: postgresql_module_streams
      changed_when: false
      failed_when: false

    - name: Install PostgreSQL 16 server
      dnf:
        name: postgresql-server
        state: present

    - name: Ensure package cache is refreshed
      dnf:
        name: "*"
        state: present
        update_cache: yes

    - name: Add custom repository
      copy:
        content: |
          [custom-repo]
          name=Custom Repository
          baseurl=http://content.example.com/custom
          gpgcheck=0
          enabled=1
        dest: /etc/yum.repos.d/custom.repo
        owner: root
        group: root
        mode: "0644"
```

### Example 2: Service Management

```yaml
---
- name: Configure system services
  hosts: all
  become: true
  tasks:
    - name: Ensure firewalld is enabled and running
      systemd:
        name: firewalld
        state: started
        enabled: yes

    - name: Ensure httpd is enabled and running
      systemd:
        name: httpd
        state: started
        enabled: yes

    - name: Stop and disable unnecessary services
      systemd:
        name: "{{ item }}"
        state: stopped
        enabled: no
      loop:
        - telnet
        - cups

    - name: Reload httpd configuration without restarting
      systemd:
        name: httpd
        state: reloaded
      when: ansible_facts.services | select("attr", "name", "match", "httpd") | list | length > 0

    - name: Restart service on configuration change
      systemd:
        name: sshd
        daemon_reload: yes
        state: restarted
```

### Example 3: Firewall Configuration

```yaml
---
- name: Configure firewall rules
  hosts: all
  become: true
  tasks:
    - name: Ensure firewalld is running
      systemd:
        name: firewalld
        state: started
        enabled: yes

    - name: Open HTTP and HTTPS permanently
      firewalld:
        port: "{{ item }}"
        permanent: yes
        immediate: yes
        state: enabled
      loop:
        - "80/tcp"
        - "443/tcp"

    - name: Open custom port range
      firewalld:
        port: "8080-8090/tcp"
        permanent: yes
        immediate: yes
        state: enabled

    - name: Allow SSH service
      firewalld:
        service: ssh
        permanent: yes
        immediate: yes
        state: enabled

    - name: Set default zone to public
      firewalld:
        zone: public
        default: yes
        permanent: yes

    - name: Forward port 8443 to 443
      firewalld:
        port: "8443/tcp"
        toport: "443/tcp"
        toaddr: ""
        permanent: yes
        immediate: yes
        state: enabled

    - name: Remove unused service
      firewalld:
        service: dhcpv6-client
        permanent: yes
        immediate: yes
        state: disabled
```

### Example 4: User and Group Management

```yaml
---
- name: Manage users and groups
  hosts: all
  become: true
  vars:
    admin_group_members:
      - alice
      - bob
    standard_users:
      - name: charlie
        uid: 3001
        groups: wheel
        comment: "Charlie Admin"
      - name: dave
        uid: 3002
        groups: docker
        comment: "Dave Developer"
      - name: eve
        uid: 3003
        comment: "Eve Temp User"
        shell: /sbin/nologin
  tasks:
    - name: Create administrative group
      group:
        name: admins
        system: yes

    - name: Add users to admin group
      user:
        name: "{{ item }}"
        groups: admins
        append: yes
      loop: "{{ admin_group_members }}"

    - name: Create standard users with specific attributes
      user:
        name: "{{ item.name }}"
        uid: "{{ item.uid | default(omit) }}"
        groups: "{{ item.groups | default(omit) }}"
        comment: "{{ item.comment | default(omit) }}"
        shell: "{{ item.shell | default(omit) }}"
        createhome: yes
        state: present
      loop: "{{ standard_users }}"

    - name: Set password for user alice
      user:
        name: alice
        password: "{{ 'P@ssw0rd123' | password_hash('sha512') }}"
        update_password: on_create

    - name: Lock user account
      user:
        name: eve
        password_lock: lock

    - name: Remove user and home directory
      user:
        name: olduser
        state: absent
        remove: yes

    - name: Ensure sudoers file has wheel group entry
      lineinfile:
        path: /etc/sudoers.d/wheel
        line: "%wheel ALL=(ALL) NOPASSWD: ALL"
        create: yes
        validate: "visudo -cf %s"
        mode: "0440"
```

### Example 5: File System and Storage Management

```yaml
---
- name: Configure file systems and storage
  hosts: all
  become: true
  tasks:
    - name: Ensure device exists and is recognized
      command: lsblk
      register: disk_info
      changed_when: false

    - name: Create volume group on new disk
      lvg:
        vg: vg_data
        pvs: /dev/vdb
      when: ansible_facts.devices.vdb is defined

    - name: Create logical volume
      lvol:
        vg: vg_data
        lv: lv_storage
        size: 10G
      when: ansible_facts.devices.vdb is defined

    - name: Create XFS filesystem
      filesystem:
        dev: /dev/vg_data/lv_storage
        fstype: xfs
      when: ansible_facts.devices.vdb is defined

    - name: Mount filesystem persistently
      mount:
        path: /mnt/data
        src: /dev/vg_data/lv_storage
        fstype: xfs
        state: mounted
        opts: defaults
      when: ansible_facts.devices.vdb is defined

    - name: Create mount point directory
      file:
        path: /mnt/data
        state: directory
        owner: root
        group: root
        mode: "0755"

    - name: Mount tmpfs for temporary storage
      mount:
        path: /tmp/shared
        src: tmpfs
        fstype: tmpfs
        state: mounted
        opts: "size=1G,mode=1777"
```

### Example 6: File Content Management

```yaml
---
- name: Manage file content
  hosts: all
  become: true
  tasks:
    - name: Create configuration directory
      file:
        path: /etc/myapp
        state: directory
        owner: root
        group: root
        mode: "0755"

    - name: Deploy configuration file
      copy:
        content: |
          [main]
          log_level = info
          max_connections = 100
        dest: /etc/myapp/config.ini
        owner: root
        group: root
        mode: "0644"

    - name: Add line to configuration if not present
      lineinfile:
        path: /etc/myapp/config.ini
        line: "timeout = 30"
        insertafter: "\\[main\\]"
        state: present

    - name: Replace existing value
      lineinfile:
        path: /etc/myapp/config.ini
        regexp: "^log_level = "
        line: "log_level = debug"
        state: present

    - name: Remove a line from configuration
      lineinfile:
        path: /etc/myapp/config.ini
        regexp: "^timeout = "
        state: absent

    - name: Block replacement in configuration
      blockinfile:
        path: /etc/myapp/config.ini
        block: |
          [logging]
          file = /var/log/myapp/app.log
          rotate = 7
        marker: "# {mark} ANSIBLE MANAGED BLOCK"
        state: present

    - name: Ensure file permissions are correct
      file:
        path: /etc/myapp/config.ini
        owner: root
        group: myapp
        mode: "0640"
        state: file
```

### Example 7: Task Scheduling

```yaml
---
- name: Configure task scheduling
  hosts: all
  become: true
  tasks:
    - name: Create daily backup cron job
      cron:
        name: "Daily backup"
        minute: "0"
        hour: "2"
        day: "*"
        month: "*"
        weekday: "*"
        job: "/usr/local/bin/backup.sh >> /var/log/backup.log 2>&1"
        user: root

    - name: Create weekly report cron job
      cron:
        name: "Weekly report"
        special_time: weekly
        job: "/usr/local/bin/generate_report.sh"
        user: root

    - name: Run task every 15 minutes
      cron:
        name: "Health check"
        minute: "*/15"
        job: "/usr/local/bin/health_check.sh"
        user: root

    - name: Remove a cron job by name
      cron:
        name: "Old backup job"
        state: absent

    - name: Create anacron job for intermittent systems
      cron:
        name: "Monthly maintenance"
        job: "/usr/local/bin/monthly_maintenance.sh"
        day: "1"
        month: "*"
        user: root
        env: "PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin"
```

### Example 8: Security Configuration

```yaml
---
- name: Configure system security
  hosts: all
  become: true
  tasks:
    - name: Ensure SELinux is enforcing
      selinux:
        state: enforcing
        policy: targeted

    - name: Set SELinux boolean permanently
      seboolean:
        name: httpd_can_network_connect
        state: yes
        persistent: yes

    - name: Set SELinux file context
      sefcontext:
        target: "/var/www/html/uploads(/.*)?"
        setype: httpd_sys_rw_content_t
        state: present

    - name: Apply SELinux file context
      command: restorecon -Rv /var/www/html/uploads
      changed_when: false

    - name: Configure SSH security
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
        state: present
        backup: yes
      loop:
        - { regexp: "^#?PermitRootLogin", line: "PermitRootLogin no" }
        - { regexp: "^#?PasswordAuthentication", line: "PasswordAuthentication no" }
        - { regexp: "^#?X11Forwarding", line: "X11Forwarding no" }

    - name: Restart SSH after configuration change
      systemd:
        name: sshd
        state: restarted

    - name: Set password aging policy
      pam:
        name: system-auth
        param: password
        control: required
        type: pam_pwquality.so
        new_type: pam_pwquality.so retry=3 minlen=14

    - name: Configure login.defs password aging
      lineinfile:
        path: /etc/login.defs
        regexp: "^{{ item.key }}"
        line: "{{ item.key }} {{ item.value }}"
      loop:
        - { key: "PASS_MAX_DAYS", value: "90" }
        - { key: "PASS_MIN_DAYS", value: "7" }
        - { key: "PASS_MIN_LEN", value: "14" }
```

### Example 9: Archiving

```yaml
---
- name: Manage archives
  hosts: all
  become: true
  tasks:
    - name: Create tar.gz archive of configuration
      archive:
        path:
          - /etc/myapp
          - /var/log/myapp
        dest: /backup/myapp-config-{{ ansible_date_time.date }}.tar.gz
        format: gz
        owner: root
        group: root
        mode: "0600"

    - name: Create compressed archive with specific owner
      archive:
        path: /var/www/html
        dest: /backup/website.tar.bz2
        format: bz2
        owner: root
        group: root
        mode: "0640"
        remove: no

    - name: Extract archive to destination
      unarchive:
        src: /backup/myapp-config-latest.tar.gz
        dest: /restore/
        owner: root
        group: root
        mode: "0644"

    - name: Extract remote archive
      unarchive:
        src: http://example.com/packages/app.tar.gz
        dest: /opt/
        remote_src: yes
```

### Example 10: Shell Script Analysis and Automation

```yaml
---
- name: Analyze and automate shell script tasks
  hosts: all
  become: true
  tasks:
    - name: Check if script exists
      stat:
        path: /usr/local/bin/backup.sh
      register: script_check

    - name: Deploy backup script
      copy:
        content: |
          #!/bin/bash
          # Backup script for system configuration
          set -euo pipefail

          BACKUP_DIR="/backup/$(date +%Y%m%d)"
          mkdir -p "$BACKUP_DIR"

          # Backup critical directories
          tar czf "$BACKUP_DIR/etc.tar.gz" /etc
          tar czf "$BACKUP_DIR/var-log.tar.gz" /var/log

          # Cleanup backups older than 30 days
          find /backup -maxdepth 1 -type d -mtime +30 -exec rm -rf {} \;

          echo "Backup completed: $BACKUP_DIR"
        dest: /usr/local/bin/backup.sh
        owner: root
        group: root
        mode: "0755"

    - name: Run script and capture output
      command: /usr/local/bin/backup.sh
      register: backup_result
      changed_when: false

    - name: Display backup result
      debug:
        var: backup_result.stdout

    - name: Run inline shell commands
      shell: |
        for file in /var/log/*.log; do
          gzip "$file"
        done
      args:
        creates: /var/log/*.gz
      register: compress_result

    - name: Use shell for complex logic
      shell: |
        df -h / | awk 'NR==2 {print $5}' | tr -d '%'
      register: disk_usage
      changed_when: false

    - name: Alert if disk usage exceeds threshold
      debug:
        msg: "WARNING: Disk usage is {{ disk_usage.stdout }}%"
      when: disk_usage.stdout | int > 80
```

## Explanation of Each Playbook

### Playbook 1: Software Package Management

This playbook demonstrates the `dnf` module, which replaces the manual `dnf install` and `dnf remove` commands. The `state: present` parameter ensures packages exist without upgrading unnecessarily. The `state: latest` parameter upgrades to the newest available version. DNF module-stream handling is environment-sensitive; verify the supported method in the provided execution environment before using module streams. The `update_cache: yes` parameter forces a metadata refresh, equivalent to running `dnf makecache`.

The `copy` module at the end demonstrates how to add custom repositories programmatically. This replaces the manual process of creating `.repo` files in `/etc/yum.repos.d/`.

### Playbook 2: Service Management

The `systemd` module replaces `systemctl` commands. Setting `state: started` and `enabled: yes` ensures a service is both running now and will start at boot. The `daemon_reload: yes` parameter runs `systemctl daemon-reload` before making changes, which is critical after modifying unit files. The `when` conditional uses Ansible facts to check whether a service exists before attempting to reload it, preventing failures on nodes where the service is not installed.

### Playbook 3: Firewall Configuration

The `firewalld` module replaces `firewall-cmd` commands. Setting `permanent: yes` and `immediate: yes` applies rules to both the runtime and permanent configuration, equivalent to using `--permanent` and then `--reload`. The `loop` directive iterates over a list of ports, avoiding task duplication. Port ranges use the hyphen syntax (`8080-8090/tcp`).

### Playbook 4: User and Group Management

The `group` module creates system groups. The `user` module handles user creation with the `createhome: yes` flag. The `append: yes` parameter adds users to supplemental groups without removing existing group memberships. The `password_hash` filter generates a hashed password string suitable for the `password` parameter. The `update_password: on_create` setting ensures existing passwords are not overwritten on subsequent runs.

### Playbook 5: File System and Storage

The `lvg` and `lvol` modules manage LVM volume groups and logical volumes. The `filesystem` module formats devices. The `mount` module with `state: mounted` both adds the entry to `/etc/fstab` and mounts the filesystem immediately. The `when` clause checks `ansible_facts.devices.vdb` to conditionally run storage tasks only on hosts with the required disk.

### Playbook 6: File Content Management

This playbook demonstrates four approaches to file management:

1. `copy` with `content` — writes inline text to a file
2. `lineinfile` — manages individual lines with regex matching
3. `blockinfile` — manages multi-line blocks with markers
4. `file` — sets ownership and permissions

The `blockinfile` module is particularly valuable because it uses marker comments to track managed content, ensuring idempotency when modifying configuration files that contain both managed and unmanaged sections.

### Playbook 7: Task Scheduling

The `cron` module replaces manual `crontab` editing. Each cron job is identified by a `name` field, which Ansible uses to ensure idempotency. The `special_time` parameter supports named schedules like `weekly`, `monthly`, `yearly`, `reboot`, and `midnight`. The `env` parameter sets environment variables within the cron context.

### Playbook 8: Security Configuration

The `selinux` module sets the global SELinux mode. The `seboolean` module toggles SELinux booleans persistently. The `sefcontext` module defines custom file context rules, which must be followed by `restorecon` to apply them. The `lineinfile` module with a `loop` demonstrates batch configuration of SSH parameters. The `validate` parameter in the sudoers task runs `visudo -cf` before writing, preventing syntax errors from locking out sudo access.

### Playbook 9: Archiving

The `archive` module creates compressed archives with configurable formats (`gz`, `bz2`, `xz`, `zip`). The `unarchive` module extracts archives. Setting `remote_src: yes` tells Ansible the source archive already exists on the managed node rather than needing to be transferred from the control node.

### Playbook 10: Shell Script Automation

The `command` module runs binaries without shell interpretation. The `shell` module runs commands through `/bin/sh`, enabling pipes, redirections, and loops. The `creates` parameter provides idempotency by skipping execution if a file already exists. The `stat` module checks file existence without running commands.

## Variables and Templates

### Using Registered Variables

Capture command output for conditional logic:

```yaml
---
- name: Use registered variables
  hosts: all
  become: true
  tasks:
    - name: Check disk usage
      command: df -h /
      register: disk_output
      changed_when: false

    - name: Display disk usage
      debug:
        var: disk_output.stdout_lines

    - name: Get package information
      command: rpm -q vim-enhanced
      register: vim_check
      changed_when: false
      failed_when: false

    - name: Report package status
      debug:
        msg: "Vim is installed"
      when: vim_check.rc == 0
```

### Using Jinja2 Templates

Templates enable dynamic configuration file generation:

```yaml
# Template: httpd.conf.j2
ServerRoot "/etc/httpd"
Listen {{ httpd_port | default(80) }}
MaxClients {{ max_clients | default(256) }}

<VirtualHost *:{{ httpd_port | default(80) }}>
    ServerName {{ inventory_hostname }}
    DocumentRoot "/var/www/{{ app_name }}"
{% if enable_ssl %}
    SSLEngine on
    SSLCertificateFile /etc/pki/tls/certs/server.crt
    SSLCertificateKeyFile /etc/pki/tls/private/server.key
{% endif %}
</VirtualHost>
```

```yaml
# Playbook using template
---
- name: Deploy httpd configuration
  hosts: all
  become: true
  vars:
    httpd_port: 8080
    app_name: myapp
    enable_ssl: true
  tasks:
    - name: Deploy httpd configuration from template
      template:
        src: httpd.conf.j2
        dest: /etc/httpd/conf.d/myapp.conf
        owner: root
        group: root
        mode: "0644"
        validate: "httpd -t -f %s"

    - name: Restart httpd after config change
      systemd:
        name: httpd
        state: restarted
      notify: never
```

### Template Conditional Logic

```yaml
# Firewall template: firewalld-zones.conf.j2
# Managed zones configuration
{% for zone in enabled_zones %}
# Zone: {{ zone.name }}
# Target: {{ zone.target | default('DEFAULT') }}
{% for service in zone.services %}
zone={{ zone.name }} service={{ service }} enable
{% endfor %}
{% endfor %}
```

## Verification Procedures

### Verify Package Installation

```bash
# Manual verification
rpm -q vim-enhanced net-tools wget curl httpd

# Ansible verification
ansible all -m dnf -a "list_installed vim-enhanced"
ansible all -m command -a "rpm -qa | grep httpd"
```

### Verify Service Status

```bash
# Manual verification
systemctl is-active firewalld httpd
systemctl is-enabled firewalld httpd

# Ansible verification
ansible all -m command -a "systemctl is-active firewalld"
ansible all -m setup -a "filter=ansible_facts.services"
```

### Verify Firewall Rules

```bash
# Manual verification
firewall-cmd --list-all
firewall-cmd --list-all-zones

# Ansible verification
ansible all -m command -a "firewall-cmd --list-services"
```

### Verify User Accounts

```bash
# Manual verification
id alice
id charlie
getent group admins
grep alice /etc/shadow | cut -c1-2

# Ansible verification
ansible all -m command -a "id alice"
ansible all -m getent -a "database=group key=admins"
```

### Verify File Systems

```bash
# Manual verification
df -hT /mnt/data
lsblk
vgs
lvs

# Ansible verification
ansible all -m command -a "df -hT /mnt/data"
ansible all -m setup -a "filter=ansible_facts.mounts"
```

### Verify Cron Jobs

```bash
# Manual verification
crontab -l
cat /etc/cron.d/*

# Ansible verification
ansible all -m command -a "crontab -l"
```

### Verify SELinux

```bash
# Manual verification
getenforce
sestatus
getsebool httpd_can_network_connect
ls -Z /var/www/html/uploads

# Ansible verification
ansible all -m command -a "getenforce"
ansible all -m command -a "getsebool -a | grep httpd"
```

## Troubleshooting

### Module Execution Failures

```yaml
# Debug failed tasks by increasing verbosity
ansible-playbook playbook.yml -vvv

# Run specific play or task
ansible-playbook playbook.yml --start-at-task="Install required packages"

# Dry run to preview changes
ansible-playbook playbook.yml --check --diff
```

### Common Issues and Solutions

**Issue: GPG key errors during package installation**

```yaml
# Solution: Disable GPG check for trusted internal repos
dnf:
  name: mypackage
  state: present
  disable_gpg_check: true
```

**Issue: Service fails to start because dependencies are not installed**

```yaml
# Solution: Use handlers and proper task ordering
tasks:
  - name: Install httpd
    dnf:
      name: httpd
      state: present
    notify: restart httpd

  - name: Deploy configuration
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

**Issue: SELinux prevents service from functioning**

```yaml
# Solution: Set SELinux context before starting service
tasks:
  - name: Set SELinux boolean
    seboolean:
      name: httpd_can_network_connect
      state: yes
      persistent: yes

  - name: Apply file contexts
    command: restorecon -Rv /var/www
    changed_when: false
```

**Issue: Mount fails because device does not exist**

```yaml
# Solution: Check device existence before mounting
tasks:
  - name: Check if device exists
    stat:
      path: /dev/vg_data/lv_storage
    register: lv_check

  - name: Mount filesystem
    mount:
      path: /mnt/data
      src: /dev/vg_data/lv_storage
      fstype: xfs
      state: mounted
    when: lv_check.stat.exists
```

**Issue: Cron job not executing**

```yaml
# Solution: Ensure PATH is set and script is executable
cron:
  name: "Backup job"
  job: "/bin/bash /usr/local/bin/backup.sh"
  user: root
  env: "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
```

**Issue: Privilege escalation failures**

```yaml
# Solution: Verify become configuration
ansible all -m ping -b
# Check /etc/sudoers on managed node
# Ensure ansible user has sudo privileges
```

**Issue: Template rendering errors**

```yaml
# Solution: Validate template syntax
ansible localhost -m template -a "src=mytemplate.j2 dest=/dev/null"
# Check for undefined variables
ansible-playbook playbook.yml --check
```

## Real-World Automation Examples

### Complete Web Server Provisioning

```yaml
---
- name: Provision complete web server
  hosts: webservers
  become: true
  vars:
    app_name: myapp
    document_root: "/var/www/{{ app_name }}"
    httpd_port: 80
  tasks:
    - name: Update all packages to latest
      dnf:
        name: "*"
        state: latest
        update_cache: yes

    - name: Install web server packages
      dnf:
        name:
          - httpd
          - mod_ssl
          - firewalld
        state: present

    - name: Create document root
      file:
        path: "{{ document_root }}"
        state: directory
        owner: apache
        group: apache
        mode: "0755"

    - name: Deploy default page
      copy:
        content: |
          <html>
          <head><title>{{ inventory_hostname }}</title></head>
          <body>
          <h1>Welcome to {{ inventory_hostname }}</h1>
          <p>Managed by Ansible</p>
          </body>
          </html>
        dest: "{{ document_root }}/index.html"
        owner: apache
        group: apache
        mode: "0644"

    - name: Configure SELinux for web content
      sefcontext:
        target: "{{ document_root }}(/.*)?"
        setype: httpd_sys_content_t
        state: present

    - name: Apply SELinux contexts
      command: restorecon -Rv "{{ document_root }}"
      changed_when: false

    - name: Open HTTP port
      firewalld:
        port: "{{ httpd_port }}/tcp"
        permanent: yes
        immediate: yes
        state: enabled

    - name: Enable and start httpd
      systemd:
        name: httpd
        state: started
        enabled: yes

    - name: Verify web server is responding
      uri:
        url: "http://localhost:{{ httpd_port }}"
        status_code: 200
      retries: 5
      delay: 2
```

### System Hardening Playbook

```yaml
---
- name: Harden system security
  hosts: all
  become: true
  tasks:
    - name: Enable SELinux enforcing mode
      selinux:
        state: enforcing
        policy: targeted

    - name: Configure SSH security settings
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
        backup: yes
      loop:
        - { regexp: "^#?PermitRootLogin", line: "PermitRootLogin no" }
        - { regexp: "^#?PasswordAuthentication", line: "PasswordAuthentication no" }
        - { regexp: "^#?PermitEmptyPasswords", line: "PermitEmptyPasswords no" }
        - { regexp: "^#?X11Forwarding", line: "X11Forwarding no" }
      notify: restart sshd

    - name: Set strong password policy
      lineinfile:
        path: /etc/login.defs
        regexp: "^{{ item.key }}"
        line: "{{ item.key }} {{ item.value }}"
      loop:
        - { key: "PASS_MAX_DAYS", value: "90" }
        - { key: "PASS_MIN_DAYS", value: "7" }
        - { key: "PASS_MIN_LEN", value: "14" }
        - { key: "PASS_WARN_AGE", value: "7" }

    - name: Restrict unused ports
      firewalld:
        port: "{{ item }}"
        permanent: yes
        immediate: yes
        state: disabled
      loop:
        - "21/tcp"
        - "23/tcp"

    - name: Ensure auditd is running
      systemd:
        name: auditd
        state: started
        enabled: yes

    - name: Create system administration user
      user:
        name: sysadmin
        groups: wheel
        shell: /bin/bash
        createhome: yes
        state: present

    - name: Set sysadmin password
      user:
        name: sysadmin
        password: "{{ 'SecureP@ss2024!' | password_hash('sha512') }}"
        update_password: on_create

handlers:
  - name: restart sshd
    systemd:
      name: sshd
      state: restarted
```

## RHCE Exam Notes

### Critical Exam Points

1. **RHCSA knowledge is assumed**: You will not be given RHCSA-specific tasks, but every automation scenario requires understanding of the underlying system administration concepts. Know how to troubleshoot when an Ansible module fails because of a system-level issue.

2. **ansible-navigator is your primary tool**: The exam environment provides ansible-navigator. Know how to:
   - Run playbooks: `ansible-navigator run playbook.yml`
   - Search documentation: Use the documentation browser to look up module parameters
   - Create and edit inventories from within the interface
   - Switch between modes (standalone, execution environment)

3. **Use provided documentation**: The exam provides access to Ansible documentation. You are expected to look up module parameters rather than memorize them. Know how to search for modules by keyword.

4. **Idempotency matters**: Your playbooks should be safe to run multiple times. A task that changes system state every time it runs will fail on the second execution or produce unintended side effects.

5. **Privilege escalation**: Most tasks require root. Always include `become: true` at the play level unless a task explicitly should not run as root.

6. **Time management**: You have 4 hours. Do not spend excessive time on any single task. If a playbook fails, check the error message, fix the specific issue, and move forward.

### Exam Environment Setup

The exam provides:
- A control node with Ansible installed
- ansible-navigator configured
- Access to Ansible documentation
- Managed nodes defined in inventory
- SSH key-based authentication pre-configured
- Content collections available for installation

### Scoring Strategy

- Partial credit is awarded for partially working playbooks
- Focus on getting tasks working rather than producing perfect code
- Use `--check` mode to validate playbooks before running them for real
- Save frequently and commit to Git as you work

## Common Mistakes

### Mistake 1: Forgetting `become: true`

Tasks that modify system configuration will fail without privilege escalation.

```yaml
# WRONG - will fail for system-level tasks
- name: Install package
  dnf:
    name: vim
    state: present

# CORRECT
- name: Install package
  become: true
  dnf:
    name: vim
    state: present
```

### Mistake 2: Incorrect File Mode Syntax

File modes must be quoted strings with leading zero.

```yaml
# WRONG - interpreted as octal 644 = decimal 420
mode: 0644

# CORRECT - treated as string, then converted to octal
mode: "0644"
```

### Mistake 3: Using `shell` When `command` Suffices

The `shell` module invokes a shell interpreter, which introduces security risks and inconsistent behavior across systems.

```yaml
# WRONG - unnecessary shell usage
- name: Run command
  shell: ls /tmp

# CORRECT - direct command execution
- name: Run command
  command: ls /tmp
```

### Mistake 4: Not Handling Missing Dependencies

A service cannot start if its package is not installed.

```yaml
# WRONG - service may fail if package is missing
- name: Start httpd
  systemd:
    name: httpd
    state: started

# CORRECT - install package first
- name: Install httpd
  dnf:
    name: httpd
    state: present

- name: Start httpd
  systemd:
    name: httpd
    state: started
    enabled: yes
```

### Mistake 5: Overwriting Files Without Backup

```yaml
# WRONG - destroys existing content
- name: Edit config
  copy:
    content: "new setting"
    dest: /etc/myapp/config.ini

# CORRECT - preserve existing content with backup
- name: Edit config
  lineinfile:
    path: /etc/myapp/config.ini
    regexp: "^setting"
    line: "setting = value"
    backup: yes
```

### Mistake 6: Ignoring `changed_when`

Commands that do not modify system state should report no change.

```yaml
# WRONG - always reports changed
- name: Check disk usage
  command: df -h /

# CORRECT - never reports changed
- name: Check disk usage
  command: df -h /
  changed_when: false
```

### Mistake 7: Hardcoding Hostnames

```yaml
# WRONG - breaks on different hosts
- name: Deploy config
  template:
    src: config.j2
    dest: /etc/myapp/config
  vars:
    server_name: "servera.lab.example.com"

# CORRECT - uses inventory hostname
# Template uses {{ inventory_hostname }} automatically
```

### Mistake 8: Not Using Handlers for Service Restart

```yaml
# WRONG - restarts service every run
- name: Deploy config
  copy:
    src: httpd.conf
    dest: /etc/httpd/conf/httpd.conf
  notify: never

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

## Chapter Summary

This chapter established the foundation for RHCE automation by connecting RHCSA system administration competencies to Ansible modules. Key takeaways:

- Every RHCSA task has a corresponding Ansible module
- The `dnf` module manages packages; `systemd` manages services; `firewalld` manages firewall rules
- User and group management uses the `user` and `group` modules
- Storage and file systems are managed through `lvg`, `lvol`, `filesystem`, and `mount` modules
- File content is controlled via `copy`, `template`, `lineinfile`, and `blockinfile`
- Cron jobs are managed with the `cron` module
- SELinux is configured through `selinux`, `seboolean`, and `sefcontext` modules
- Archives are created and extracted with `archive` and `unarchive`
- Shell scripts can be analyzed, deployed, and executed via `command` and `shell` modules
- Registered variables capture command output for conditional logic
- Jinja2 templates enable dynamic configuration file generation
- Idempotency ensures playbooks are safe to run repeatedly
- `become: true` is required for most system-level tasks
- ansible-navigator is the primary interface in the exam environment

## Quick Reference

| Task | Module | Key Parameters |
|---|---|---|
| Install package | `dnf` | `name`, `state: present` |
| Remove package | `dnf` | `name`, `state: absent` |
| Start service | `systemd` | `name`, `state: started`, `enabled: yes` |
| Stop service | `systemd` | `name`, `state: stopped`, `enabled: no` |
| Open firewall port | `firewalld` | `port`, `permanent: yes`, `immediate: yes` |
| Create user | `user` | `name`, `createhome: yes` |
| Create group | `group` | `name` |
| Mount filesystem | `mount` | `src`, `path`, `fstype`, `state: mounted` |
| Create LVM volume | `lvol` | `vg`, `lv`, `size` |
| Copy file | `copy` | `src`, `dest`, `owner`, `group`, `mode` |
| Deploy template | `template` | `src`, `dest` |
| Edit single line | `lineinfile` | `path`, `regexp`, `line` |
| Edit block | `blockinfile` | `path`, `block` |
| Create cron job | `cron` | `name`, `minute`, `hour`, `job` |
| Set SELinux mode | `selinux` | `state: enforcing` |
| Set SELinux boolean | `seboolean` | `name`, `state`, `persistent: yes` |
| Create archive | `archive` | `path`, `dest`, `format` |
| Extract archive | `unarchive` | `src`, `dest` |
| Run command | `command` | (command string) |
| Run shell command | `shell` | (shell command string) |
| Check file exists | `stat` | `path` |

## Review Questions

1. Which Ansible module would you use to install a software package on a RHEL system, and what parameter ensures the package is present without upgrading to a newer version?

2. What is the difference between `state: started` and `enabled: yes` in the `systemd` module?

3. Why must `permanent: yes` and `immediate: yes` both be set in the `firewalld` module for production use?

4. What is the purpose of the `become: true` directive in an Ansible play?

5. How does the `lineinfile` module ensure idempotency when modifying a configuration file?

6. What is the role of the `changed_when: false` parameter, and when should it be used?

7. What is the difference between the `command` and `shell` modules?

8. How do Jinja2 templates differ from the `copy` module with the `content` parameter?

9. What does the `notify` directive do in relation to handlers?

10. Why is it important to use `validate: "visudo -cf %s"` when writing to sudoers files?

## Answers

1. Use the `dnf` module with `state: present`. This ensures the package is installed but does not upgrade it. Use `state: latest` if you want the newest version.

2. `state: started` starts the service immediately during the playbook run. `enabled: yes` ensures the service starts automatically at boot time. Both should typically be set together for services that should be running now and persist across reboots.

3. `permanent: yes` writes the rule to the persistent configuration so it survives reboots. `immediate: yes` applies the rule to the running firewall without requiring a reload. Without both, the rule either disappears after reboot or does not take effect until the next reload.

4. `become: true` enables privilege escalation, typically to root. Most system administration tasks require root privileges. Without this directive, tasks that modify system configuration, install packages, or manage services will fail with permission errors.

5. The `lineinfile` module uses the `regexp` parameter to search for an existing matching line. If found, it replaces it with the specified `line`. If not found, it adds the line. This ensures the file reaches the desired state regardless of its current content, making the task idempotent.

6. The `changed_when: false` parameter tells Ansible to never report the task as changed, regardless of the command output. It should be used for read-only operations like checking disk usage, listing files, or gathering information — tasks that do not modify system state.

7. The `command` module executes a binary directly without shell interpretation. It does not support pipes, redirections, or shell builtins. The `shell` module executes commands through `/bin/sh`, enabling full shell features. Use `command` by default for security and consistency; use `shell` only when shell features are required.

8. Jinja2 templates (used with the `template` module) support variables, conditionals, loops, and filters, enabling dynamic content generation. The `copy` module with `content` writes static text. Templates are essential when configuration files must differ per host or adapt to system facts.

9. The `notify` directive triggers a handler when the task reports a change. Handlers run once at the end of the play, regardless of how many tasks notify them. This prevents unnecessary service restarts when configuration files have not actually changed.

10. The `validate` parameter runs the specified command with the temporary file path substituted for `%s`. For sudoers files, `visudo -cf` checks syntax before the file is written. Without this, a syntax error in the sudoers file can lock out all sudo access, requiring single-user mode recovery.


---

# Chapter 01 — Reviewed Edition Closing Checkpoint

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
