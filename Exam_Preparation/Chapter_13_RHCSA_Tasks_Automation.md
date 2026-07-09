# Chapter 13: Automate Standard RHCSA Tasks Using Ansible Modules

## Learning Objectives

By the end of this chapter, you will be able to

- Automate software package and repository management
- Automate service deployment and configuration
- Automate firewall rule configuration
- Automate file system and storage device configuration
- Automate file content management
- Automate archiving and backup operations
- Automate task scheduling with cron and systemd timers
- Automate security configurations including SELinux and firewall
- Automate user and group management
- Combine multiple modules for complex RHCSA workflows
- Troubleshoot module execution failures

## Concept Overview

The RHCSA exam tests hands-on system administration skills. Ansible automates these tasks by using modules that correspond to manual commands. Each RHCSA task maps to one or more Ansible modules that achieve the same result idempotently.

### Why Automation for RHCSA Tasks Matters

Without automation:

- Manual configuration is time-consuming
- Human error is common
- Configuration drift occurs
- Repeating tasks is inefficient
- Auditing is difficult
- Scaling is impossible

### How Ansible Automates RHCSA Tasks

```
RHCSA Task → Ansible Module → Idempotent Automation
─────────────────────────────────────────────────────────────────
Install package → dnf → state: present
Remove package → dnf → state: absent
Start service → systemd → state: started
Open firewall → firewalld → state: enabled
Create user → user → state: present
Mount filesystem → mount → state: mounted
Create cron job → cron → state: present
```

## Ansible Fundamentals Required

### RHCSA Task Module Mapping

| RHCSA Task | Ansible Module | Key Parameters |
|---|---|---|
| Install packages | `dnf`, `dnf_module` | `name`, `state: present` |
| Remove packages | `dnf` | `name`, `state: absent` |
| Start services | `systemd` | `name`, `state: started` |
| Stop services | `systemd` | `name`, `state: stopped` |
| Configure firewall | `firewalld` | `port`, `service`, `permanent: yes` |
| Create users | `user` | `name`, `state: present` |
| Create groups | `group` | `name`, `system: yes` |
| Mount filesystems | `mount` | `src`, `path`, `fstype`, `state: mounted` |
| Create cron jobs | `cron` | `name`, `minute`, `hour`, `job` |
| Deploy files | `copy`, `template` | `src`, `dest`, `content` |
| Create archives | `archive` | `path`, `dest`, `format` |
| Extract archives | `unarchive` | `src`, `dest`, `remote_src: yes` |
| Configure SELinux | `selinux`, `seboolean` | `state`, `name`, `persistent: yes` |

## Commands and Tools

```bash
# Package management
ansible all -m dnf -a "name=httpd state=present"
ansible all -m dnf -a "name=httpd state=absent"
ansible all -m dnf_module -a "name=postgresql:16 state=enabled"

# Service management
ansible all -m systemd -a "name=firewalld state=started enabled=yes"
ansible all -m systemd -a "name=httpd state=reloaded"

# Firewall configuration
ansible all -m firewalld -a "port=80/tcp permanent=yes immediate=yes state=enabled"
ansible all -m firewalld -a "service=ssh permanent=yes immediate=yes state=enabled"

# User and group management
ansible all -m user -a "name=myapp system=yes shell=/sbin/nologin"
ansible all -m group -a "name=myapp system=yes"

# File system and storage
ansible all -m mount -a "src=/dev/vg_data/lv_storage path=/mnt/data fstype=xfs state=mounted"
ansible all -m filesystem -a "dev=/dev/vg_data/lv_storage fstype=xfs"

# File content management
ansible all -m copy -a "src=/tmp/config dest=/etc/config owner=root group=root mode=0644"
ansible all -m template -a "src=/tmp/config.j2 dest=/etc/config owner=root group=root mode=0644"

# Archiving
ansible all -m archive -a "path=/etc/config dest=/backup/config.tar.gz format=gz"
ansible all -m unarchive -a "src=/backup/config.tar.gz dest=/restore/ remote_src=no"

# Task scheduling
ansible all -m cron -a "name=backup minute=0 hour=2 job=/usr/local/bin/backup.sh user=root"

# Security
ansible all -m selinux -a "state=enforcing policy=targeted"
ansible all -m seboolean -a "name=httpd_can_network_connect state=yes persistent=yes"
ansible all -m sefcontext -a "target=/var/www/html(/.*)? setype=httpd_sys_content_t"
```

## Playbook Examples

### Example 1: Software Packages and Repositories

