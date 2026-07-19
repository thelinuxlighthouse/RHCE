# Chapter 4: Configure Ansible Managed Nodes


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


## Added Review: Managed Node Readiness Checklist

A managed node is ready when all of this is true:

```text
DNS or inventory name resolves
SSH service is reachable
remote_user can log in
Python required by the execution environment is available
remote_user can become root when needed
firewall allows SSH
SELinux is not blocking the required service behavior
```

Verify with increasing depth:

```bash
ansible all -m ansible.builtin.ping
ansible all -m ansible.builtin.command -a 'whoami'
ansible all -b -m ansible.builtin.command -a 'whoami'
ansible all -m ansible.builtin.setup -a 'filter=ansible_distribution*'
```

Do not start troubleshooting playbook logic until the basic connection path works.


## Learning Objectives

By the end of this chapter, you will be able to:

- Configure Ansible managed nodes for automation readiness
- Install and configure required software on managed nodes
- Create and distribute SSH keys to managed nodes
- Configure privilege escalation on managed nodes
- Deploy files to managed nodes using various methods
- Manage software packages and repositories
- Configure and manage system services
- Configure firewall rules on managed nodes
- Manage file systems and storage devices
- Manage file content and deployment
- Manage archives and backups
- Configure task scheduling
- Configure security settings including SELinux
- Manage users and groups on managed nodes

## Concept Overview

Ansible managed nodes are the target systems that Ansible configures and manages. Before playbooks can successfully configure a node, the node must be prepared with the necessary infrastructure: SSH access, Python runtime, sudo privileges, and system services.

### Why Managed Node Configuration Matters

Without proper managed node configuration, Ansible automation fails:

- No SSH access = no connectivity to managed nodes
- No sudo privileges = no privilege escalation for root tasks
- No Python = modules cannot execute on managed nodes
- Missing services = automation cannot manage services that don't exist
- Missing packages = dependencies fail to install

### How Ansible Configures Managed Nodes

Ansible manages nodes by:

1. Connecting via SSH to the managed node
2. Executing modules with the Ansible user
3. Escalating to root using sudo when required
4. Receiving structured JSON output from modules
5. Converging the managed node to the desired state

## Ansible Fundamentals Required

### Managed Node Prerequisites

Each managed node requires:

| Component | Requirement | Purpose |
|---|---|---|
| SSH Server | Running and accessible | Ansible uses SSH for communication |
| Python 3 | Installed (3.8+) | Ansible modules are Python-based |
| Sudo Access | Configured for Ansible user | Execute root-level tasks |
| Firewall | SSH port open | Allow Ansible connections |
| SELinux | Configurable | May need to be configured for services |

### Ansible Module Categories for Managed Nodes

| Category | Modules | Purpose |
|---|---|---|
| Packages | `ansible.builtin.dnf` / verified `ansible.builtin.dnf5`, and only environment-verified module-stream methods | Install and manage software |
| Services | `systemd` | Start, stop, enable services |
| Firewall | `firewalld` | Configure firewall rules |
| Files | `copy`, `template` | Deploy files and configurations |
| Users/Groups | `user`, `group` | Manage system users and groups |
| Storage | `mount`, `lvg`, `lvol` | Configure storage and filesystems |
| Security | `selinux`, `seboolean` | Configure SELinux policies |
| Scheduling | `ansible.builtin.cron` and verified systemd timer unit management | Schedule recurring tasks |

## Commands and Tools

```bash
# Test managed node connectivity
ansible all -m ping
ansible all -m ping -vvv

# Check Python version on managed nodes
ansible all -m command -a "python3 --version"

# Check installed packages
ansible all -m dnf -a "list_installed"
ansible all -m dnf -a "list | grep httpd"

# Check service status
ansible all -m command -a "systemctl is-active firewalld"

# Check installed facts
ansible all -m setup -a "filter=ansible_facts.services"

# Test privilege escalation
ansible all -m ping -b
ansible all -m command -a "whoami" -b

# Check SELinux status
ansible all -m command -a "getenforce"

# Check mounted filesystems
ansible all -m command -a "df -h"

# Check cron jobs
ansible all -m command -a "crontab -l"

# Install packages ad hoc
ansible all -m dnf -a "name=httpd state=present"

# Start service ad hoc
ansible all -m systemd -a "name=firewalld state=started enabled=true"
```

## Playbook Examples

### Example 1: Software Package and Repository Management

```yaml
---
- name: Configure software packages and repositories
  hosts: all
  become: true
  gather_facts: yes
  vars:
    common_packages:
      - vim-enhanced
      - net-tools
      - wget
      - curl
      - tree
      - httpd
      - firewalld
      - sudo
    optional_packages:
      - python3-libselinux
      - python3-firewall
      - python3-cryptography
  tasks:
    - name: Update package cache
      dnf:
        name: "*"
        state: present
        update_cache: yes

    - name: Install common packages
      dnf:
        name: "{{ item }}"
        state: present
      loop: "{{ common_packages }}"
      notify: reboot if required

    - name: Install optional Python packages
      dnf:
        name: "{{ item }}"
        state: present
      loop: "{{ optional_packages }}"
      when: ansible_os_family == "RedHat"

    - name: Install latest version of firewalld
      dnf:
        name: firewalld
        state: latest

    - name: Install DNF module for PostgreSQL 16
      ansible.builtin.command: dnf module list postgresql
      register: postgresql_module_streams
      changed_when: false
      failed_when: false

    - name: Install PostgreSQL server
      dnf:
        name: postgresql-server
        state: present

    - name: Install and enable crontabs
      dnf:
        name: crontabs
        state: present

    - name: Install rsync for file synchronization
      dnf:
        name: rsync
        state: present

    - name: Install git for version control
      dnf:
        name: git
        state: present

    - name: Configure custom repository
      copy:
        content: |
          [myrepo]
          name=My Custom Repository
          baseurl=http://repo.example.com/custom
          gpgcheck=0
          enabled=1
        dest: /etc/yum.repos.d/myrepo.repo
        owner: root
        group: root
        mode: "0644"
        validate: "ls -la %s"
```

