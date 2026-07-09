# Chapter 7: Deploy Files to Managed Nodes

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand file deployment methods in Ansible
- Use the `copy` module to deploy files from the control node
- Use the `template` module to deploy Jinja2 templates
- Use the `fetch` module to retrieve files from managed nodes
- Configure file ownership, permissions, and attributes
- Deploy configuration files with variable substitution
- Manage file symlinks and hard links
- Handle archive deployment and extraction
- Use `lineinfile` and `blockinfile` for file content management
- Create and manage file directories
- Troubleshoot file deployment issues

## Concept Overview

File deployment is a fundamental Ansible task. Playbooks must deploy configuration files, scripts, and data to managed nodes to configure systems. Ansible provides several modules for file management, each with specific use cases.

### Why File Deployment Matters

Without file deployment:

- Configuration files don't exist on managed nodes
- Applications cannot start
- System settings cannot be configured
- Custom scripts cannot run
- Automation is incomplete

### How File Deployment Works

```
Control Node                          Managed Node
─────────────                         ────────────
1. Playbook defines file content
   │
   ├── Static content (copy module)
   │
   └── Template content (template module)
         │
         ├── Variables substituted
         │
         └── Jinja2 processing

2. File transferred to managed node
   │
   ├── Overwrites existing file
   │
   └── Creates file if not exists

3. File permissions set
   │
   ├── Ownership configured
   │
   └── Permissions applied
```

## Ansible Fundamentals Required

### File Modules Overview

| Module | Purpose | Use Case |
|---|---|---|
| `copy` | Deploy static files | Simple file transfer |
| `template` | Deploy templates | Dynamic content |
| `fetch` | Retrieve files | Backup/analysis |
| `lineinfile` | Edit lines | Configuration updates |
| `blockinfile` | Edit blocks | Multi-line configs |
| `file` | Manage files | Permissions/ownership |
| `mount` | Mount files | Filesystem mounting |
| `archive` | Create archives | Backup files |
| `unarchive` | Extract archives | Install from tarball |

### File Transfer Methods

```
copy module: Control Node → Managed Node
fetch module: Managed Node → Control Node
```

### File Permissions

| Permission | Symbol | Description |
|---|---|---|
| Read | r | Read file content |
| Write | w | Modify file content |
| Execute | x | Execute file as script |
| Owner | u | User permissions |
| Group | g | Group permissions |
| Others | o | World permissions |

Example: `0644` = `rw-r--r--` (owner read/write, group/others read)

## Commands and Tools

```bash
# Deploy file from control node
ansible all -m copy -a "src=/tmp/config dest=/etc/config"

# Deploy template
ansible all -m template -a "src=/tmp/config.j2 dest=/etc/config"

# Fetch file from managed node
ansible all -m fetch -a "src=/etc/config dest=/tmp/"

# Check file exists
ansible all -m stat -a "path=/etc/config"

# Check file content
ansible all -m command -a "cat /etc/config"

# Check file permissions
ansible all -m command -a "ls -la /etc/config"

# Check file ownership
ansible all -m command -a "stat /etc/config"

# Deploy with lineinfile
ansible all -m lineinfile -a "path=/etc/config line='setting=value'"

# Deploy with blockinfile
ansible all -m blockinfile -a "path=/etc/config block='setting=value'"

# Create archive
ansible all -m archive -a "path=/etc/config dest=/backup/config.tar.gz"

# Extract archive
ansible all -m unarchive -a "src=/backup/config.tar.gz dest=/tmp/"
```

## Playbook Examples

### Example 1: Deploy Static Files with `copy`

```yaml
---
- name: Deploy static files using copy module
  hosts: all
  become: yes
  gather_facts: yes
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
        src: /home/ansible/files/myapp.conf
        dest: /etc/myapp/myapp.conf
        owner: root
        group: root
        mode: "0644"
        backup: yes

    - name: Deploy multiple configuration files
      copy:
        src: "{{ item.src }}"
        dest: "{{ item.dest }}"
        owner: "{{ item.owner | default('root') }}"
        group: "{{ item.group | default('root') }}"
        mode: "{{ item.mode | default('0644') }}"
        backup: yes
      loop:
        - { src: files/config1.conf, dest: /etc/myapp/config1.conf }
        - { src: files/config2.conf, dest: /etc/myapp/config2.conf }
        - { src: files/settings.ini, dest: /etc/myapp/settings.ini }

    - name: Deploy script file
      copy:
        src: /home/ansible/files/backup.sh
        dest: /usr/local/bin/backup.sh
        owner: root
        group: root
        mode: "0755"

    - name: Deploy README file
      copy:
        content: |
          # MyApp Configuration
          # Managed by Ansible
          # Last updated: {{ ansible_date_time.date }}
        dest: /etc/myapp/README.md
        owner: root
        group: root
        mode: "0644"

    - name: Verify file deployment
      stat:
        path: /etc/myapp/myapp.conf
      register: file_stat

    - name: Display deployment status
      debug:
        msg: "File {{ 'deployed' if file_stat.stat.exists else 'not found' }}"
```