```yaml
---
- name: Automate software packages and repositories
  hosts: all
  become: yes
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
    # Update package cache
    - name: Update all packages to latest version
      dnf:
        name: "*"
        state: present
        update_cache: yes

    # Install common packages
    - name: Install common system packages
      dnf:
        name: "{{ item }}"
        state: present
      loop: "{{ common_packages }}"

    # Install latest version of specific package
    - name: Install latest version of firewalld
      dnf:
        name: firewalld
        state: latest

    # Install specific version using dnf module
    - name: Install PostgreSQL 16 module
      dnf_module:
        name: postgresql:16
        state: enabled

    # Install PostgreSQL server
    - name: Install PostgreSQL server
      dnf:
        name: postgresql-server
        state: present

    # Install crontabs package
    - name: Install crontabs package
      dnf:
        name: crontabs
        state: present

    # Install rsync for file synchronization
    - name: Install rsync package
      dnf:
        name: rsync
        state: present

    # Install git for version control
    - name: Install git package
      dnf:
        name: git
        state: present

    # Install specific package with GPG check disabled
    - name: Install package with GPG check disabled
      dnf:
        name: optional-package
        state: present
        disable_gpg_check: yes

    # Remove unwanted packages
    - name: Remove telnet package
      dnf:
        name: telnet
        state: absent

    # Remove rsyslog package
    - name: Remove rsyslog package
      dnf:
        name: rsyslog
        state: absent

    # Add custom repository
    - name: Add custom repository
      copy:
        content: |
          [custom-repo]
          name=Custom Repository
          baseurl=http://repo.example.com/custom
          gpgcheck=0
          enabled=1
        dest: /etc/yum.repos.d/custom.repo
        owner: root
        group: root
        mode: "0644"

    # Verify package installation
    - name: Verify httpd installation
      command: rpm -q httpd
      register: httpd_check
      changed_when: false
      failed_when: false

    - name: Display package status
      debug:
        msg: "HTTPD {{ 'installed' if httpd_check.rc == 0 else 'not installed' }}"
```

### Example 2: Services Management

```yaml
---
- name: Automate services management
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
    # Ensure firewalld is running and enabled
    - name: Ensure firewalld is running and enabled
      systemd:
        name: firewalld
        state: started
        enabled: yes

    # Ensure SSH is running and enabled
    - name: Ensure SSH is running and enabled
      systemd:
        name: sshd
        state: started
        enabled: yes

    # Ensure NTP service is running and enabled
    - name: Ensure NTP service is running and enabled
      systemd:
        name: chronyd
        state: started
        enabled: yes

    # Ensure httpd is running and enabled
    - name: Ensure httpd is running and enabled
      systemd:
        name: httpd
        state: started
        enabled: yes

    # Stop and disable unnecessary services
    - name: Stop and disable telnet service
      systemd:
        name: telnet
        state: stopped
        enabled: no

    - name: Stop and disable cups service
      systemd:
        name: cups
        state: stopped
        enabled: no

    # Reload systemd daemon after service changes
    - name: Reload systemd daemon
      systemd:
        daemon_reload: yes

    # Restart service on configuration change
    - name: Restart httpd on configuration change
      systemd:
        name: httpd
        state: reloaded

    # Start service in background
    - name: Start service in background
      systemd:
        name: "{{ item }}"
        state: started
        daemon_reload: yes
      loop:
        - httpd
        - firewalld
      loop_control:
        label: "{{ item }}"

    # Enable service at boot
    - name: Enable services at boot
      systemd:
        name: "{{ item }}"
        enabled: yes
      loop:
        - firewalld
        - sshd
        - httpd
      loop_control:
        label: "{{ item }}"

    # Disable service at boot
    - name: Disable services at boot
      systemd:
        name: "{{ item }}"
        enabled: no
      loop:
        - telnet
        - cups
      loop_control:
        label: "{{ item }}"

    # Verify service status
    - name: Verify firewalld status
      command: systemctl is-active firewalld
      register: firewalld_status
      changed_when: false

    - name: Display firewalld status
      debug:
        msg: "Firewalld status: {{ firewalld_status.stdout }}"
```

### Example 3: Firewall Rules