### Example 2: Service Management

```yaml
---
- name: Configure system services
  hosts: all
  become: true
  gather_facts: yes
  vars:
    required_services:
      - name: firewalld
        enabled: yes
      - name: sshd
        enabled: yes
      - name: chronyd
        enabled: yes
    optional_services:
      - name: httpd
        enabled: yes
      - name: postfix
        enabled: yes
  tasks:
    - name: Ensure firewalld is running and enabled
      systemd:
        name: firewalld
        state: started
        enabled: yes

    - name: Ensure SSH service is running and enabled
      systemd:
        name: sshd
        state: started
        enabled: yes

    - name: Ensure NTP service is running and enabled
      systemd:
        name: chronyd
        state: started
        enabled: yes

    - name: Start httpd service if package is installed
      systemd:
        name: httpd
        state: started
        enabled: yes
      when:
        - ansible_facts.packages.httpd is defined
        - ansible_facts.packages.httpd.state == "present"

    - name: Ensure postfix is stopped and disabled
      systemd:
        name: postfix
        state: stopped
        enabled: no
      when:
        - ansible_facts.packages.postfix is defined
        - ansible_facts.packages.postfix.state == "present"

    - name: Reload systemd daemon
      systemd:
        daemon_reload: yes

    - name: Restart services that need configuration reload
      systemd:
        name: "{{ item }}"
        state: reloaded
      loop:
        - httpd
        - firewalld
      when:
        - ansible_facts.services | select("attr", "name", "match", item) | list | length > 0
      loop_control:
        label: "{{ item }}"

    - name: Create systemd timer for backup
      systemd:
        name: backup.timer
        daemon_reload: yes
      when:
        - ansible_facts.services.backup.service is defined
```

### Example 3: Firewall Configuration

```yaml
---
- name: Configure firewall rules
  hosts: all
  become: true
  gather_facts: yes
  tasks:
    - name: Ensure firewalld is running
      systemd:
        name: firewalld
        state: started
        enabled: yes

    - name: Open HTTP port for web traffic
      firewalld:
        port: "80/tcp"
        permanent: yes
        immediate: yes
        state: enabled

    - name: Open HTTPS port for secure web traffic
      firewalld:
        port: "443/tcp"
        permanent: yes
        immediate: yes
        state: enabled

    - name: Open custom application port
      firewalld:
        port: "8080/tcp"
        permanent: yes
        immediate: yes
        state: enabled

    - name: Open port range for multiple services
      firewalld:
        port: "9000-9010/tcp"
        permanent: yes
        immediate: yes
        state: enabled

    - name: Allow SSH service
      firewalld:
        service: ssh
        permanent: yes
        immediate: yes
        state: enabled

    - name: Allow NTP service
      firewalld:
        service: ntp
        permanent: yes
        immediate: yes
        state: enabled

    - name: Set default zone to public
      firewalld:
        zone: public
        default: yes
        permanent: yes

    - name: Configure port forwarding
      firewalld:
        port: "8443/tcp"
        toport: "443/tcp"
        toaddr: ""
        permanent: yes
        immediate: yes
        state: enabled

    - name: Block insecure FTP service
      firewalld:
        service: ftp
        permanent: yes
        immediate: yes
        state: disabled

    - name: Remove unused services
      firewalld:
        service: "{{ item }}"
        permanent: yes
        immediate: yes
        state: disabled
      loop:
        - dhcpv6-client
        - avahi-autoipd
```

### Example 4: File System and Storage Management

```yaml
---
- name: Configure file systems and storage
  hosts: all
  become: true
  gather_facts: yes
  vars:
    storage_devices:
      - device: /dev/vdb
        vg_name: vg_data
        lv_name: lv_storage
        mount_point: /mnt/data
        fstype: xfs
        size: 10G
    tmpfs_mounts:
      - path: /tmp/shared
        fstype: tmpfs
        size: 1G
  tasks:
    - name: Check if storage device exists
      command: lsblk
      register: lsblk_output
      changed_when: false
      failed_when: false

    - name: Create volume group
      lvg:
        vg: "{{ item.vg_name }}"
        pvs: "{{ item.device }}"
      loop: "{{ storage_devices }}"
      when:
        - item.device is defined
        - ansible_devices | select("test", item.device | regex_replace('/dev/', '')) | list | length > 0

    - name: Create logical volume
      lvol:
        vg: "{{ item.vg_name }}"
        lv: "{{ item.lv_name }}"
        size: "{{ item.size }}"
      loop: "{{ storage_devices }}"
      when:
        - item.vg_name is defined
        - item.lv_name is defined

    - name: Create filesystem on logical volume
      filesystem:
        dev: "/dev/{{ item.vg_name }}/{{ item.lv_name }}"
        fstype: "{{ item.fstype }}"
      loop: "{{ storage_devices }}"
      when:
        - item.vg_name is defined
        - item.lv_name is defined
        - item.fstype is defined

    - name: Create mount point directory
      file:
        path: "{{ item.mount_point }}"
        state: directory
        owner: root
        group: root
        mode: "0755"
      loop: "{{ storage_devices }}"
      when:
        - item.mount_point is defined

    - name: Mount filesystem persistently
      mount:
        path: "{{ item.mount_point }}"
        src: "/dev/{{ item.vg_name }}/{{ item.lv_name }}"
        fstype: "{{ item.fstype }}"
        state: mounted
        opts: defaults
        dump: yes
        passno: 2
      loop: "{{ storage_devices }}"
      when:
        - item.mount_point is defined
        - item.fstype is defined

    - name: Configure tmpfs mount
      mount:
        path: "{{ item.path }}"
        src: tmpfs
        fstype: tmpfs
        state: mounted
        opts: "size={{ item.size }},mode=1777"
      loop: "{{ tmpfs_mounts }}"

    - name: Ensure /etc/fstab has correct entries
      lineinfile:
        path: /etc/fstab
        line: "{{ item }}"
        create: yes
        validate: "mount -a"
      loop:
        - "{{ storage_devices[0].mount_point }} /dev/{{ storage_devices[0].vg_name }}/{{ storage_devices[0].lv_name }} {{ storage_devices[0].fstype }} defaults 0 2"
```