### Example 2: Deploy Jinja2 Templates

```yaml
---
- name: Deploy configuration using templates
  hosts: all
  become: yes
  gather_facts: yes
  vars:
    app_name: myapp
    app_port: 8080
    app_user: myapp
    log_level: info
    max_connections: 100
    environment: production
  tasks:
    - name: Create application directory
      file:
        path: /etc/{{ app_name }}
        state: directory
        owner: root
        group: root
        mode: "0755"

    - name: Deploy main configuration template
      template:
        src: templates/{{ app_name }}/config.ini.j2
        dest: /etc/{{ app_name }}/config.ini
        owner: root
        group: root
        mode: "0644"
        backup: yes
        validate: "grep -q ^\[main\] %s"

    - name: Deploy SSH configuration template
      template:
        src: templates/sshd_config.d/{{ app_name }}.conf.j2
        dest: /etc/ssh/sshd_config.d/{{ app_name }}.conf
        owner: root
        group: root
        mode: "0600"
        validate: "sshd -t -f %s"
      notify: restart sshd

    - name: Deploy httpd configuration template
      template:
        src: templates/httpd.conf.j2
        dest: /etc/httpd/conf.d/{{ app_name }}.conf
        owner: root
        group: root
        mode: "0644"
        validate: "httpd -t -f %s"
      notify: restart httpd

    - name: Deploy systemd service template
      template:
        src: templates/systemd/{{ app_name }}.service.j2
        dest: /etc/systemd/system/{{ app_name }}.service
        owner: root
        group: root
        mode: "0644"
        validate: "systemd-analyze verify %s"
      notify: daemon reload

  handlers:
    - name: restart sshd
      systemd:
        name: sshd
        state: restarted

    - name: restart httpd
      systemd:
        name: httpd
        state: restarted

    - name: daemon reload
      systemd:
        daemon_reload: yes
```

```yaml
# Template: templates/myapp/config.ini.j2
# Managed by Ansible - Do not edit manually
# Generated: {{ ansible_date_time.iso8601 }}

[application]
name = {{ app_name }}
version = 1.0.0
port = {{ app_port }}
log_level = {{ log_level }}
max_connections = {{ max_connections }}
environment = {{ environment | default('production') }}

[logging]
log_file = /var/log/{{ app_name }}/app.log
max_size = {{ log_max_size | default('100M') }}
rotate_count = {{ log_rotate_count | default(7) }}

[security]
{% if enable_ssl | default(false) %}
ssl_enabled = true
ssl_cert = /etc/ssl/certs/{{ app_name }}.crt
ssl_key = /etc/ssl/private/{{ app_name }}.key
{% else %}
ssl_enabled = false
{% endif %}

{% if app_user | default('myapp') %}
[permissions]
user = {{ app_user }}
group = {{ app_user }}
{% endif %}
```

### Example 3: File Content Management with `lineinfile`

```yaml
---
- name: Manage file content with lineinfile
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
    - name: Create configuration file
      copy:
        content: |
          [settings]
          port = 8080
          log_level = info
        dest: /etc/myapp/config.ini
        owner: root
        group: root
        mode: "0644"

    - name: Add line if not present
      lineinfile:
        path: /etc/myapp/config.ini
        line: "timeout = 30"
        create: yes

    - name: Add line after specific pattern
      lineinfile:
        path: /etc/myapp/config.ini
        line: "max_connections = 100"
        insertafter: "^log_level"
        create: yes

    - name: Update existing line
      lineinfile:
        path: /etc/myapp/config.ini
        regexp: "^port = "
        line: "port = {{ app_port | default(8080) }}"
        state: present

    - name: Replace commented line
      lineinfile:
        path: /etc/myapp/config.ini
        regexp: "^# port = "
        line: "port = 8080"
        state: present

    - name: Remove specific line
      lineinfile:
        path: /etc/myapp/config.ini
        regexp: "^debug = "
        state: absent

    - name: Add database connection
      lineinfile:
        path: /etc/myapp/config.ini
        line: "database_host = localhost"
        insertafter: "\\[settings\\]"
        state: present
```

### Example 4: File Content Management with `blockinfile`