```yaml
---
- name: Automate firewall configuration
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
    # Ensure firewalld is running
    - name: Ensure firewalld is running
      systemd:
        name: firewalld
        state: started
        enabled: yes

    # Open HTTP port
    - name: Open HTTP port permanently and immediately
      firewalld:
        port: "80/tcp"
        permanent: yes
        immediate: yes
        state: enabled

    # Open HTTPS port
    - name: Open HTTPS port permanently and immediately
      firewalld:
        port: "443/tcp"
        permanent: yes
        immediate: yes
        state: enabled

    # Open custom application port
    - name: Open custom application port
      firewalld:
        port: "8080/tcp"
        permanent: yes
        immediate: yes
        state: enabled

    # Open port range
    - name: Open port range for multiple services
      firewalld:
        port: "9000-9010/tcp"
        permanent: yes
        immediate: yes
        state: enabled

    # Allow SSH service
    - name: Allow SSH service permanently
      firewalld:
        service: ssh
        permanent: yes
        immediate: yes
        state: enabled

    # Allow NTP service
    - name: Allow NTP service permanently
      firewalld:
        service: ntp
        permanent: yes
        immediate: yes
        state: enabled

    # Allow DNS service
    - name: Allow DNS service permanently
      firewalld:
        service: dns
        permanent: yes
        immediate: yes
        state: enabled

    # Set default zone to public
    - name: Set default zone to public
      firewalld:
        zone: public
        default: yes
        permanent: yes

    # Configure port forwarding
    - name: Configure port forwarding
      firewalld:
        port: "8443/tcp"
        toport: "443/tcp"
        toaddr: ""
        permanent: yes
        immediate: yes
        state: enabled

    # Block insecure FTP service
    - name: Block FTP service
      firewalld:
        service: ftp
        permanent: yes
        immediate: yes
        state: disabled

    # Remove unused services
    - name: Remove unused services
      firewalld:
        service: "{{ item }}"
        permanent: yes
        immediate: yes
        state: disabled
      loop:
        - dhcpv6-client
        - avahi-autoipd

    # List all open ports
    - name: List all open ports
      command: firewall-cmd --list-ports
      register: open_ports
      changed_when: false

    - name: Display open ports
      debug:
        msg: "Open ports: {{ open_ports.stdout_lines }}"

    # List all services
    - name: List all services
      command: firewall-cmd --list-services
      register: open_services
      changed_when: false

    - name: Display open services
      debug:
        msg: "Open services: {{ open_services.stdout_lines }}"
```

### Example 4: File Systems and Storage Devices

```yaml
---
- name: Automate file systems and storage devices
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
    # Check if storage device exists
    - name: Check if storage device exists
      command: lsblk
      register: lsblk_output
      changed_when: false
      failed_when: false

    # Create volume group
    - name: Create volume group
      lvg:
        vg: vg_data
        pvs: /dev/vdb
      when:
        - ansible_devices.vdb is defined
        - ansible_devices.vdb.size is defined

    # Create logical volume
    - name: Create logical volume
      lvol:
        vg: vg_data
        lv: lv_storage
        size: 10G
      when:
        - ansible_devices.vdb is defined

    # Create XFS filesystem
    - name: Create XFS filesystem
      filesystem:
        dev: /dev/vg_data/lv_storage
        fstype: xfs
      when:
        - ansible_devices.vdb is defined

    # Create EXT4 filesystem
    - name: Create EXT4 filesystem
      filesystem:
        dev: /dev/vg_data/lv_storage
        fstype: ext4
      when:
        - ansible_devices.vdb is defined

    # Create mount point directory
    - name: Create mount point directory
      file:
        path: /mnt/data
        state: directory
        owner: root
        group: root
        mode: "0755"

    # Mount filesystem persistently
    - name: Mount filesystem persistently
      mount:
        path: /mnt/data
        src: /dev/vg_data/lv_storage
        fstype: xfs
        state: mounted
        opts: defaults
        dump: yes
        passno: 2

    # Mount tmpfs for temporary storage
    - name: Mount tmpfs for temporary storage
      mount:
        path: /tmp/shared
        src: tmpfs
        fstype: tmpfs
        state: mounted
        opts: "size=1G,mode=1777"

    # Create LVM physical volume
    - name: Create LVM physical volume
      lvm:
        vg: vg_data
        pvs: /dev/vdb
      when:
        - ansible_devices.vdb is defined

    # Extend volume group
    - name: Extend volume group
      lvg:
        vg: vg_data
        pvs: /dev/vdc
      when:
        - ansible_devices.vdc is defined

    # Resize logical volume
    - name: Resize logical volume
      lvol:
        vg: vg_data
        lv: lv_storage
        size: +10G
      when:
        - ansible_devices.vdb is defined

    # Resize filesystem
    - name: Resize filesystem
      filesystem:
        dev: /dev/vg_data/lv_storage
        fstype: xfs
        state: present

    # Unmount filesystem
    - name: Unmount filesystem
      mount:
        path: /mnt/data
        state: unmounted

    # Check filesystem
    - name: Check filesystem
      command: xfs_repair /dev/vg_data/lv_storage
      when:
        - ansible_devices.vdb is defined
      changed_when: false

    # Display LVM information
    - name: Display LVM information
      command: vgs
      register: vgs_output
      changed_when: false

    - name: Display volume groups
      debug:
        msg: "Volume groups: {{ vgs_output.stdout }}"

    - name: Display logical volumes
      command: lvs
      register: lvs_output
      changed_when: false

    - name: Display logical volumes
      debug:
        msg: "Logical volumes: {{ lvs_output.stdout }}"

    - name: Display physical volumes
      command: pvs
      register: pvs_output
      changed_when: false

    - name: Display physical volumes
      debug:
        msg: "Physical volumes: {{ pvs_output.stdout }}"
```

### Example 5: File Content Management