### Example 5: File Content Management

```yaml
---
- name: Manage file content
  hosts: all
  become: true
  gather_facts: yes
  tasks:
    - name: Create application directory
      file:
        path: /etc/myapp
        state: directory
        owner: root
        group: root
        mode: "0755"

    - name: Deploy main configuration file
      copy:
        content: |
          [application]
          name = myapp
          version = 1.0.0
          port = 8080
          log_level = info
        dest: /etc/myapp/config.ini
        owner: root
        group: root
        mode: "0644"

    - name: Add database connection if not present
      lineinfile:
        path: /etc/myapp/config.ini
        line: "database_host = localhost"
        insertafter: "\\[application\\]"
        state: present

    - name: Update port configuration
      lineinfile:
        path: /etc/myapp/config.ini
        regexp: "^port = "
        line: "port = {{ app_port | default(8080) }}"
        state: present

    - name: Add application-specific block
      blockinfile:
        path: /etc/myapp/config.ini
        block: |
          [logging]
          log_file = /var/log/myapp/app.log
          max_size = 100M
          rotate_count = 7
        marker: "# {mark} ANSIBLE MANAGED BLOCK"
        state: present

    - name: Remove commented debug line
      lineinfile:
        path: /etc/myapp/config.ini
        regexp: "^# debug = "
        state: absent

    - name: Deploy SSH configuration with validation
      copy:
        content: |
          Port 22
          PermitRootLogin no
          PasswordAuthentication no
          X11Forwarding no
          AllowUsers ansible admin
        dest: /etc/ssh/sshd_config.d/myapp.conf
        owner: root
        group: root
        mode: "0600"
        validate: "sshd -t -f %s"

    - name: Deploy httpd configuration from template
      template:
        src: templates/httpd.conf.j2
        dest: /etc/httpd/conf.d/myapp.conf
        owner: root
        group: root
        mode: "0644"
        validate: "httpd -t -f %s"
      notify: restart httpd

    - name: Create and deploy backup script
      copy:
        content: |
          #!/bin/bash
          # Backup script for myapp
          set -euo pipefail

          BACKUP_DIR="/backup/myapp/{{ ansible_date_time.date }}"
          mkdir -p "$BACKUP_DIR"

          tar czf "$BACKUP_DIR/config.tar.gz" /etc/myapp

          echo "Backup completed: $BACKUP_DIR"
        dest: /usr/local/bin/myapp-backup.sh
        owner: root
        group: root
        mode: "0755"

  handlers:
    - name: restart httpd
      systemd:
        name: httpd
        state: restarted
```

### Example 6: Task Scheduling

```yaml
---
- name: Configure task scheduling
  hosts: all
  become: true
  gather_facts: yes
  tasks:
    - name: Install crontabs package
      dnf:
        name: crontabs
        state: present

    - name: Create daily backup cron job
      cron:
        name: "Daily myapp backup"
        minute: "0"
        hour: "2"
        day: "*"
        month: "*"
        weekday: "*"
        job: "/usr/local/bin/myapp-backup.sh >> /var/log/myapp/backup.log 2>&1"
        user: root

    - name: Create weekly log rotation job
      cron:
        name: "Weekly log rotation"
        special_time: weekly
        day: "0"
        job: "/usr/sbin/logrotate -s /var/log/myapp/myapp.logrotate /etc/myapp/myapp.logrotate"
        user: root

    - name: Create health check cron job (every 5 minutes)
      cron:
        name: "Health check"
        minute: "*/5"
        hour: "*"
        job: "/usr/local/bin/health-check.sh"
        user: root

    - name: Create monthly maintenance job
      cron:
        name: "Monthly maintenance"
        special_time: monthly
        day: "1"
        job: "/usr/local/bin/monthly-maintenance.sh"
        user: root
        env:
          PATH: "/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin"

    - name: Remove old backup cron job
      cron:
        name: "Old backup job"
        state: absent

    - name: Create systemd timer for application restart
      systemd:
        name: app-restart.timer
        enabled: yes
        daemon_reload: yes
      when:
        - ansible_facts.services.app-restart.service is defined

    - name: Create systemd service for scheduled task
      copy:
        content: |
          [Unit]
          Description=Myapp Scheduled Task
          After=network.target

          [Service]
          Type=oneshot
          ExecStart=/usr/local/bin/myapp-task.sh
          User=root

          [Install]
          WantedBy=multi-user.target
        dest: /etc/systemd/system/myapp-task.service
        owner: root
        group: root
        mode: "0644"
      notify: daemon reload

  handlers:
    - name: daemon reload
      systemd:
        daemon_reload: yes
```