```yaml
---
- name: Manage file content with blockinfile
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
    - name: Create configuration file
      copy:
        content: |
          [application]
          name = myapp
        dest: /etc/myapp/config.ini
        owner: root
        group: root
        mode: "0644"

    - name: Add application block
      blockinfile:
        path: /etc/myapp/config.ini
        block: |
          [application]
          name = myapp
          version = 1.0.0
          port = {{ app_port | default(8080) }}
        marker: "# {mark} ANSIBLE MANAGED BLOCK - application"
        create: yes

    - name: Add logging block
      blockinfile:
        path: /etc/myapp/config.ini
        block: |
          [logging]
          log_file = /var/log/myapp/app.log
          log_level = {{ log_level | default('info') }}
          max_size = 100M
          rotate_count = 7
        marker: "# {mark} ANSIBLE MANAGED BLOCK - logging"
        create: yes

    - name: Add security block
      blockinfile:
        path: /etc/myapp/config.ini
        block: |
          [security]
          ssl_enabled = {{ enable_ssl | default(false) | lower }}
          ssl_cert = /etc/ssl/certs/myapp.crt
          ssl_key = /etc/ssl/private/myapp.key
        marker: "# {mark} ANSIBLE MANAGED BLOCK - security"
        create: yes

    - name: Update existing block
      blockinfile:
        path: /etc/myapp/config.ini
        block: |
          [database]
          host = localhost
          port = 5432
          name = myapp_db
        marker: "# {mark} ANSIBLE MANAGED BLOCK - database"
        insertafter: "\\[security\\]"
        state: present

    - name: Remove block
      blockinfile:
        path: /etc/myapp/config.ini
        marker: "# {mark} ANSIBLE MANAGED BLOCK - debug"
        state: absent
```

### Example 5: File Ownership and Permissions

```yaml
---
- name: Manage file ownership and permissions
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
    - name: Create application directory
      file:
        path: /var/www/{{ app_name }}
        state: directory
        owner: "{{ app_user | default('myapp') }}"
        group: "{{ app_group | default('myapp') }}"
        mode: "0755"

    - name: Create log directory
      file:
        path: /var/log/{{ app_name }}
        state: directory
        owner: "{{ app_user | default('myapp') }}"
        group: "{{ app_group | default('myapp') }}"
        mode: "0755"

    - name: Create data directory
      file:
        path: /var/lib/{{ app_name }}
        state: directory
        owner: "{{ app_user | default('myapp') }}"
        group: "{{ app_group | default('myapp') }}"
        mode: "0750"

    - name: Deploy configuration file with permissions
      copy:
        content: |
          [application]
          name = myapp
        dest: /etc/myapp/config.ini
        owner: root
        group: root
        mode: "0644"

    - name: Deploy script with execute permissions
      copy:
        content: |
          #!/bin/bash
          echo "Starting {{ app_name }}"
        dest: /usr/local/bin/start-{{ app_name }}
        owner: root
        group: root
        mode: "0755"

    - name: Deploy log file with specific permissions
      copy:
        content: "Application log file"
        dest: /var/log/{{ app_name }}/app.log
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0640"

    - name: Set sticky bit on shared directory
      file:
        path: /tmp/shared
        state: directory
        mode: "1777"

    - name: Set setgid bit on group directory
      file:
        path: /var/www/shared
        state: directory
        mode: "2775"

    - name: Verify file permissions
      stat:
        path: "{{ item }}"
      loop:
        - /var/www/{{ app_name }}
        - /var/log/{{ app_name }}
        - /etc/myapp/config.ini
      register: file_perms

    - name: Display permission results
      debug:
        var: file_perms.results
      no_log: true
```

### Example 6: Archive Deployment and Extraction

```yaml
---
- name: Deploy archives
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
    - name: Create backup directory
      file:
        path: /backup
        state: directory
        owner: root
        group: root
        mode: "0755"

    - name: Create archive of configuration
      archive:
        path:
          - /etc/myapp
          - /var/log/myapp
        dest: /backup/myapp-config-{{ ansible_date_time.date }}.tar.gz
        format: gz
        owner: root
        group: root
        mode: "0600"

    - name: Create compressed archive with specific format
      archive:
        path: /etc/myapp
        dest: /backup/myapp-config.tar.bz2
        format: bz2
        owner: root
        group: root
        mode: "0600"

    - name: Create zip archive
      archive:
        path: /etc/myapp
        dest: /backup/myapp-config.zip
        format: zip
        owner: root
        group: root
        mode: "0600"

    - name: Extract archive to destination
      unarchive:
        src: /backup/myapp-config-latest.tar.gz
        dest: /restore/
        owner: root
        group: root
        mode: "0644"
        remote_src: no

    - name: Extract remote archive
      unarchive:
        src: http://example.com/packages/app.tar.gz
        dest: /opt/
        remote_src: yes
        owner: root
        group: root
        mode: "0755"

    - name: Extract with specific options
      unarchive:
        src: /backup/myapp-config.tar.gz
        dest: /restore/
        owner: root
        group: root
        mode: "0644"
        remote_src: no
        copy: yes
        extra_opts:
          - --strip-components=1

    - name: Create and extract nested archive
      archive:
        path:
          - /etc/myapp/config.ini
          - /etc/myapp/data/
        dest: /backup/myapp-nested.tar.gz
        format: gz
        owner: root
        group: root
        mode: "0600"

      unarchive:
        src: /backup/myapp-nested.tar.gz
        dest: /restore/
        remote_src: no
        owner: root
        group: root
        mode: "0644"
```