```yaml
---
- name: Automate file content management
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
    # Create application directory
    - name: Create application directory
      file:
        path: /etc/myapp
        state: directory
        owner: root
        group: root
        mode: "0755"

    # Deploy configuration file with copy
    - name: Deploy configuration file
      copy:
        content: |
          [application]
          name = myapp
          port = 8080
          log_level = info
        dest: /etc/myapp/config.ini
        owner: root
        group: root
        mode: "0644"

    # Add line to configuration file
    - name: Add line to configuration
      lineinfile:
        path: /etc/myapp/config.ini
        line: "timeout = 30"
        create: yes

    # Add line after specific pattern
    - name: Add line after log_level
      lineinfile:
        path: /etc/myapp/config.ini
        line: "max_connections = 100"
        insertafter: "^log_level"
        create: yes

    # Update existing line
    - name: Update port configuration
      lineinfile:
        path: /etc/myapp/config.ini
        regexp: "^port = "
        line: "port = {{ app_port | default(8080) }}"
        state: present

    # Replace commented line
    - name: Replace commented line
      lineinfile:
        path: /etc/myapp/config.ini
        regexp: "^# port = "
        line: "port = 8080"
        state: present

    # Remove specific line
    - name: Remove debug line
      lineinfile:
        path: /etc/myapp/config.ini
        regexp: "^debug = "
        state: absent

    # Add block to configuration file
    - name: Add application block
      blockinfile:
        path: /etc/myapp/config.ini
        block: |
          [application]
          name = myapp
          version = 1.0.0
        marker: "# {mark} ANSIBLE MANAGED BLOCK - application"
        create: yes

    # Add logging block
    - name: Add logging block
      blockinfile:
        path: /etc/myapp/config.ini
        block: |
          [logging]
          log_file = /var/log/myapp/app.log
          log_level = info
          max_size = 100M
          rotate_count = 7
        marker: "# {mark} ANSIBLE MANAGED BLOCK - logging"
        create: yes

    # Deploy file with template
    - name: Deploy configuration template
      template:
        src: templates/myapp.conf.j2
        dest: /etc/myapp/myapp.conf
        owner: root
        group: root
        mode: "0644"
        validate: "grep -q ^\[main\] %s"

    # Deploy SSH configuration
    - name: Deploy SSH configuration
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

    # Create log file
    - name: Create log file
      file:
        path: /var/log/myapp/app.log
        state: touch
        owner: myapp
        group: myapp
        mode: "0640"

    # Set file permissions
    - name: Set file permissions
      file:
        path: /etc/myapp/config.ini
        owner: root
        group: myapp
        mode: "0640"

    # Set directory permissions
    - name: Set directory permissions
      file:
        path: /etc/myapp
        owner: root
        group: myapp
        mode: "0750"

    # Create symbolic link
    - name: Create symbolic link
      file:
        src: /etc/myapp/config.ini
        dest: /etc/myapp/current.ini
        state: link

    # Create hard link
    - name: Create hard link
      file:
        src: /etc/myapp/config.ini
        dest: /etc/myapp/config.backup
        state: link
        force: yes

    # Display file information
    - name: Display file information
      command: ls -la /etc/myapp/
      register: file_info
      changed_when: false

    - name: Display file permissions
      debug:
        msg: "File permissions: {{ file_info.stdout }}"
```

### Example 6: Archiving

```yaml
---
- name: Automate archiving and backup operations
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
    # Create backup directory
    - name: Create backup directory
      file:
        path: /backup
        state: directory
        owner: root
        group: root
        mode: "0755"

    # Create tar.gz archive
    - name: Create tar.gz archive
      archive:
        path:
          - /etc/myapp
          - /var/log/myapp
        dest: /backup/myapp-config-{{ ansible_date_time.date }}.tar.gz
        format: gz
        owner: root
        group: root
        mode: "0600"

    # Create tar.bz2 archive
    - name: Create tar.bz2 archive
      archive:
        path: /etc/myapp
        dest: /backup/myapp-config.tar.bz2
        format: bz2
        owner: root
        group: root
        mode: "0600"

    # Create tar.xz archive
    - name: Create tar.xz archive
      archive:
        path: /etc/myapp
        dest: /backup/myapp-config.tar.xz
        format: xz
        owner: root
        group: root
        mode: "0600"

    # Create zip archive
    - name: Create zip archive
      archive:
        path: /etc/myapp
        dest: /backup/myapp-config.zip
        format: zip
        owner: root
        group: root
        mode: "0600"

    # Create archive with specific options
    - name: Create archive with compression
      archive:
        path: /etc/myapp
        dest: /backup/myapp-config-compressed.tar.gz
        format: gz
        owner: root
        group: root
        mode: "0600"
        remove: no

    # Extract archive to destination
    - name: Extract archive to destination
      unarchive:
        src: /backup/myapp-config-latest.tar.gz
        dest: /restore/
        owner: root
        group: root
        mode: "0644"
        remote_src: no

    # Extract remote archive
    - name: Extract remote archive
      unarchive:
        src: http://example.com/packages/app.tar.gz
        dest: /opt/
        remote_src: yes
        owner: root
        group: root
        mode: "0755"

    # Extract archive with options
    - name: Extract archive with options
      unarchive:
        src: /backup/myapp-config.tar.gz
        dest: /restore/
        remote_src: no
        owner: root
        group: root
        mode: "0644"
        copy: yes
        extra_opts:
          - --strip-components=1

    # Create nested archive
    - name: Create nested archive
      archive:
        path:
          - /etc/myapp/config.ini
          - /etc/myapp/data/
        dest: /backup/myapp-nested.tar.gz
        format: gz
        owner: root
        group: root
        mode: "0600"

    # List archive contents
    - name: List archive contents
      command: tar -tzf /backup/myapp-config.tar.gz
      register: archive_contents
      changed_when: false

    - name: Display archive contents
      debug:
        msg: "Archive contents: {{ archive_contents.stdout_lines }}"

    # Check archive size
    - name: Check archive size
      command: ls -lh /backup/myapp-config.tar.gz
      register: archive_size
      changed_when: false

    - name: Display archive size
      debug:
        msg: "Archive size: {{ archive_size.stdout }}"

    # Verify archive integrity
    - name: Verify archive integrity
      command: tar -tzf /backup/myapp-config.tar.gz
      register: archive_verify
      changed_when: false

    - name: Display archive verification
      debug:
        msg: "Archive {{ 'valid' if archive_verify.rc == 0 else 'invalid' }}"
```