### Example 7: Security Configuration

```yaml
---
- name: Configure system security
  hosts: all
  become: true
  gather_facts: yes
  tasks:
    - name: Ensure SELinux is enforcing
      selinux:
        state: enforcing
        policy: targeted

    - name: Set SELinux boolean for HTTPD
      seboolean:
        name: httpd_can_network_connect
        state: yes
        persistent: yes

    - name: Set SELinux file context for web content
      sefcontext:
        target: "/var/www/html(/.*)?"
        setype: httpd_sys_content_t
        state: present

    - name: Apply SELinux file contexts
      command: restorecon -Rv /var/www/html
      changed_when: false

    - name: Configure SSH security settings
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
        - { regexp: "^#?AllowTcpForwarding", line: "AllowTcpForwarding no" }
      notify: restart sshd

    - name: Configure password aging policy
      pam:
        name: system-auth
        param: password
        control: required
        type: "pam_pwquality.so retry=3 minlen=14"
        new_type: "pam_pwquality.so retry=3 minlen=14"

    - name: Configure login.defs password aging
      lineinfile:
        path: /etc/login.defs
        regexp: "^{{ item.key }}"
        line: "{{ item.key }} {{ item.value }}"
      loop:
        - { key: "PASS_MAX_DAYS", value: "90" }
        - { key: "PASS_MIN_DAYS", value: "7" }
        - { key: "PASS_MIN_LEN", value: "14" }
        - { key: "PASS_WARN_AGE", value: "7" }

    - name: Ensure auditd is running
      systemd:
        name: auditd
        state: started
        enabled: yes

    - name: Configure firewall to block insecure protocols
      firewalld:
        service: "{{ item }}"
        permanent: yes
        immediate: yes
        state: disabled
      loop:
        - telnet
        - ftp
        - tftp
      notify: reload firewalld

    - name: Create secure sudoers file
      copy:
        content: |
          # Sudoers file managed by Ansible
          %wheel ALL=(ALL) NOPASSWD: ALL
          ansible ALL=(ALL) NOPASSWD: ALL
        dest: /etc/sudoers.d/managed
        owner: root
        group: root
        mode: "0440"
        validate: "visudo -cf %s"

  handlers:
    - name: restart sshd
      systemd:
        name: sshd
        state: restarted

    - name: reload firewalld
      systemd:
        name: firewalld
        state: reloaded
```

### Example 8: User and Group Management

```yaml
---
- name: Manage users and groups
  hosts: all
  become: true
  gather_facts: yes
  vars:
    admin_group_members:
      - alice
      - bob
    standard_users:
      - name: charlie
        uid: 3001
        groups: wheel
        comment: "Charlie Admin"
        shell: /bin/bash
      - name: dave
        uid: 3002
        groups: docker
        comment: "Dave Developer"
        shell: /bin/zsh
      - name: eve
        uid: 3003
        comment: "Eve Temp User"
        shell: /sbin/nologin
        password_lock: lock
    service_accounts:
      - name: myapp
        system: yes
        shell: /sbin/nologin
        home: /var/lib/myapp
      - name: webuser
        system: yes
        shell: /sbin/nologin
        home: /var/www
  tasks:
    - name: Create admin group
      group:
        name: admins
        system: yes
        gid: 2000

    - name: Add users to admin group
      user:
        name: "{{ item }}"
        groups: admins
        append: yes
      loop: "{{ admin_group_members }}"

    - name: Create standard users with specific attributes
      user:
        name: "{{ item.name }}"
        uid: "{{ item.uid }}"
        groups: "{{ item.groups }}"
        comment: "{{ item.comment }}"
        shell: "{{ item.shell }}"
        createhome: yes
        state: present
      loop: "{{ standard_users }}"

    - name: Create service accounts
      user:
        name: "{{ item.name }}"
        system: yes
        shell: "{{ item.shell }}"
        home: "{{ item.home | default('/var/lib') }}"
        createhome: no
        state: present
      loop: "{{ service_accounts }}"

    - name: Ensure sudoers has wheel group entry
      lineinfile:
        path: /etc/sudoers.d/wheel
        line: "%wheel ALL=(ALL) NOPASSWD: ALL"
        create: yes
        validate: "visudo -cf %s"
        mode: "0440"

    - name: Set password for alice
      user:
        name: alice
        password: "{{ 'P@ssw0rd123' | password_hash('sha512') }}"
        update_password: on_create

    - name: Lock eve's account
      user:
        name: eve
        password_lock: lock

    - name: Remove olduser account
      user:
        name: olduser
        state: absent
        remove: yes

    - name: Ensure shadow group exists
      group:
        name: shadow
        state: present
```

## Explanation of Each Playbook

### Playbook 1: Software Package Management

This playbook demonstrates the `dnf` module for package management. The `update_cache: yes` parameter refreshes the package metadata before installing packages. The `state: present` parameter ensures packages are installed without upgrading unless explicitly requested. DNF module-stream handling is environment-sensitive; verify the supported method in the provided execution environment before using module streams.

The `copy` module creates custom repositories by writing to `/etc/yum.repos.d/`. The `validate` parameter ensures the repository file is valid before writing. This replaces the manual process of creating and editing `.repo` files.

### Playbook 2: Service Management

The `systemd` module replaces `systemctl` commands. Setting `state: started` ensures the service is running now. Setting `enabled: yes` ensures the service starts at boot. Using `daemon_reload: yes` runs `systemctl daemon-reload` before making changes, which is required after modifying unit files. The `when` conditionals check if a package is installed before attempting to manage its service, preventing failures on nodes where the package is not present.

### Playbook 3: Firewall Configuration