### Example 7: Complete File Deployment

```yaml
---
- name: Complete file deployment
  hosts: all
  become: yes
  gather_facts: yes
  vars:
    app_name: myapp
    app_user: myapp
    app_group: myapp
    app_port: 8080
    log_level: info
  tasks:
    # Create directories
    - name: Create application directory
      file:
        path: /etc/{{ app_name }}
        state: directory
        owner: root
        group: root
        mode: "0755"

    - name: Create web directory
      file:
        path: /var/www/{{ app_name }}
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0755"

    - name: Create log directory
      file:
        path: /var/log/{{ app_name }}
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0755"

    - name: Create data directory
      file:
        path: /var/lib/{{ app_name }}
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0750"

    # Deploy configuration files
    - name: Deploy main configuration
      template:
        src: templates/{{ app_name }}/config.ini.j2
        dest: /etc/{{ app_name }}/config.ini
        owner: root
        group: root
        mode: "0644"
        validate: "grep -q ^\[main\] %s"

    - name: Deploy web configuration
      template:
        src: templates/{{ app_name }}/web.conf.j2
        dest: /etc/httpd/conf.d/{{ app_name }}.conf
        owner: root
        group: root
        mode: "0644"
        validate: "httpd -t -f %s"
      notify: restart httpd

    - name: Deploy systemd service
      template:
        src: templates/systemd/{{ app_name }}.service.j2
        dest: /etc/systemd/system/{{ app_name }}.service
        owner: root
        group: root
        mode: "0644"
        validate: "systemd-analyze verify %s"
      notify: daemon reload

    - name: Deploy backup script
      copy:
        content: |
          #!/bin/bash
          # Backup script for {{ app_name }}
          set -euo pipefail

          BACKUP_DIR="/backup/{{ app_name }}/{{ ansible_date_time.date }}"
          mkdir -p "$BACKUP_DIR"

          tar czf "$BACKUP_DIR/config.tar.gz" /etc/{{ app_name }}

          echo "Backup completed: $BACKUP_DIR"
        dest: /usr/local/bin/backup-{{ app_name }}
        owner: root
        group: root
        mode: "0755"

    - name: Deploy startup script
      copy:
        content: |
          #!/bin/bash
          # Startup script for {{ app_name }}
          systemctl start {{ app_name }}
        dest: /usr/local/bin/start-{{ app_name }}
        owner: root
        group: root
        mode: "0755"

    - name: Deploy stop script
      copy:
        content: |
          #!/bin/bash
          # Stop script for {{ app_name }}
          systemctl stop {{ app_name }}
        dest: /usr/local/bin/stop-{{ app_name }}
        owner: root
        group: root
        mode: "0755"

    - name: Create application user
      user:
        name: "{{ app_user }}"
        system: yes
        shell: /sbin/nologin
        home: /var/lib/{{ app_name }}
        createhome: no

    - name: Create application group
      group:
        name: "{{ app_group }}"
        system: yes

    - name: Set ownership on web directory
      file:
        path: /var/www/{{ app_name }}
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0755"

    - name: Set ownership on log directory
      file:
        path: /var/log/{{ app_name }}
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0755"

    - name: Set ownership on data directory
      file:
        path: /var/lib/{{ app_name }}
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0750"

    - name: Create log file
      file:
        path: /var/log/{{ app_name }}/app.log
        state: touch
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0640"

  handlers:
    - name: restart httpd
      systemd:
        name: httpd
        state: restarted

    - name: daemon reload
      systemd:
        daemon_reload: yes
```

## Explanation of Each Playbook

### Playbook 1: Static File Deployment

The `copy` module transfers files from the control node to managed nodes. The `src` parameter specifies the source file path on the control node. The `dest` parameter specifies the destination path on the managed node. File ownership and permissions are set using `owner`, `group`, and `mode` parameters. The `backup: yes` parameter creates a backup of the existing file before overwriting.

### Playbook 2: Jinja2 Template Deployment

The `template` module processes Jinja2 templates with variable substitution. The `src` parameter points to the template file on the control node. Variables defined in the playbook are substituted into the template. The `validate` parameter runs a command to check file syntax before writing. Handlers notify the playbook to restart services after configuration changes.

### Playbook 3: Line Management

The `lineinfile` module manages individual lines in files. It can add new lines, modify existing lines using regex, or remove lines. The `regexp` parameter matches existing lines to replace them. The `insertafter` parameter adds lines after a specific pattern. The `create: yes` parameter creates the file if it doesn't exist.

### Playbook 4: Block Management