### Example 7: Task Scheduling

```yaml
---
- name: Automate task scheduling
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
    # Install crontabs package
    - name: Install crontabs package
      dnf:
        name: crontabs
        state: present

    # Create daily backup cron job
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

    # Create weekly report cron job
    - name: Create weekly report cron job
      cron:
        name: "Weekly report"
        special_time: weekly
        day: "0"
        job: "/usr/local/bin/generate_report.sh"
        user: root

    # Create health check cron job (every 5 minutes)
    - name: Create health check cron job
      cron:
        name: "Health check"
        minute: "*/5"
        hour: "*"
        job: "/usr/local/bin/health-check.sh"
        user: root

    # Create monthly maintenance cron job
    - name: Create monthly maintenance cron job
      cron:
        name: "Monthly maintenance"
        special_time: monthly
        day: "1"
        job: "/usr/local/bin/monthly-maintenance.sh"
        user: root
        env:
          PATH: "/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin"

    # Create cron job with environment variables
    - name: Create cron job with environment variables
      cron:
        name: "Custom cron job"
        minute: "0"
        hour: "3"
        job: "/usr/local/bin/custom-job.sh"
        user: root
        env:
          BACKUP_DIR: "/backup"
          LOG_FILE: "/var/log/custom-job.log"

    # Remove cron job by name
    - name: Remove old backup cron job
      cron:
        name: "Old backup job"
        state: absent

    # Create systemd timer
    - name: Create systemd timer
      systemd:
        name: backup.timer
        enabled: yes
        daemon_reload: yes

    # Create systemd service
    - name: Create systemd service
      copy:
        content: |
          [Unit]
          Description=Backup Service
          After=network.target

          [Service]
          Type=oneshot
          ExecStart=/usr/local/bin/backup.sh
          User=root

          [Install]
          WantedBy=multi-user.target
        dest: /etc/systemd/system/backup.service
        owner: root
        group: root
        mode: "0644"
      notify: daemon reload

    # List cron jobs
    - name: List cron jobs
      command: crontab -l
      register: crontab_output
      changed_when: false

    - name: Display cron jobs
      debug:
        msg: "Cron jobs: {{ crontab_output.stdout }}"

    # List systemd timers
    - name: List systemd timers
      command: systemctl list-timers
      register: timers_output
      changed_when: false

    - name: Display systemd timers
      debug:
        msg: "Systemd timers: {{ timers_output.stdout }}"
```

### Example 8: Security Configuration

```yaml
---
- name: Automate security configuration
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
    # Configure SELinux
    - name: Ensure SELinux is enforcing
      selinux:
        state: enforcing
        policy: targeted

    # Set SELinux mode to permissive
    - name: Set SELinux to permissive
      selinux:
        state: permissive
        policy: targeted

    # Set SELinux mode to disabled
    - name: Disable SELinux
      selinux:
        state: disabled
        policy: targeted

    # Set SELinux boolean
    - name: Set httpd_can_network_connect boolean
      seboolean:
        name: httpd_can_network_connect
        state: yes
        persistent: yes

    # Set SELinux file context
    - name: Set SELinux file context
      sefcontext:
        target: "/var/www/html(/.*)?"
        setype: httpd_sys_content_t
        state: present

    # Apply SELinux file contexts
    - name: Apply SELinux file contexts
      command: restorecon -Rv /var/www/html
      changed_when: false

    # Configure SSH security
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

    # Configure password aging
    - name: Configure password aging policy
      pam:
        name: system-auth
        param: password
        control: required
        type: "pam_pwquality.so retry=3 minlen=14"
        new_type: "pam_pwquality.so retry=3 minlen=14"

    # Configure login.defs
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

    # Ensure auditd is running
    - name: Ensure auditd is running
      systemd:
        name: auditd
        state: started
        enabled: yes

    # Configure firewall to block insecure protocols
    - name: Block insecure protocols
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

    # Create secure sudoers file
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

    # Verify SELinux status
    - name: Verify SELinux status
      command: getenforce
      register: selinux_status
      changed_when: false

    - name: Display SELinux status
      debug:
        msg: "SELinux status: {{ selinux_status.stdout }}"

    # Verify SSH configuration
    - name: Verify SSH configuration
      command: sshd -t
      register: sshd_test
      changed_when: false

    - name: Display SSH configuration status
      debug:
        msg: "SSH configuration {{ 'valid' if sshd_test.rc == 0 else 'invalid' }}"
```