The `firewalld` module replaces `firewall-cmd` commands. Setting `permanent: yes` and `immediate: yes` applies rules to both the runtime and permanent configuration, equivalent to using `--permanent` and then `--reload`. The `port` parameter accepts both single ports and ranges (e.g., `8080-8090/tcp`). The `toport` and `toaddr` parameters configure port forwarding.

### Playbook 4: File System and Storage Management

This playbook demonstrates LVM and filesystem management. The `lvg` module creates volume groups from physical volumes. The `lvol` module creates logical volumes. The `filesystem` module formats devices with XFS or EXT4. The `mount` module both mounts the filesystem and adds an entry to `/etc/fstab` for persistence. The `when` conditions check if devices exist before attempting to create volume groups, preventing errors on nodes without the required hardware.

### Playbook 5: File Content Management

The `copy` module writes content to files. The `lineinfile` module manages individual lines with regex matching, replacing existing lines or adding new ones. The `blockinfile` module manages multi-line blocks with marker comments for tracking. The `template` module processes Jinja2 templates with variables. The `validate` parameter runs syntax checks before writing configuration files.

### Playbook 6: Task Scheduling

The `cron` module replaces manual `crontab` editing. Each job is identified by a `name` field for idempotency. The `special_time` parameter supports named schedules like `weekly`, `monthly`, `yearly`, and `reboot`. The `systemd` module creates and enables timers for scheduled tasks. The `env` parameter sets environment variables within the cron context.

### Playbook 7: Security Configuration

The `selinux` module sets the global SELinux mode. The `seboolean` module toggles SELinux booleans persistently. The `sefcontext` module defines custom file context rules, applied by `restorecon`. The `lineinfile` module configures SSH parameters with `backup: yes` to preserve existing configurations. The `pam` module configures PAM modules for password quality.

### Playbook 8: User and Group Management

The `group` module creates system groups. The `user` module handles user creation with the `createhome: yes` flag. The `append: yes` parameter adds users to supplemental groups without removing existing memberships. The `system: yes` parameter creates system accounts. The `password_hash` filter generates hashed passwords. The `update_password: on_create` setting prevents overwriting existing passwords.

## Variables and Templates

### Using Variables in Managed Node Configuration

```yaml
---
- name: Configure with variables
  hosts: all
  become: true
  vars:
    app_name: myapp
    app_port: 8080
    log_level: info
    max_connections: 100
  tasks:
    - name: Deploy configuration
      copy:
        content: |
          [application]
          name = {{ app_name }}
          port = {{ app_port }}
          log_level = {{ log_level }}
          max_connections = {{ max_connections }}
        dest: /etc/{{ app_name }}/config.ini
```

### Using Host Variables

```yaml
---
- name: Use host variables
  hosts: all
  become: true
  vars:
    common_packages:
      - vim-enhanced
      - net-tools
  tasks:
    - name: Install common packages
      dnf:
        name: "{{ item }}"
        state: present
      loop: "{{ common_packages }}"

    - name: Install web-specific packages
      dnf:
        name: "{{ web_packages }}"
        state: present
      when: inventory_hostname in groups.webservers
      vars:
        web_packages:
          - httpd
          - mod_ssl
```

### Using Jinja2 Templates

```yaml
# Template: httpd.conf.j2
# Managed by Ansible - Do not edit manually

ServerRoot "/etc/httpd"
Listen {{ httpd_port | default(80) }}
MaxClients {{ max_clients | default(256) }}

{% if enable_ssl | default(false) %}
SSLEngine on
SSLCertificateFile /etc/pki/tls/certs/server.crt
SSLCertificateKeyFile /etc/pki/tls/private/server.key
{% endif %}

<VirtualHost *:{{ httpd_port | default(80) }}>
    ServerName {{ inventory_hostname }}
    DocumentRoot "/var/www/{{ app_name }}"
    ServerAlias www.{{ inventory_hostname }}
{% if enable_logging | default(true) %}
    ErrorLog /var/log/httpd/{{ app_name }}_error.log
    CustomLog /var/log/httpd/{{ app_name }}_access.log combined
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
    max_clients: 256
  tasks:
    - name: Create document root
      file:
        path: /var/www/{{ app_name }}
        state: directory
        owner: apache
        group: apache
        mode: "0755"

    - name: Deploy httpd configuration
      template:
        src: templates/httpd.conf.j2
        dest: /etc/httpd/conf.d/{{ app_name }}.conf
        owner: root
        group: root
        mode: "0644"
        validate: "httpd -t -f %s"
      notify: restart httpd

    - name: Restart httpd
      systemd:
        name: httpd
        state: restarted
  handlers:
    - name: restart httpd
      systemd:
        name: httpd
        state: restarted
```

### Using Facts in Templates

```yaml
---
- name: Use facts in configuration
  hosts: all
  become: true
  gather_facts: yes
  tasks:
    - name: Deploy configuration with facts
      template:
        src: templates/system.conf.j2
        dest: /etc/myapp/system.conf
      vars:
        system_name: "{{ inventory_hostname }}"
        environment: "{{ environment | default('production') }}"
        log_dir: "/var/log/{{ app_name }}"
```

## Verification Procedures

### Verify Package Installation

```bash
# Check installed packages
ansible all -m dnf -a "list_installed vim-enhanced"
ansible all -m command -a "rpm -q vim-enhanced httpd firewalld"

# Check package cache
ansible all -m command -a "dnf repolist"
```

### Verify Service Status

```bash
# Check service status
ansible all -m command -a "systemctl is-active firewalld sshd httpd"
ansible all -m command -a "systemctl is-enabled firewalld sshd httpd"

# Check service units
ansible all -m command -a "systemctl list-units --type=service"
```