The `blockinfile` module manages multi-line blocks in files. The `block` parameter contains the content to add or update. The `marker` parameter defines marker comments that track managed content. This ensures idempotency by detecting if a block already exists. The `insertafter` parameter adds blocks after specific patterns.

### Playbook 5: File Permissions

The `file` module manages file and directory attributes. The `state: directory` parameter creates directories. The `owner`, `group`, and `mode` parameters set ownership and permissions. Special bits like sticky bit (1) and setgid (2) can be set in the mode. The `stat` module verifies file permissions after deployment.

### Playbook 6: Archive Management

The `archive` module creates compressed archives from files or directories. The `format` parameter specifies the compression format (gz, bz2, xz, zip). The `unarchive` module extracts archives. Setting `remote_src: yes` tells Ansible the archive already exists on the managed node.

### Playbook 7: Complete Deployment

This comprehensive playbook demonstrates a complete file deployment workflow: creating directories, deploying configuration files with templates, deploying scripts, managing user/group accounts, and setting permissions. It shows how multiple modules work together to configure a complete application environment.

## Variables and Templates

### Using Variables in File Deployment

```yaml
---
- name: File deployment with variables
  hosts: all
  become: yes
  vars:
    app_name: myapp
    app_port: 8080
    log_level: info
  tasks:
    - name: Create directory
      file:
        path: /etc/{{ app_name }}
        state: directory
        mode: "0755"

    - name: Deploy configuration
      template:
        src: templates/{{ app_name }}/config.ini.j2
        dest: /etc/{{ app_name }}/config.ini
```

### Advanced Jinja2 Template Features

```yaml
# Template: advanced_template.j2
# Variable substitution
{{ app_name }}

# Variable defaults
{{ app_port | default(8080) }}

# Variable filtering
{{ app_name | upper }}
{{ app_name | lower }}
{{ app_name | capitalize }}

# Conditional blocks
{% if enable_ssl | default(false) %}
ssl_enabled = true
{% else %}
ssl_enabled = false
{% endif %}

# Loops
{% for service in services %}
[{{ service }}]
enabled = true
{% endfor %}

# Include other templates
{% include 'common.conf' %}

# Include with variables
{% include 'app.conf' with context %}

# Environment variables
{% set env_var = env | default({}) %}
{{ env_var | default('production') }}

# Date and time
{{ ansible_date_time.date }}
{{ ansible_date_time.iso8601 }}
{{ ansible_date_time.epoch }}

# Facts
{{ ansible_distribution }}
{{ ansible_facts.services.httpd.service }}

# Jinja2 filters
{{ items | join(', ') }}
{{ items | length }}
{{ items | sort }}
{{ items | map(attribute='name') | list }}
```

```yaml
# Playbook using advanced template
---
- name: Advanced template deployment
  hosts: all
  become: yes
  vars:
    services:
      - httpd
      - firewalld
      - sshd
    environments:
      - production
      - staging
      - development
  tasks:
    - name: Deploy configuration
      template:
        src: templates/app.conf.j2
        dest: /etc/app.conf
```

### Using Facts in File Deployment

```yaml
---
- name: File deployment with facts
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
    - name: Deploy OS-specific configuration
      template:
        src: templates/config.{{ ansible_os_family }}.j2
        dest: /etc/config
      when: ansible_os_family is defined

    - name: Deploy architecture-specific configuration
      template:
        src: templates/config.{{ ansible_architecture }}.j2
        dest: /etc/config
      when: ansible_architecture is defined

    - name: Deploy version-specific configuration
      template:
        src: templates/config.{{ ansible_distribution_major_version }}.j2
        dest: /etc/config
```

## Verification Procedures

### Verify File Deployment

```bash
# Check file exists
ansible all -m stat -a "path=/etc/myapp/config.ini"

# Check file content
ansible all -m command -a "cat /etc/myapp/config.ini"

# Check file permissions
ansible all -m command -a "ls -la /etc/myapp/config.ini"

# Check file ownership
ansible all -m command -a "stat /etc/myapp/config.ini"

# Check file size
ansible all -m command -a "du -h /etc/myapp/config.ini"

# Check file modification time
ansible all -m command -a "ls -l /etc/myapp/config.ini"
```

### Verify Template Variables

```bash
# Display deployed template content
ansible all -m command -a "cat /etc/myapp/config.ini"

# Check specific variables
ansible all -m command -a "grep port /etc/myapp/config.ini"
ansible all -m command -a "grep log_level /etc/myapp/config.ini"
```

### Verify File Permissions

```bash
# Check file permissions
ansible all -m command -a "stat -c '%a %U %G %n' /etc/myapp/*"

# Verify ownership
ansible all -m command -a "ls -ln /etc/myapp/"

# Check directory permissions
ansible all -m command -a "ls -ld /var/www/myapp /var/log/myapp"
```

### Verify Archive Deployment