### Example 9: Users and Groups

```yaml
---
- name: Automate users and groups management
  hosts: all
  become: yes
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
    # Create admin group
    - name: Create admin group
      group:
        name: admins
        system: yes
        gid: 2000

    # Add users to admin group
    - name: Add users to admin group
      user:
        name: "{{ item }}"
        groups: admins
        append: yes
      loop: "{{ admin_group_members }}"

    # Create standard users with specific attributes
    - name: Create standard users
      user:
        name: "{{ item.name }}"
        uid: "{{ item.uid }}"
        groups: "{{ item.groups }}"
        comment: "{{ item.comment }}"
        shell: "{{ item.shell }}"
        createhome: yes
        state: present
      loop: "{{ standard_users }}"

    # Create service accounts
    - name: Create service accounts
      user:
        name: "{{ item.name }}"
        system: yes
        shell: "{{ item.shell }}"
        home: "{{ item.home | default('/var/lib') }}"
        createhome: no
        state: present
      loop: "{{ service_accounts }}"

    # Set password for user
    - name: Set password for alice
      user:
        name: alice
        password: "{{ 'P@ssw0rd123' | password_hash('sha512') }}"
        update_password: on_create

    # Lock user account
    - name: Lock eve's account
      user:
        name: eve
        password_lock: lock

    # Unlock user account
    - name: Unlock charlie's account
      user:
        name: charlie
        password_lock: unlock

    # Remove user account
    - name: Remove olduser account
      user:
        name: olduser
        state: absent
        remove: yes

    # Ensure shadow group exists
    - name: Ensure shadow group exists
      group:
        name: shadow
        state: present

    # Create group with specific GID
    - name: Create group with specific GID
      group:
        name: developers
        system: yes
        gid: 3000

    # Add user to multiple groups
    - name: Add user to multiple groups
      user:
        name: alice
        groups:
          - wheel
          - docker
          - admins
        append: yes

    # Create home directory for user
    - name: Create home directory
      file:
        path: /home/{{ item.name }}
        state: directory
        owner: "{{ item.name }}"
        group: "{{ item.name }}"
        mode: "0755"
      loop:
        - name: alice
        - name: bob
        - name: charlie

    # Display user information
    - name: Display user information
      command: id {{ item }}
      register: user_info
      loop:
        - alice
        - bob
        - charlie
      changed_when: false

    - name: Display user information
      debug:
        msg: "{{ item.item }}: {{ user_info.results[item.index].stdout }}"
```

## Explanation of Each Playbook

### Playbook 1: Software Packages and Repositories

This playbook demonstrates package management using the `dnf` module. The `update_cache: yes` parameter forces a metadata refresh. The `state: present` parameter ensures packages are installed without upgrading. The `dnf_module` module handles DNF module streams for specific versions. The `copy` module creates custom repositories.

### Playbook 2: Services Management

This playbook demonstrates service management using the `systemd` module. `state: started` ensures the service is running now. `enabled: yes` ensures the service starts at boot. `state: reloaded` reloads the service configuration. The `daemon_reload: yes` parameter runs `systemctl daemon-reload` before making changes.

### Playbook 3: Firewall Rules

This playbook demonstrates firewall configuration using the `firewalld` module. `permanent: yes` writes rules to the persistent configuration. `immediate: yes` applies rules to the running firewall. Port ranges use the hyphen syntax. `toport` and `toaddr` configure port forwarding.

### Playbook 4: File Systems and Storage

This playbook demonstrates LVM and filesystem management. The `lvg` module creates volume groups. The `lvol` module creates logical volumes. The `filesystem` module formats devices. The `mount` module mounts filesystems and adds entries to `/etc/fstab`. The `when` conditions check device existence.

### Playbook 5: File Content Management

This playbook demonstrates file management using multiple modules. The `copy` module deploys static files. The `lineinfile` module manages individual lines. The `blockinfile` module manages multi-line blocks. The `template` module processes Jinja2 templates. The `file` module sets permissions.

### Playbook 6: Archiving