### Verify Firewall Rules

```bash
# List firewall services
ansible all -m command -a "firewall-cmd --list-services"

# List firewall ports
ansible all -m command -a "firewall-cmd --list-ports"

# List all zones
ansible all -m command -a "firewall-cmd --get-zones"
```

### Verify File System

```bash
# Check mounted filesystems
ansible all -m command -a "df -hT"
ansible all -m command -a "mount | grep -v tmpfs"

# Check LVM
ansible all -m command -a "vgs"
ansible all -m command -a "lvs"
ansible all -m command -a "pvs"
```

### Verify File Content

```bash
# Check deployed files
ansible all -m command -a "ls -la /etc/myapp/"
ansible all -m command -a "cat /etc/myapp/config.ini"

# Check file permissions
ansible all -m stat -a "path=/etc/myapp/config.ini"
```

### Verify Cron Jobs

```bash
# List cron jobs
ansible all -m command -a "crontab -l"
ansible all -m command -a "ls -la /etc/cron.d/"
```

### Verify SELinux

```bash
# Check SELinux status
ansible all -m command -a "getenforce"
ansible all -m command -a "sestatus"

# Check SELinux booleans
ansible all -m command -a "getsebool httpd_can_network_connect"

# Check file contexts
ansible all -m command -a "ls -Z /var/www/html/"
```

### Verify Users and Groups

```bash
# List users
ansible all -m command -a "id alice charlie dave"
ansible all -m command -a "getent passwd myapp webuser"

# List groups
ansible all -m command -a "getent group admins wheel docker"

# Check sudoers
ansible all -m command -a "cat /etc/sudoers.d/managed"
```

## Troubleshooting

### Package Installation Failures

**Problem: Package not found in repository**

```yaml
# Solution: Update package cache first
- name: Update package cache
  dnf:
    name: "*"
    state: present
    update_cache: yes

# Then install package
- name: Install package
  dnf:
    name: package_name
    state: present
```

**Problem: GPG key verification failure**

```yaml
# Solution: Disable GPG check for trusted internal repos
- name: Install package
  dnf:
    name: package_name
    state: present
    disable_gpg_check: true
```

### Service Management Issues

**Problem: Service fails to start**

```yaml
# Solution: Check dependencies first
- name: Check dependencies
  command: rpm -q --queryformat '%{REQUIRES}\n' httpd
  register: dependencies

- name: Install dependencies
  dnf:
    name: "{{ dependencies.stdout_lines }}"
    state: present

# Then start service
- name: Start service
  systemd:
    name: httpd
    state: started
    enabled: yes
```

**Problem: Service restarts fail due to config syntax errors**

```yaml
# Solution: Validate configuration before restarting
- name: Validate httpd configuration
  command: httpd -t
  register: httpd_test
  changed_when: false

- name: Deploy configuration
  template:
    src: httpd.conf.j2
    dest: /etc/httpd/conf.d/myapp.conf
  notify: restart httpd
  when: httpd_test.rc == 0

- name: Restart httpd
  systemd:
    name: httpd
    state: restarted
```

### Firewall Configuration Issues

**Problem: Port forwarding not working**

```yaml
# Solution: Verify firewalld is running first
- name: Ensure firewalld is running
  systemd:
    name: firewalld
    state: started
    enabled: yes
  notify: reload firewalld

# Then configure port forwarding
- name: Configure port forwarding
  firewalld:
    port: "8443/tcp"
    toport: "443/tcp"
    toaddr: "127.0.0.1"
    permanent: yes
    immediate: yes
    state: enabled
  notify: reload firewalld
```

### File System Issues

**Problem: Mount fails because device does not exist**

```yaml
# Solution: Check device existence before mounting
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

### User and Group Issues

**Problem: User creation fails due to UID conflict**

```yaml
# Solution: Check UID availability first
- name: Check if UID is available
  command: getent passwd {{ user_uid }}
  register: uid_check
  failed_when: false
  changed_when: false

- name: Create user
  user:
    name: {{ user_name }}
    uid: {{ user_uid }}
    state: present
  when: uid_check.rc != 0
```

## Real-World Automation Examples

### Complete Managed Node Configuration

```yaml
---
- name: Complete managed node configuration
  hosts: all
  become: true
  gather_facts: yes
  vars:
    app_name: myapp
    app_port: 8080
    log_level: info
  tasks:
    # Package Installation
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
          - httpd
          - firewalld
          - crontabs
        state: present

    # Service Configuration
    - name: Ensure firewalld is running
      systemd:
        name: firewalld
        state: started
        enabled: yes
      notify: reload firewalld

    - name: Ensure SSH is running
      systemd:
        name: sshd
        state: started
        enabled: yes
      notify: restart sshd

    # Firewall Configuration
    - name: Open HTTP port
      firewalld:
        port: "80/tcp"
        permanent: yes
        immediate: yes
        state: enabled
      notify: reload firewalld

    - name: Open HTTPS port
      firewalld:
        port: "443/tcp"
        permanent: yes
        immediate: yes
        state: enabled
      notify: reload firewalld

    # File System Configuration
    - name: Create application directory
      file:
        path: /etc/{{ app_name }}
        state: directory
        owner: root
        group: root
        mode: "0755"

    - name: Create document root
      file:
        path: /var/www/{{ app_name }}
        state: directory
        owner: apache
        group: apache
        mode: "0755"

    # File Content Deployment
    - name: Deploy application configuration
      template:
        src: templates/myapp.conf.j2
        dest: /etc/{{ app_name }}/config.ini
        owner: root
        group: root
        mode: "0644"

    # User and Group Management
    - name: Create application user
      user:
        name: "{{ app_name }}"
        system: yes
        shell: /sbin/nologin
        home: /var/lib/{{ app_name }}
        createhome: no

    - name: Create application group
      group:
        name: "{{ app_name }}"
        system: yes

    # Task Scheduling
    - name: Create backup cron job
      cron:
        name: "Daily backup"
        minute: "0"
        hour: "2"
        job: "/usr/local/bin/backup.sh >> /var/log/backup.log 2>&1"
        user: root

  handlers:
    - name: reload firewalld
      systemd:
        name: firewalld
        state: reloaded

    - name: restart sshd
      systemd:
        name: sshd
        state: restarted