```bash
# List archive contents
ansible all -m command -a "tar -tzf /backup/myapp-config.tar.gz"

# Check archive size
ansible all -m command -a "ls -lh /backup/myapp-config.tar.gz"

# Verify archive integrity
ansible all -m command -a "tar -tzf /backup/myapp-config.tar.gz"
```

## Troubleshooting

### File Not Found on Managed Node

**Problem: Deployed file doesn't exist**

```bash
# Solution: Check if playbook ran successfully
ansible-playbook playbook.yml -vvv

# Solution: Check file path
ansible all -m stat -a "path=/etc/myapp/config.ini"

# Solution: Verify src file exists on control node
ls -la /home/ansible/files/myapp.conf
```

**Ansible playbook:**
```yaml
- name: Verify source file exists
  stat:
    path: /home/ansible/files/myapp.conf
  register: source_stat

- name: Deploy file
  copy:
    src: /home/ansible/files/myapp.conf
    dest: /etc/myapp/myapp.conf
  when: source_stat.stat.exists
```

### Permission Denied

**Problem: Cannot write to destination**

```bash
# Solution: Check destination directory permissions
ansible all -m command -a "ls -ld /etc/myapp"

# Solution: Ensure become is enabled
- name: Deploy file
  become: yes
  copy:
    src: /tmp/config
    dest: /etc/config
```

**Ansible playbook:**
```yaml
- name: Create directory with correct permissions
  file:
    path: /etc/myapp
    state: directory
    owner: root
    group: root
    mode: "0755"

- name: Deploy file
  copy:
    src: /tmp/config
    dest: /etc/myapp/config
    owner: root
    group: root
    mode: "0644"
```

### Template Variable Not Substituted

**Problem: Variables not replaced in template**

```bash
# Solution: Check variable definition
ansible-playbook playbook.yml -e "app_port=8080"

# Solution: Verify template syntax
cat templates/config.j2

# Solution: Check variable name matches
# Template: {{ app_port }}
# Playbook: app_port: 8080
```

**Ansible playbook:**
```yaml
- name: Check template rendering
  template:
    src: templates/config.j2
    dest: /tmp/test-config
    validate: "cat %s"
  register: template_result

- name: Display rendered content
  debug:
    var: template_result
```

### Invalid Syntax After Deployment

**Problem: Deployed file has syntax errors**

```bash
# Solution: Validate before deploying
- name: Validate configuration
  command: httpd -t -f /etc/httpd/conf.d/myapp.conf
  register: httpd_test
  changed_when: false

- name: Deploy configuration
  template:
    src: templates/httpd.conf.j2
    dest: /etc/httpd/conf.d/myapp.conf
  when: httpd_test.rc == 0
  notify: restart httpd
```

### File Not Overwritten

**Problem: File not updated despite changes**

```bash
# Solution: Check checksum
ansible all -m command -a "md5sum /etc/myapp/config.ini"

# Solution: Force update with force=yes
- name: Force file update
  copy:
    content: "new content"
    dest: /etc/myapp/config.ini
    force: yes
```

## Real-World Automation Examples

### Complete Web Server File Deployment

```yaml
---
- name: Complete web server file deployment
  hosts: all
  become: yes
  gather_facts: yes
  vars:
    app_name: myapp
    app_port: 8080
    app_user: myapp
    app_group: myapp
  tasks:
    # Create directories
    - name: Create application configuration directory
      file:
        path: /etc/{{ app_name }}
        state: directory
        owner: root
        group: root
        mode: "0755"

    - name: Create web content directory
      file:
        path: /var/www/{{ app_name }}
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0755"

    - name: Create log directory
      file:
        path: /var/log/{{ app_name }}
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0755"

    - name: Create data directory
      file:
        path: /var/lib/{{ app_name }}
        state: directory
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0750"

    # Deploy configuration files
    - name: Deploy main application configuration
      template:
        src: templates/{{ app_name }}/config.ini.j2
        dest: /etc/{{ app_name }}/config.ini
        owner: root
        group: root
        mode: "0644"
        validate: "grep -q ^\[main\] %s"

    - name: Deploy httpd configuration
      template:
        src: templates/httpd/{{ app_name }}.conf.j2
        dest: /etc/httpd/conf.d/{{ app_name }}.conf
        owner: root
        group: root
        mode: "0644"
        validate: "httpd -t -f %s"
      notify: restart httpd

    - name: Deploy systemd service
      template:
        src: templates/systemd/{{ app_name }}.service.j2
        dest: /etc/systemd/system/{{ app_name }}.service
        owner: root
        group: root
        mode: "0644"
        validate: "systemd-analyze verify %s"
      notify: daemon reload

    # Deploy scripts
    - name: Deploy startup script
      copy:
        content: |
          #!/bin/bash
          echo "Starting {{ app_name }}"
          systemctl start {{ app_name }}
        dest: /usr/local/bin/start-{{ app_name }}
        owner: root
        group: root
        mode: "0755"

    - name: Deploy backup script
      copy:
        content: |
          #!/bin/bash
          # Backup script for {{ app_name }}
          BACKUP_DIR="/backup/{{ app_name }}/{{ ansible_date_time.date }}"
          mkdir -p "$BACKUP_DIR"
          tar czf "$BACKUP_DIR/config.tar.gz" /etc/{{ app_name }}
        dest: /usr/local/bin/backup-{{ app_name }}
        owner: root
        group: root
        mode: "0755"

    # Create user and group
    - name: Create application user
      user:
        name: "{{ app_user }}"
        system: yes
        shell: /sbin/nologin
        home: /var/lib/{{ app_name }}

    - name: Create application group
      group:
        name: "{{ app_group }}"
        system: yes

    # Set permissions
    - name: Set ownership on web directory
      file:
        path: /var/www/{{ app_name }}
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0755"

    - name: Set ownership on log directory
      file:
        path: /var/log/{{ app_name }}
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0755"

  handlers:
    - name: restart httpd
      systemd:
        name: httpd
        state: restarted

    - name: daemon reload
      systemd:
        daemon_reload: yes
```