This playbook demonstrates archive operations. The `archive` module creates compressed archives with various formats. The `unarchive` module extracts archives. The `remote_src: yes` parameter indicates the archive exists on the managed node.

### Playbook 7: Task Scheduling

This playbook demonstrates task scheduling using `cron` and `systemd`. The `cron` module creates cron jobs with various schedules. The `special_time` parameter supports named schedules. The `systemd` module creates and enables timers.

### Playbook 8: Security Configuration

This playbook demonstrates security configuration. The `selinux` module sets SELinux mode. The `seboolean` module toggles SELinux booleans. The `sefcontext` module defines file contexts. The `lineinfile` module configures SSH parameters. The `pam` module configures PAM modules.

### Playbook 9: Users and Groups

This playbook demonstrates user and group management. The `group` module creates groups. The `user` module creates users with various attributes. The `password_hash` filter generates hashed passwords. The `password_lock` parameter locks or unlocks accounts.

## Variables and Templates

### Using Variables for RHCSA Automation

```yaml
---
- name: RHCSA automation with variables
  hosts: all
  become: yes
  vars:
    app_name: myapp
    app_port: 8080
    log_level: info
    max_connections: 100
  tasks:
    - name: Create directory
      file:
        path: /etc/{{ app_name }}
        state: directory

    - name: Deploy configuration
      copy:
        content: |
          [application]
          name = {{ app_name }}
          port = {{ app_port }}
          log_level = {{ log_level }}
        dest: /etc/{{ app_name }}/config.ini
```

### Using Templates for Configuration

```yaml
# Template: myapp.conf.j2
[application]
name = {{ app_name }}
port = {{ app_port }}
log_level = {{ log_level }}
environment = {{ environment | default('production') }}

{% if enable_ssl | default(false) %}
ssl_enabled = true
{% else %}
ssl_enabled = false
{% endif %}
```

## Verification Procedures

### Verify Package Installation

```bash
ansible all -m command -a "rpm -q httpd vim-enhanced wget"
ansible all -m dnf -a "list_installed httpd"
```

### Verify Service Status

```bash
ansible all -m command -a "systemctl is-active firewalld sshd httpd"
ansible all -m command -a "systemctl is-enabled firewalld sshd httpd"
```

### Verify Firewall Rules

```bash
ansible all -m command -a "firewall-cmd --list-ports"
ansible all -m command -a "firewall-cmd --list-services"
```

### Verify File Systems

```bash
ansible all -m command -a "df -hT /mnt/data"
ansible all -m command -a "vgs lvs pvs"
```

### Verify File Content

```bash
ansible all -m command -a "cat /etc/myapp/config.ini"
ansible all -m stat -a "path=/etc/myapp/config.ini"
```

### Verify Cron Jobs

```bash
ansible all -m command -a "crontab -l"
ansible all -m command -a "systemctl list-timers"
```

### Verify Users and Groups

```bash
ansible all -m command -a "id alice charlie"
ansible all -m command -a "getent group admins wheel"
```

## Troubleshooting

### Package Installation Failures

**Problem: Package not found**

```yaml
- name: Update package cache
  dnf:
    name: "*"
    state: present
    update_cache: yes

- name: Install package
  dnf:
    name: package_name
    state: present
```

### Service Management Issues

**Problem: Service fails to start**

```yaml
- name: Check dependencies
  command: rpm -q --queryformat '%{REQUIRES}\n' httpd
  register: dependencies

- name: Install dependencies
  dnf:
    name: "{{ dependencies.stdout_lines }}"
    state: present

- name: Start service
  systemd:
    name: httpd
    state: started
    enabled: yes
```

### File System Issues

**Problem: Mount fails**

```yaml
- name: Check device exists
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

## Real-World Automation Examples

### Complete RHCSA Automation

```yaml
---
- name: Complete RHCSA automation
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
    # Packages
    - name: Install httpd
      dnf:
        name: httpd
        state: present

    # Services
    - name: Start httpd
      systemd:
        name: httpd
        state: started
        enabled: yes

    # Firewall
    - name: Open HTTP port
      firewalld:
        port: "80/tcp"
        permanent: yes
        immediate: yes
        state: enabled

    # Users
    - name: Create myapp user
      user:
        name: myapp
        system: yes
        shell: /sbin/nologin

    # Files
    - name: Create document root
      file:
        path: /var/www/html
        state: directory
        owner: apache
        group: apache
        mode: "0755"

    # Cron
    - name: Create backup cron
      cron:
        name: "Daily backup"
        minute: "0"
        hour: "2"
        job: "/usr/local/bin/backup.sh"
        user: root