```

## RHCE Exam Notes

### Critical Exam Points

1. **Managed node configuration is exam-critical**: You will be expected to configure managed nodes from scratch. Know all prerequisites: SSH, Python, sudo access.

2. **SSH keys must work before playbooks**: If you cannot ping managed nodes, no playbook will work. Test connectivity early and fix authentication issues first.

3. **Sudoers validation is mandatory**: Always use `validate: "visudo -cf %s"` when writing sudoers files. A syntax error locks out sudo entirely.

4. **Facts are your friend**: Use facts to make decisions about what to install and configure. Check if packages exist before installing, if services exist before starting.

5. **Firewall must be configured**: Many exam scenarios require opening specific ports. Know the `firewalld` module parameters.

6. **SELinux affects services**: Some services won't work without proper SELinux contexts. Know when to configure `seboolean` and `sefcontext`.

7. **Service dependencies matter**: Some services require other packages to be installed first. Check and install dependencies.

8. **Use handlers for service restarts**: Services should only restart when configuration changes. Use `notify` with handlers.

### Exam Workflow

1. Create or verify `ansible.cfg` with correct settings
2. Create or verify inventory file
3. Test SSH connectivity to all managed nodes
4. Configure managed nodes (SSH keys, sudoers, packages)
5. Run configuration playbooks
6. Verify services are running
7. Verify firewall rules are applied
8. Commit changes to Git

### Time Management

- 30 minutes: Initial configuration (ansible.cfg, inventory, SSH)
- 15 minutes: Managed node configuration (packages, services)
- 15 minutes: Task-specific automation (firewall, users, files)
- Reserve 30 minutes for verification and Git commits

## Common Mistakes

### Mistake 1: Not Checking if Package is Installed Before Service

```yaml
# WRONG - service may not exist
- name: Start httpd
  systemd:
    name: httpd
    state: started
    enabled: yes

# CORRECT - check package first
- name: Install httpd
  dnf:
    name: httpd
    state: present

- name: Start httpd
  systemd:
    name: httpd
    state: started
    enabled: yes
  when: ansible_facts.packages.httpd.state == "present"
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

### Mistake 3: Not Checking Firewall Service First

```yaml
# WRONG - firewalld may not be running
- name: Open port
  firewalld:
    port: "80/tcp"
    permanent: yes
    immediate: yes
    state: enabled

# CORRECT - ensure firewalld is running
- name: Ensure firewalld is running
  systemd:
    name: firewalld
    state: started
    enabled: yes
  notify: reload firewalld

- name: Open port
  firewalld:
    port: "80/tcp"
    permanent: yes
    immediate: yes
    state: enabled
  notify: reload firewalld
```

### Mistake 4: Not Using Facts for Conditional Configuration

```yaml
# WRONG - installs packages even if already present
- name: Install packages
  dnf:
    name: httpd
    state: present

# CORRECT - check if package is installed
- name: Install httpd if not present
  dnf:
    name: httpd
    state: present
  when:
    - ansible_facts.packages.httpd is defined
    - ansible_facts.packages.httpd.state != "present"
```

### Mistake 5: Forgetting to Reload systemd After Changes

```yaml
# WRONG - changes not applied
- name: Create service file
  copy:
    content: "[Unit]\nDescription=My Service"
    dest: /etc/systemd/system/my.service

# CORRECT - reload systemd daemon
- name: Create service file
  copy:
    content: "[Unit]\nDescription=My Service"
    dest: /etc/systemd/system/my.service
    owner: root
    group: root
    mode: "0644"
  notify: reload systemd

handlers:
  - name: reload systemd
    systemd:
      daemon_reload: yes
```

## Chapter Summary

This chapter covered configuring Ansible managed nodes for automation:

- **Software packages** are managed with `ansible.builtin.dnf` or a verified package-management module available in the execution environment. Update the cache before installing, and check for dependencies.
- **Services** are configured with the `systemd` module. Use `state: started` and `enabled: yes` for services that should run now and persist across reboots.
- **Firewall rules** are configured with the `firewalld` module. Always ensure firewalld is running before configuring rules.
- **File systems** are managed with `lvg`, `lvol`, `filesystem`, and `mount` modules. Check device existence before creating volume groups.
- **File content** is deployed with `copy`, `template`, `lineinfile`, and `blockinfile` modules. Use `validate` for configuration files.
- **Task scheduling** uses the `cron` module and `systemd` timers. The `special_time` parameter supports named schedules.
- **Security** is configured with `selinux`, `seboolean`, and `sefcontext` modules. SELinux booleans and file contexts are often required for services to function.
- **Users and groups** are managed with the `user` and `group` modules. Use `createhome: yes` for regular users and `system: yes` for service accounts.
- **Verification** procedures check packages, services, firewall rules, filesystems, file content, cron jobs, SELinux status, and user accounts.
- **Troubleshooting** common issues includes package installation failures, service management problems, firewall configuration issues, and file system mounting failures.