## RHCE Exam Notes

### Critical Exam Points

1. **Template validation is critical**: Always use `validate` parameter with `template` module to check syntax before writing.

2. **File permissions must be correct**: Configuration files typically need `0644`, scripts need `0755`, sensitive files need `0600` or `0640`.

3. **Use `backup: yes` for important files**: This creates backups that can be recovered if deployment fails.

4. **Handlers are essential**: Services must be restarted after configuration file changes. Use `notify` with handlers.

5. **Template variables must match**: Variable names in templates must exactly match playbook variable names.

6. **Use `become: yes` for file deployment**: Most file deployments require root privileges.

7. **Verify with `stat` module**: Check file existence and permissions after deployment.

8. **Archive deployment uses `remote_src: yes`**: When extracting files already on the managed node.

### Exam Workflow

1. Create directories first
2. Deploy templates with validation
3. Set file permissions
4. Create users/groups if needed
5. Test with `stat` module
6. Notify handlers to restart services

### Common Exam Scenarios

- **Deploy configuration file**: Use `template` module with `validate`
- **Deploy script**: Use `copy` module with `mode: "0755"`
- **Deploy with variables**: Define vars in playbook, use in template
- **Restart service**: Use `notify` handler in playbook
- **Check deployment**: Use `stat` module to verify

## Common Mistakes

### Mistake 1: Not Using `validate` with Templates

```yaml
# WRONG - may deploy invalid configuration
- name: Deploy configuration
  template:
    src: templates/config.j2
    dest: /etc/config

# CORRECT - validate before deploying
- name: Deploy configuration
  template:
    src: templates/config.j2
    dest: /etc/config
    validate: "httpd -t -f %s"
```

### Mistake 2: Wrong File Permissions

```yaml
# WRONG - script not executable
- name: Deploy script
  copy:
    content: "#!/bin/bash\n..."
    dest: /usr/local/bin/script.sh
    mode: "0644"

# CORRECT - script needs execute permission
- name: Deploy script
  copy:
    content: "#!/bin/bash\n..."
    dest: /usr/local/bin/script.sh
    mode: "0755"
```

### Mistake 3: Not Creating Directory First

```yaml
# WRONG - file deployment fails
- name: Deploy file
  copy:
    content: "config"
    dest: /etc/newdir/config

# CORRECT - create directory first
- name: Create directory
  file:
    path: /etc/newdir
    state: directory

- name: Deploy file
  copy:
    content: "config"
    dest: /etc/newdir/config
```

### Mistake 4: Variable Name Mismatch

```yaml
# Template uses: {{ app_port }}
# Playbook defines: http_port: 8080

# WRONG - variable not substituted
# Template shows: {{ app_port }}

# CORRECT - use correct variable name
- name: Deploy
  template:
    src: templates/config.j2
    dest: /etc/config
  vars:
    app_port: 8080
```

### Mistake 5: Not Using Handlers

```yaml
# WRONG - service doesn't reload config
- name: Deploy configuration
  template:
    src: templates/httpd.conf.j2
    dest: /etc/httpd/conf.d/myapp.conf

# CORRECT - notify handler to restart
- name: Deploy configuration
  template:
    src: templates/httpd.conf.j2
    dest: /etc/httpd/conf.d/myapp.conf
  notify: restart httpd

handlers:
  - name: restart httpd
    systemd:
      name: httpd
      state: restarted
```

## Chapter Summary

This chapter covered file deployment to managed nodes:

- **`copy` module** deploys static files from the control node with variable content.
- **`template` module** processes Jinja2 templates with variable substitution for dynamic content.
- **`lineinfile` module** manages individual lines in configuration files.
- **`blockinfile` module** manages multi-line blocks with marker comments for idempotency.
- **`file` module** manages ownership, permissions, and special bits.
- **`archive` and `unarchive` modules** create and extract compressed archives.
- **Verification procedures** check file existence, content, permissions, and ownership.
- **Troubleshooting** addresses common issues: files not found, permission denied, variable substitution problems, and syntax errors.
- **Real-world examples** demonstrate complete application deployment workflows.

## Quick Reference

| Module | Purpose | Key Parameters |
|---|---|---|
| `copy` | Deploy static files | `src`, `dest`, `owner`, `group`, `mode` |
| `template` | Deploy templates | `src`, `dest`, `validate`, `owner`, `group`, `mode` |
| `lineinfile` | Edit lines | `path`, `line`, `regexp`, `insertafter` |
| `blockinfile` | Edit blocks | `path`, `block`, `marker`, `insertafter` |
| `file` | Manage files | `path`, `state`, `owner`, `group`, `mode` |
| `archive` | Create archives | `path`, `dest`, `format` |
| `unarchive` | Extract archives | `src`, `dest`, `remote_src` |
| `stat` | Check files | `path` |
| `command` | Read files | `path`, `cat` |

## Review Questions

1. What is the difference between the `copy` module and the `template` module?

2. How does the `validate` parameter work with the `template` module, and why is it important?

3. What is the purpose of the `marker` parameter in the `blockinfile` module?

4. How do you create a backup of an existing file before overwriting it with Ansible?

5. What is the difference between `remote_src: yes` and `remote_src: no` in the `unarchive` module?

6. How would you deploy a script that needs execute permissions?

7. What is the purpose of the `insertafter` parameter in `lineinfile` and `blockinfile`?

8. How do you use Jinja2 filters in templates, and can you give three examples?

9. What handlers should be used after deploying configuration files for services?

10. How do you verify that a file was deployed correctly with the correct permissions?

## Answers

1. The `copy` module deploys static files from the control node to managed nodes without processing. The `template` module deploys Jinja2 templates that are processed with variable substitution before writing to the managed node. Use `copy` for static content and `template` for dynamic content that varies per host or depends on variables and facts.

2. The `validate` parameter runs a command on the managed node to check the syntax or validity of a file before it is deployed. The `%s` placeholder is replaced with the path to a temporary file containing the new content. This is important because it prevents invalid configurations from being deployed, which could cause service failures. For example, `validate: "httpd -t -f %s"` validates Apache configuration syntax.

3. The `marker` parameter defines a marker comment used by the `blockinfile` module to track managed content. The marker includes `{mark}` which is replaced with a unique marker string (like `# ANSIBLE MANAGED BLOCK`). This allows Ansible to detect if a block already exists and skip adding it again, ensuring idempotency.

4. Use the `backup: yes` parameter in the `copy` or `template` module. This creates a backup of the existing file before overwriting it. The backup is stored in the same directory as the original file with a `.rpmnew` or `.bak` extension. For example: `copy: { dest: /etc/config, backup: yes }`.

5. `remote_src: no` (default) tells Ansible to transfer the source file from the control node to the managed node before extracting. `remote_src: yes` tells Ansible the source archive already exists on the managed node and should be extracted directly without transferring. Use `remote_src: yes` for archives already on the managed node or downloaded files.

6. Use the `copy` module with `mode: "0755"` to set execute permissions. For example: `copy: { content: "#!/bin/bash\n...", dest: /usr/local/bin/script.sh, mode: "0755" }`. The file content must start with a shebang line (`#!/bin/bash`) for it to be executable.

7. The `insertafter` parameter adds lines or blocks after a specific pattern or line in the file. In `lineinfile`, it inserts the new line after the matching pattern. In `blockinfile`, it inserts the block after the matching pattern. Use regex patterns like `insertafter: "^pattern"` or `insertafter: "\\[section\\]"`.

8. Jinja2 filters transform variable values. Examples: `{{ items | join(', ') }}` joins a list with commas, `{{ items | length }}` returns the list length, `{{ items | sort }}` sorts a list, `{{ items | upper }}` converts to uppercase, `{{ items | default('default_value') }}` provides a default if undefined.

9. After deploying configuration files for services, use handlers to restart the services. For httpd: `notify: restart httpd` with `systemd: { name: httpd, state: restarted }`. For systemd services: `notify: daemon reload` with `systemd: { daemon_reload: yes }`. Handlers run once at the end of the play regardless of how many tasks notify them.

10. Use the `stat` module to verify file deployment: `stat: { path: /etc/config }`. Register the result and check `stat.stat.exists` for existence, and `stat.stat.mode` or `stat.stat.group` for permissions. You can also use `command: { name: ls -la /etc/config }` to check permissions manually.