```

## RHCE Exam Notes

### Critical Exam Points

1. **Module parameters matter**: Know key parameters for each module.

2. **Idempotency is key**: Configure to state, not actions.

3. **Become is required**: Most RHCSA tasks need root.

4. **Facts help decisions**: Use facts for conditional automation.

5. **Handlers are essential**: Restart services after config changes.

6. **Tags enable selection**: Use tags for selective execution.

7. **Verify before execute**: Use `--check` to preview changes.

### Exam Workflow

1. Write playbook with modules
2. Add handlers for services
3. Use `become: yes`
4. Validate with `--syntax-check`
5. Preview with `--check`
6. Execute playbook

## Common Mistakes

### Mistake 1: Not Using State

```yaml
# WRONG - always executes
- name: Install package
  command: dnf install -y httpd

# CORRECT - idempotent
- name: Install package
  dnf:
    name: httpd
    state: present
```

### Mistake 2: Forgetting Become

```yaml
# WRONG - tasks run as non-root
- name: Install package
  dnf:
    name: httpd
    state: present

# CORRECT - escalate to root
- name: Configure servers
  hosts: all
  become: yes
  tasks:
    - name: Install package
      dnf:
        name: httpd
        state: present
```

## Chapter Summary

This chapter covered automating RHCSA tasks with Ansible modules:

- **Software packages**: Use `dnf` and `dnf_module` for package management.
- **Services**: Use `systemd` to start, stop, and enable services.
- **Firewall**: Use `firewalld` to configure firewall rules.
- **File systems**: Use `mount`, `lvg`, `lvol`, and `filesystem` for storage.
- **File content**: Use `copy`, `template`, `lineinfile`, and `blockinfile`.
- **Archiving**: Use `archive` and `unarchive` for backup operations.
- **Task scheduling**: Use `cron` and `systemd` timers.
- **Security**: Use `selinux`, `seboolean`, and `sefcontext`.
- **Users and groups**: Use `user` and `group` modules.
- **Verification**: Verify all configurations after automation.

## Quick Reference

| Task | Module | Key Parameters |
|---|---|---|
| Install package | `dnf` | `name`, `state: present` |
| Remove package | `dnf` | `name`, `state: absent` |
| Start service | `systemd` | `name`, `state: started`, `enabled: yes` |
| Open firewall | `firewalld` | `port`, `permanent: yes`, `state: enabled` |
| Create user | `user` | `name`, `state: present`, `createhome: yes` |
| Create group | `group` | `name`, `system: yes` |
| Mount filesystem | `mount` | `src`, `path`, `fstype`, `state: mounted` |
| Create cron | `cron` | `name`, `minute`, `hour`, `job` |
| Deploy file | `copy` | `src`, `dest`, `mode` |
| Deploy template | `template` | `src`, `dest`, `validate` |
| Create archive | `archive` | `path`, `dest`, `format` |
| Extract archive | `unarchive` | `src`, `dest`, `remote_src` |
| Set SELinux | `selinux` | `state: enforcing` |
| Set boolean | `seboolean` | `name`, `state`, `persistent: yes` |

## Review Questions

1. Which Ansible module is used to install software packages on RHEL systems?

2. What is the purpose of the `state: present` parameter in the `dnf` module?

3. How do you configure a service to start automatically at boot using Ansible?

4. What parameters are required to open a port in the firewall permanently?

5. Which module is used to create a new user account?

6. How do you mount a filesystem persistently with Ansible?

7. What is the purpose of the `cron` module?

8. How do you configure SELinux to be in enforcing mode?

9. Which module is used to deploy a Jinja2 template to a managed node?

10. How do you create a backup archive using Ansible?

## Answers

1. The `dnf` module is used to install software packages on RHEL systems. It can also remove packages, update packages, and manage repositories.

2. The `state: present` parameter ensures the package is installed. If the package is already installed, the task does nothing (idempotent). Use `state: absent` to remove a package.

3. Use the `systemd` module with `state: started` to start the service and `enabled: yes` to enable it at boot. For example: `systemd: { name: httpd, state: started, enabled: yes }`.

4. Use the `firewalld` module with `port: "80/tcp"`, `permanent: yes`, and `state: enabled`. The `permanent: yes` parameter writes the rule to the persistent configuration, and `state: enabled` applies it immediately.

5. The `user` module is used to create new user accounts. Key parameters include `name` for the username, `state: present` to create the user, and `createhome: yes` to create the home directory.

6. Use the `mount` module with `src` for the source device, `path` for the mount point, `fstype` for the filesystem type, and `state: mounted` to mount the filesystem. Add `dump: yes` and `passno: 2` for `/etc/fstab` entries.

7. The `cron` module creates, modifies, and removes cron jobs. Parameters include `name` for identification, `minute`, `hour`, `day`, `month`, and `weekday` for the schedule, and `job` for the command to run.

8. Use the `selinux` module with `state: enforcing` to set SELinux to enforcing mode. The `policy` parameter specifies the policy type (typically `targeted`).

9. The `template` module deploys Jinja2 templates to managed nodes. It processes templates with variable substitution and supports validation before deployment.

10. Use the `archive` module with `path` for the files to archive, `dest` for the destination, and `format` for the compression format (gz, bz2, xz, zip). The archive is created with the specified compression.