## Quick Reference

| Task | Module | Key Parameters |
|---|---|---|
| Install package | `dnf` | `name`, `state: present`, `update_cache: yes` |
| Remove package | `dnf` | `name`, `state: absent` |
| Start service | `systemd` | `name`, `state: started`, `enabled: yes` |
| Stop service | `systemd` | `name`, `state: stopped`, `enabled: no` |
| Reload service | `systemd` | `name`, `state: reloaded` |
| Reload daemon | `systemd` | `daemon_reload: yes` |
| Open firewall port | `firewalld` | `port`, `permanent: yes`, `immediate: yes` |
| Port forwarding | `firewalld` | `port`, `toport`, `toaddr`, `permanent: yes` |
| Create volume group | `lvg` | `vg`, `pvs` |
| Create logical volume | `lvol` | `vg`, `lv`, `size` |
| Create filesystem | `filesystem` | `dev`, `fstype` |
| Mount filesystem | `mount` | `src`, `path`, `fstype`, `state: mounted` |
| Deploy file | `copy` | `src`, `dest`, `owner`, `group`, `mode` |
| Deploy template | `template` | `src`, `dest`, `validate` |
| Edit line | `lineinfile` | `path`, `regexp`, `line`, `backup: yes` |
| Edit block | `blockinfile` | `path`, `block`, `marker` |
| Create cron job | `cron` | `name`, `minute`, `hour`, `job` |
| Set SELinux mode | `selinux` | `state: enforcing`, `policy: targeted` |
| Set SELinux boolean | `seboolean` | `name`, `state`, `persistent: yes` |
| Set file context | `sefcontext` | `target`, `setype`, `state: present` |
| Create user | `user` | `name`, `createhome: yes`, `shell` |
| Create group | `group` | `name`, `system: yes` |
| Validate sudoers | `copy` | `validate: "visudo -cf %s"` |

## Review Questions

1. What are the three essential prerequisites for a managed node to be configured by Ansible?

2. How do you verify which package-management and module-stream methods are available before writing package automation?

3. What is the purpose of the `when` parameter in a task, and how does it help with managed node configuration?

4. How do you configure a firewall rule that applies both immediately and permanently, and what parameter controls this?

5. What is the difference between `state: mounted` and `state: present` in the `mount` module?

6. How does the `blockinfile` module use marker comments, and why are they important?

7. What is the purpose of the `validate` parameter in the `copy` module, and why is it critical for sudoers files?

8. How does the `seboolean` module's `persistent` parameter affect SELinux boolean configuration?

9. What is the purpose of the `daemon_reload` parameter in the `systemd` module, and when is it required?

10. How would you create a cron job that runs every 15 minutes, and what parameter specifies the minute field?

## Answers

1. The three essential prerequisites for a managed node to be configured by Ansible are: (1) SSH server running and accessible from the control node, (2) Python 3 installed on the managed node for module execution, and (3) sudo privileges configured for the Ansible user to execute root-level tasks. Without SSH access, Ansible cannot connect. Without Python, modules cannot execute. Without sudo, root-level tasks fail.

2. Use `ansible.builtin.dnf` for ordinary package installation, removal, and updates. For module streams or DNF5-specific behavior, first verify the supported method with `ansible-doc`, `ansible-navigator doc`, and the official documentation available in the execution environment. Do not assume a module exists because an older example used it.

3. The `when` parameter evaluates a condition before executing a task. If the condition is true, the task runs. If false, the task is skipped. In managed node configuration, `when` is used to check if packages are installed, services exist, or facts meet certain criteria before attempting operations, preventing errors on nodes where prerequisites are not met.

4. To configure a firewall rule that applies both immediately and permanently, set both `permanent: yes` and `immediate: yes` in the `firewalld` module. The `permanent: yes` parameter writes the rule to the permanent configuration file so it survives reboots. The `immediate: yes` parameter applies the rule to the running firewall without requiring a reload.

5. The `state: mounted` parameter both mounts the filesystem immediately and adds an entry to `/etc/fstab` for persistence across reboots. The `state: present` parameter only adds the entry to `/etc/fstab` but does not mount the filesystem immediately. Use `state: mounted` when you need the filesystem mounted right away and persisted for future boots.

6. The `blockinfile` module uses marker comments (like `# {mark} ANSIBLE MANAGED BLOCK`) to identify the beginning and end of managed content within a file. This allows Ansible to detect if a block already exists and skip adding it again, ensuring idempotency. The `marker` parameter defines the marker string used for identification.

7. The `validate` parameter runs a command on the managed node to check the syntax or validity of a file before it is deployed. The `%s` placeholder is replaced with the path to a temporary file containing the new content. For sudoers files, `validate: "visudo -cf %s"` checks the syntax before writing, preventing syntax errors that could lock out sudo access entirely.

8. The `persistent: yes` parameter in the `seboolean` module ensures the SELinux boolean change is written to the SELinux policy database and persists across reboots. Without this parameter, the boolean change is only temporary and will be reset to its default value after a system reboot.

9. The `daemon_reload` parameter runs `systemctl daemon-reload` before making changes to systemd unit files. This is required whenever you modify or add systemd unit files (like service files or timer files) because systemd needs to reload its configuration to recognize the new or modified units.

10. To create a cron job that runs every 15 minutes, use the `cron` module with `minute: "*/15"`. The `*` wildcard matches all values, and `*/15` means "every 15 minutes" (at minutes 0, 15, 30, and 45 of each hour). The full crontab entry would be: `*/15 * * * * /path/to/script.sh`.


---

# Chapter 04 — Reviewed Edition Closing Checkpoint

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
