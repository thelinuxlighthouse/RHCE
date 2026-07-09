# Chapter 14: Manage Content

## Learning Objectives

By the end of this chapter, you will be able to

- Create and use Jinja2 templates for dynamic configuration files
- Understand template variables, loops, and conditionals
- Use Jinja2 filters and tests in templates
- Create templates for customized configuration files
- Use Ansible Vault to protect sensitive data
- Encrypt and decrypt vault files
- Manage multiple vault files
- Use vault variables in playbooks
- Handle vault password management
- Securely store sensitive information

## Concept Overview

Content management in Ansible involves creating configuration files that can be customized for different environments and securely storing sensitive data. Templates enable dynamic configuration generation, while Ansible Vault provides encryption for sensitive information.

### Why Content Management Matters

Without proper content management:

- Configuration files are static and not reusable
- Secrets are hardcoded in playbooks
- Configuration drift occurs
- No environment-specific customization
- Security vulnerabilities from exposed secrets

### How Templates and Vault Work

```
Template Processing:
─────────────────────────────────────────────────────────────────
1. Template file (Jinja2)
   │
   ├── Variables substituted
   │
   ├── Loops executed
   │
   └── Conditionals evaluated

2. Rendered output
   │
   └── Written to managed node

Vault Encryption:
─────────────────────────────────────────────────────────────────
1. Sensitive data (passwords, keys)
   │
   ├── Ansible Vault encrypts
   │
   ├── Stored in encrypted file
   │
   └── Decrypted at runtime with password
```

## Ansible Fundamentals Required

### Jinja2 Template Basics

| Feature | Purpose | Example |
|---|---|---|
| Variables | Substitute values | `{{ app_name }}` |
| Loops | Iterate over items | `{% for item in items %}` |
| Conditionals | Branch logic | `{% if condition %}` |
| Filters | Transform values | `{{ value | upper }}` |
| Tests | Check conditions | `{{ value is defined }}` |
| Includes | Include other templates | `{% include 'file' %}` |

### Ansible Vault Structure

```
vault/
├── secrets.yml.encrypted     # Encrypted variable file
├── passwords.yml.encrypted   # Encrypted passwords
└── keys/
    └── vault_id.key          # Vault encryption key (optional)
```

## Commands and Tools

```bash
# Create encrypted vault file
ansible-vault create secrets.yml

# Edit encrypted vault file
ansible-vault edit secrets.yml

# View encrypted vault file
ansible-vault view secrets.yml

# Create encrypted vault file from existing file
ansible-vault encrypt secrets.yml

# Re-encrypt vault file with new password
ansible-vault rekey secrets.yml

# Decrypt vault file
ansible-vault decrypt secrets.yml

# Remove encryption from vault file
ansible-vault decrypt secrets.yml --output secrets.yml

# Diff encrypted vault file
ansible-vault diff secrets.yml

# Diff vault files (compare encrypted)
ansible-vault diff secrets.yml secrets_old.yml

# List vault files in directory
ansible-vault list /path/to/vaults/

# Remove vault encryption
ansible-vault encrypt --skip vaults/

# Set vault password in environment
export ANSIBLE_VAULT_PASSWORD_FILE=/path/to/password_file

# Use vault password file
ansible-vault encrypt secrets.yml --vault-password-file /path/to/password

# Create vault with specific ID
ansible-vault create secrets.yml --vault-id 'id1:key1,id2:key2'
```

## Playbook Examples

### Example 1: Create Jinja2 Template

```yaml
---
- name: Create Jinja2 template
  hosts: localhost
  connection: local
  become: yes
  tasks:
    - name: Create httpd configuration template
      copy:
        content: |
          Listen {{ httpd_port | default(80) }}
          ServerName {{ inventory_hostname }}
          DocumentRoot /var/www/{{ app_name }}
          ErrorLog /var/log/httpd/{{ app_name }}_error.log
          CustomLog /var/log/httpd/{{ app_name }}_access.log combined

          {% if enable_ssl | default(false) %}
          SSLEngine on
          SSLCertificateFile /etc/pki/tls/certs/{{ app_name }}.crt
          SSLCertificateKeyFile /etc/pki/tls/private/{{ app_name }}.key
          {% endif %}

          <VirtualHost *:{{ httpd_port | default(80) }}>
              ServerName {{ inventory_hostname }}
              ServerAlias www.{{ inventory_hostname }}
              DocumentRoot "/var/www/{{ app_name }}"

              <Directory "/var/www/{{ app_name }}">
                  Options Indexes FollowSymLinks
                  AllowOverride All
                  Require all granted
              </Directory>
          </VirtualHost>
        dest: /home/ansible/templates/httpd.conf.j2
        owner: root
        group: root
        mode: "0644"

    - name: Create nginx configuration template
      copy:
        content: |
          server {
              listen {{ nginx_port | default(80) }};
              server_name {{ server_name | default('localhost') }};

              root {{ document_root | default('/var/www/html') }};
              index index.html index.htm;

              location / {
                  try_files $uri $uri/ =404;
              }

              {% if enable_ssl | default(false) %}
              listen {{ nginx_port | default(80) }} ssl;
              ssl_certificate /etc/nginx/ssl/{{ server_name }}.crt;
              ssl_certificate_key /etc/nginx/ssl/{{ server_name }}.key;
              {% endif %}

              location /api {
                  proxy_pass http://127.0.0.1:{{ api_port | default(8080) }};
                  proxy_set_header Host $host;
                  proxy_set_header X-Real-IP $remote_addr;
              }
        dest: /home/ansible/templates/nginx.conf.j2
        owner: root
        group: root
        mode: "0644"

    - name: Create application configuration template
      copy:
        content: |
          [application]
          name = {{ app_name }}
          version = {{ app_version | default('1.0.0') }}
          port = {{ app_port | default(8080) }}
          log_level = {{ log_level | default('info') }}
          max_connections = {{ max_connections | default(100) }}
          environment = {{ environment | default('production') }}

          [database]
          host = {{ db_host | default('localhost') }}
          port = {{ db_port | default(5432) }}
          name = {{ db_name | default('myapp') }}
          user = {{ db_user | default('myapp') }}
          password = {{ db_password | default('changeme') }}

          [cache]
          enabled = {{ cache_enabled | default(true) | lower }}
          backend = {{ cache_backend | default('redis') }}
          host = {{ cache_host | default('localhost') }}
          port = {{ cache_port | default(6379) }}

          {% if environment == 'production' %}
          [security]
          ssl_enabled = true
          ssl_cert = /etc/ssl/certs/{{ app_name }}.crt
          ssl_key = /etc/ssl/private/{{ app_name }}.key
          {% else %}
          [security]
          ssl_enabled = false
          {% endif %}

          [logging]
          log_file = /var/log/{{ app_name }}/app.log
          max_size = {{ log_max_size | default('100M') }}
          rotate_count = {{ log_rotate_count | default(7) }}
        dest: /home/ansible/templates/app.conf.j2
        owner: root
        group: root
        mode: "0644"

    - name: Create systemd service template
      copy:
        content: |
          [Unit]
          Description={{ app_name }} Service
          After=network.target

          [Service]
          Type=simple
          User={{ app_user | default('myapp') }}
          Group={{ app_group | default('myapp') }}
          WorkingDirectory={{ working_dir | default('/var/lib/{{ app_name }}') }}
          ExecStart={{ exec_start | default('/usr/bin/{{ app_name }}') }}
          Restart=on-failure
          RestartSec=5
          EnvironmentFile=/etc/{{ app_name }}/config.ini

          [Install]
          WantedBy=multi-user.target
        dest: /home/ansible/templates/systemd.service.j2
        owner: root
        group: root
        mode: "0644"

    - name: Verify templates created
      stat:
        path: "{{ item }}"
      loop:
        - /home/ansible/templates/httpd.conf.j2
        - /home/ansible/templates/nginx.conf.j2
        - /home/ansible/templates/app.conf.j2
        - /home/ansible/templates/systemd.service.j2
      register: template_stats
      changed_when: false

    - name: Display template status
      debug:
        msg: "{{ 'Template created' if item.stat.exists else 'Template not found' }}"
      loop: "{{ template_stats.results }}"
      loop_control:
        label: "{{ item.path }}"
```

### Example 2: Use Template in Playbook

```yaml
---
- name: Use Jinja2 template in playbook
  hosts: all
  become: yes
  gather_facts: yes
  vars:
    app_name: myapp
    app_port: 8080
    app_user: myapp
    app_group: myapp
    enable_ssl: false
    log_max_size: 100M
    log_rotate_count: 7
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

    # Deploy configuration template
    - name: Deploy application configuration
      template:
        src: templates/app.conf.j2
        dest: /etc/{{ app_name }}/config.ini
        owner: root
        group: root
        mode: "0644"
        validate: "grep -q ^\[application\] %s"

    # Deploy httpd configuration template
    - name: Deploy httpd configuration
      template:
        src: templates/httpd.conf.j2
        dest: /etc/httpd/conf.d/{{ app_name }}.conf
        owner: root
        group: root
        mode: "0644"
        validate: "httpd -t -f %s"
      notify: restart httpd

    # Deploy systemd service template
    - name: Deploy systemd service
      template:
        src: templates/systemd.service.j2
        dest: /etc/systemd/system/{{ app_name }}.service
        owner: root
        group: root
        mode: "0644"
        validate: "systemd-analyze verify %s"
      notify: daemon reload

    # Deploy nginx configuration template
    - name: Deploy nginx configuration
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/conf.d/{{ app_name }}.conf
        owner: root
        group: root
        mode: "0644"
        validate: "nginx -t -f %s"
      notify: restart nginx

    # Create log file
    - name: Create log file
      file:
        path: /var/log/{{ app_name }}/app.log
        state: touch
        owner: "{{ app_user }}"
        group: "{{ app_group }}"
        mode: "0640"

    # Verify template rendering
    - name: Display rendered configuration
      command: cat /etc/{{ app_name }}/config.ini
      register: rendered_config
      changed_when: false

    - name: Display rendered configuration
      debug:
        msg: "{{ rendered_config.stdout }}"

  handlers:
    - name: restart httpd
      systemd:
        name: httpd
        state: restarted

    - name: restart nginx
      systemd:
        name: nginx
        state: restarted

    - name: daemon reload
      systemd:
        daemon_reload: yes
```

### Example 3: Advanced Template Features

```yaml
---
- name: Advanced template features
  hosts: localhost
  connection: local
  become: yes
  tasks:
    - name: Create advanced template with loops
      copy:
        content: |
          # Services Configuration
          {% for service in services %}
          [{{ service.name }}]
          enabled = {{ service.enabled | default(true) | lower }}
          port = {{ service.port | default(80) }}
          protocol = {{ service.protocol | default('tcp') }}
          {% if service.description %}
          description = {{ service.description }}
          {% endif %}
          {% endfor %}

          # Database Configuration
          {% for db in databases %}
          [{{ db.name }}]
          host = {{ db.host | default('localhost') }}
          port = {{ db.port | default(5432) }}
          name = {{ db.name }}
          user = {{ db.user | default('dbuser') }}
          {% if db.ssl_enabled | default(false) %}
          ssl_enabled = true
          ssl_cert = {{ db.ssl_cert | default('/etc/ssl/certs/db.crt') }}
          {% endif %}
          {% endfor %}

          {% if environment == 'production' %}
          # Production Settings
          [production]
          log_level = error
          max_connections = 1000
          cache_enabled = true
          {% else %}
          # Development Settings
          [development]
          log_level = debug
          max_connections = 100
          cache_enabled = false
          {% endif %}

          # Contact Information
          {% for contact in contacts %}
          contact_{{ loop.index }} = {{ contact.email }}
          {% endfor %}
        dest: /home/ansible/templates/advanced.conf.j2
        owner: root
        group: root
        mode: "0644"

    - name: Create template with includes
      copy:
        content: |
          # Main Configuration
          {% include 'header.conf' %}

          [main]
          name = {{ app_name }}
          version = {{ app_version }}
          {% include 'main_config.conf' %}

          {% include 'database.conf' %}

          {% if enable_ssl | default(false) %}
          {% include 'ssl_config.conf' %}
          {% endif %}

          {% include 'footer.conf' %}
        dest: /home/ansible/templates/main.conf.j2
        owner: root
        group: root
        mode: "0644"

    - name: Create template with conditionals
      copy:
        content: |
          {% if ansible_os_family == 'RedHat' %}
          # Red Hat Configuration
          package_manager = dnf
          {% elif ansible_os_family == 'Debian' %}
          # Debian Configuration
          package_manager = apt
          {% else %}
          # Unknown OS Configuration
          package_manager = unknown
          {% endif %}

          {% if ansible_architecture == 'x86_64' %}
          architecture = x86_64
          {% elif ansible_architecture == 'aarch64' %}
          architecture = aarch64
          {% else %}
          architecture = {{ ansible_architecture }}
          {% endif %}

          {% if enable_monitoring | default(true) %}
          [monitoring]
          enabled = true
          metrics_port = 9090
          {% else %}
          [monitoring]
          enabled = false
          {% endif %}

          {% if environment | default('production') == 'production' %}
          [security]
          ssl_enabled = true
          audit_logging = true
          {% else %}
          [security]
          ssl_enabled = false
          audit_logging = false
          {% endif %}
        dest: /home/ansible/templates/os_config.j2
        owner: root
        group: root
        mode: "0644"

    - name: Create template with filters
      copy:
        content: |
          # Application Information
          name = {{ app_name | upper }}
          version = {{ app_version | default('1.0.0') }}
          environment = {{ environment | default('production') }}

          # Host Information
          hostname = {{ inventory_hostname }}
          short_hostname = {{ inventory_hostname_short | default(inventory_hostname) }}
          distribution = {{ ansible_distribution }}
          version = {{ ansible_distribution_version }}
          codename = {{ ansible_distribution_codename | default('N/A') }}

          # Network Information
          primary_ip = {{ ansible_default_ipv4.address | default('N/A') }}
          all_ips = {{ ansible_all_ipv4_addresses | join(', ') | default('N/A') }}

          # Date and Time
          current_date = {{ ansible_date_time.date }}
          current_time = {{ ansible_date_time.time }}
          iso_timestamp = {{ ansible_date_time.iso8601 }}

          # System Information
          processor_cores = {{ ansible_processor_vcpus }}
          total_memory = {{ ansible_memtotal_mb }} MB
          total_disk = {{ ansible_disksize }} bytes
        dest: /home/ansible/templates/info.conf.j2
        owner: root
        group: root
        mode: "0644"

    - name: Verify template created
      stat:
        path: /home/ansible/templates/advanced.conf.j2
      register: template_stat

    - name: Display template status
      debug:
        msg: "Advanced template {{ 'created' if template_stat.stat.exists else 'not created' }}"
```

### Example 4: Ansible Vault for Sensitive Data

```yaml
---
- name: Manage content with Ansible Vault
  hosts: localhost
  connection: local
  become: yes
  tasks:
    - name: Create encrypted secrets file
      command: ansible-vault create secrets.yml
      args:
        creates: /home/ansible/secrets.yml

    - name: Edit encrypted secrets file
      command: ansible-vault edit secrets.yml
      args:
        creates: /home/ansible/secrets.yml
      environment:
        ANSIBLE_VAULT_PASSWORD_FILE: /home/ansible/.vault_password

    - name: Encrypt existing file
      command: ansible-vault encrypt /home/ansible/passwords.yml
      args:
        creates: /home/ansible/passwords.yml.encrypted

    - name: Re-encrypt vault file with new password
      command: ansible-vault rekey secrets.yml
      args:
        creates: /home/ansible/secrets.yml

    - name: View encrypted vault file
      command: ansible-vault view secrets.yml
      register: vault_content
      changed_when: false

    - name: Display vault content (encrypted)
      debug:
        msg: "Vault file exists: {{ vault_content.rc == 0 }}"

    - name: Create vault with specific ID
      command: ansible-vault create secrets.yml --vault-id 'id1:${ANSIBLE_VAULT_PASSWORD_1}'
      args:
        creates: /home/ansible/secrets.yml

    - name: List vault files
      command: ansible-vault list /home/ansible/
      register: vault_list
      changed_when: false

    - name: Display vault files
      debug:
        msg: "{{ vault_list.stdout_lines }}"

    - name: Create encrypted variable file
      template:
        src: templates/secrets.j2
        dest: /home/ansible/secrets.yml
        mode: "0600"
      args:
        validate: "ansible-vault encrypt --output %s"

    - name: Decrypt vault file
      command: ansible-vault decrypt secrets.yml --output /tmp/secrets.yml
      args:
        creates: /tmp/secrets.yml
      environment:
        ANSIBLE_VAULT_PASSWORD_FILE: /home/ansible/.vault_password

    - name: Display decrypted content
      command: cat /tmp/secrets.yml
      register: decrypted_content
      changed_when: false

    - name: Create password file for vault
      copy:
        content: "{{ vault_password }}"
        dest: /home/ansible/.vault_password
        owner: root
        group: root
        mode: "0600"

    - name: Create encrypted database credentials
      template:
        src: templates/db_credentials.j2
        dest: /home/ansible/db_credentials.yml
        mode: "0600"
      args:
        validate: "ansible-vault encrypt --output %s"

    - name: Verify vault encryption
      stat:
        path: /home/ansible/secrets.yml
      register: vault_file

    - name: Display vault status
      debug:
        msg: "Vault file {{ 'encrypted' if vault_file.stat.exists and vault_file.stat.mode | int >= 0600 else 'not encrypted' }}"
```

### Example 5: Complete Content Management

```yaml
---
- name: Complete content management with templates and vault
  hosts: all
  become: yes
  gather_facts: yes
  vars:
    app_name: myapp
    app_port: 8080
    app_user: myapp
    app_group: myapp
    database:
      host: localhost
      port: 5432
      name: myapp_db
      user: myapp
      password: "{{ vault_db_password }}"
    redis:
      host: localhost
      port: 6379
      password: "{{ vault_redis_password }}"
    services:
      - name: httpd
        enabled: true
        port: 80
      - name: nginx
        enabled: true
        port: 8080
      - name: postgresql
        enabled: true
        port: 5432
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

    # Deploy configuration template
    - name: Deploy application configuration
      template:
        src: templates/app.conf.j2
        dest: /etc/{{ app_name }}/config.ini
        owner: root
        group: root
        mode: "0644"
        validate: "grep -q ^\[application\] %s"

    # Deploy httpd configuration
    - name: Deploy httpd configuration
      template:
        src: templates/httpd.conf.j2
        dest: /etc/httpd/conf.d/{{ app_name }}.conf
        owner: root
        group: root
        mode: "0644"
        validate: "httpd -t -f %s"
      notify: restart httpd

    # Deploy systemd service
    - name: Deploy systemd service
      template:
        src: templates/systemd.service.j2
        dest: /etc/systemd/system/{{ app_name }}.service
        owner: root
        group: root
        mode: "0644"
        validate: "systemd-analyze verify %s"
      notify: daemon reload

    # Create user and group
    - name: Create application group
      group:
        name: "{{ app_group }}"
        system: yes

    - name: Create application user
      user:
        name: "{{ app_user }}"
        system: yes
        shell: /sbin/nologin
        home: /var/lib/{{ app_name }}
        group: "{{ app_group }}"

    # Deploy database configuration with vault
    - name: Deploy database configuration
      template:
        src: templates/database.conf.j2
        dest: /etc/{{ app_name }}/database.conf
        owner: root
        group: root
        mode: "0600"
      no_log: true

    # Deploy service configuration with loops
    - name: Deploy service configurations
      template:
        src: templates/services.conf.j2
        dest: /etc/{{ app_name }}/services.conf
        owner: root
        group: root
        mode: "0644"

    # Deploy advanced configuration
    - name: Deploy advanced configuration
      template:
        src: templates/advanced.conf.j2
        dest: /etc/{{ app_name }}/advanced.conf
        owner: root
        group: root
        mode: "0644"

    # Verify deployment
    - name: Verify application configuration
      stat:
        path: /etc/{{ app_name }}/config.ini
      register: config_stat

    - name: Verify database configuration
      stat:
        path: /etc/{{ app_name }}/database.conf
      register: db_stat

    - name: Display deployment status
      debug:
        msg: |
          Application config: {{ 'deployed' if config_stat.stat.exists else 'not deployed' }}
          Database config: {{ 'deployed' if db_stat.stat.exists else 'not deployed' }}
          Services config: {{ 'deployed' if 'services.conf' in ['/etc/' | concat(['/etc/' + app_name + '/'])] else 'not deployed' }}
```

## Explanation of Each Playbook

### Playbook 1: Create Jinja2 Template

This playbook demonstrates creating various Jinja2 templates with different features. It creates httpd configuration, nginx configuration, application configuration, and systemd service templates. The templates include variables, conditionals, loops, and includes.

### Playbook 2: Use Template in Playbook

This playbook demonstrates using templates in a real automation scenario. It creates directories, deploys templates for configuration files, manages users and groups, and verifies deployment. It shows how templates integrate with other Ansible modules.

### Playbook 3: Advanced Template Features

This playbook demonstrates advanced Jinja2 features including loops with conditionals, includes for modular templates, OS-specific conditionals, architecture detection, and various filters for transforming values.

### Playbook 4: Ansible Vault

This playbook demonstrates creating, editing, encrypting, and managing Ansible Vault files. It shows how to use vault passwords, rekey vault files, and decrypt vault content. It emphasizes secure storage of sensitive data.

### Playbook 5: Complete Content Management

This comprehensive playbook combines templates and vault for complete content management. It demonstrates variable substitution, loops, conditionals, and secure storage of sensitive data like database passwords.

## Variables and Templates

### Template Variables

```yaml
# Playbook variables for templates
vars:
  app_name: myapp
  app_port: 8080
  app_user: myapp
  enable_ssl: true
  log_max_size: 100M
```

### Jinja2 Filters

```jinja2
{{ value | default('default_value') }}        # Default value
{{ value | upper }}                          # Uppercase
{{ value | lower }}                          # Lowercase
{{ value | capitalize }}                     # Capitalize first letter
{{ value | length }}                         # String length
{{ value | join(', ') }}                     # Join list items
{{ value | sort }}                           # Sort list
{{ value | map(attribute='name') | list }}   # Map attribute to list
{{ value | bool }}                           # Convert to boolean
{{ value | ternary(true_value, false_value) }} # Ternary operator
```

### Jinja2 Tests

```jinja2
{{ value is defined }}                       # Value is defined
{{ value is not defined }}                   # Value is not defined
{{ value is string }}                        # Value is a string
{{ value is integer }}                       # Value is an integer
{{ value is iterable }}                      # Value is iterable
{{ value is sequence }}                      # Value is a sequence
{{ value is mapping }}                       # Value is a mapping
{{ value is sameas(other_value) }}           # Values are the same object
{{ value is iterable }}                      # Value is iterable
{{ value is not iterable }}                  # Value is not iterable
```

### Jinja2 Includes

```jinja2
{% include 'header.conf' %}                  # Include file
{% include 'header.conf' with context %}     # Include with parent context
{% include 'config_' + section + '.conf' %}  # Dynamic include
```

## Verification Procedures

### Verify Template Creation

```bash
# Check template exists
ls -la /home/ansible/templates/*.j2

# Check template content
cat /home/ansible/templates/app.conf.j2

# Check template syntax
python3 -c "import jinja2; jinja2.Environment().from_string(open('/home/ansible/templates/app.conf.j2').read())"
```

### Verify Template Deployment

```bash
# Check deployed file exists
ls -la /etc/myapp/config.ini

# Check file content
cat /etc/myapp/config.ini

# Check file permissions
stat /etc/myapp/config.ini
```

### Verify Vault Encryption

```bash
# List vault files
ansible-vault list /home/ansible/

# Check file is encrypted
file /home/ansible/secrets.yml

# Check file permissions
ls -la /home/ansible/secrets.yml
```

## Troubleshooting

### Template Variable Not Substituted

**Problem: Variables not replaced in template**

```bash
# Solution: Check variable definition
ansible-playbook playbook.yml -e "app_port=8080"

# Solution: Verify template syntax
cat templates/app.conf.j2

# Solution: Check variable name matches
{{ app_port }} vs app_port: 8080
```

### Vault Password Issues

**Problem: Vault password not accepted**

```bash
# Solution: Set vault password in environment
export ANSIBLE_VAULT_PASSWORD_FILE=/path/to/password

# Solution: Use --vault-password-file
ansible-playbook playbook.yml --vault-password-file /path/to/password

# Solution: Create password file
echo "password" > /path/to/password
chmod 600 /path/to/password
```

### Template Validation Fails

**Problem: Template validation fails**

```yaml
# Solution: Use correct validate command
template:
  src: templates/config.j2
  dest: /etc/config
  validate: "httpd -t -f %s"  # For httpd
  validate: "nginx -t -f %s"  # For nginx
```

## Real-World Automation Examples

### Complete Application Deployment

```yaml
---
- name: Complete application deployment
  hosts: all
  become: yes
  gather_facts: yes
  vars:
    app_name: myapp
    app_port: 8080
    db_password: "{{ vault_db_password }}"
  tasks:
    # Deploy all configuration files
    - name: Deploy application config
      template:
        src: templates/app.conf.j2
        dest: /etc/{{ app_name }}/config.ini

    - name: Deploy database config
      template:
        src: templates/database.conf.j2
        dest: /etc/{{ app_name }}/database.conf

    - name: Deploy service config
      template:
        src: templates/services.conf.j2
        dest: /etc/{{ app_name }}/services.conf

    - name: Deploy nginx config
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/conf.d/{{ app_name }}.conf

    - name: Deploy systemd service
      template:
        src: templates/systemd.service.j2
        dest: /etc/systemd/system/{{ app_name }}.service
```

## RHCE Exam Notes

### Critical Exam Points

1. **Templates are essential**: Use `template` module for dynamic configuration.

2. **Vault protects secrets**: Always use vault for passwords and sensitive data.

3. **Variables enable customization**: Use variables for environment-specific values.

4. **Handlers restart services**: Use handlers after deploying service configurations.

5. **Validation prevents errors**: Always use `validate` parameter with templates.

6. **Vault ID supports multiple passwords**: Use `--vault-id` for multiple vault passwords.

7. **File permissions matter**: Set restrictive permissions on vault files.

### Exam Workflow

1. Create templates with variables
2. Create vault files for secrets
3. Deploy templates with `template` module
4. Use vault variables in playbooks
5. Validate and test

## Common Mistakes

### Mistake 1: Not Using Vault for Secrets

```yaml
# WRONG - secrets in plaintext
- name: Deploy config
  template:
    src: config.j2
    dest: /etc/config
  vars:
    db_password: "secret123"

# CORRECT - use vault
- name: Deploy config
  template:
    src: config.j2
    dest: /etc/config
  vars:
    db_password: "{{ vault_db_password }}"
```

### Mistake 2: Wrong Template Variable Name

```yaml
# Template uses: {{ app_port }}
# Playbook defines: http_port: 8080

# CORRECT - use correct variable
- name: Deploy
  template:
    src: config.j2
    dest: /etc/config
  vars:
    app_port: 8080
```

## Chapter Summary

This chapter covered content management with templates and vault:

- **Jinja2 templates** enable dynamic configuration generation with variables, loops, conditionals, filters, and tests.
- **Template module** deploys templates to managed nodes with validation.
- **Ansible Vault** encrypts sensitive data like passwords and API keys.
- **Vault commands** include create, edit, view, encrypt, rekey, decrypt, and list.
- **Vault variables** use `vault_variable_name` syntax in playbooks.
- **Multiple vault passwords** supported with `--vault-id` parameter.
- **File permissions** must be restrictive for vault files.

## Quick Reference

| Command | Purpose |
|---|---|
| `ansible-vault create file.yml` | Create encrypted file |
| `ansible-vault edit file.yml` | Edit encrypted file |
| `ansible-vault view file.yml` | View encrypted file |
| `ansible-vault encrypt file.yml` | Encrypt existing file |
| `ansible-vault rekey file.yml` | Re-encrypt with new password |
| `ansible-vault decrypt file.yml` | Decrypt file |
| `ansible-vault list /path/` | List vault files |
| `export ANSIBLE_VAULT_PASSWORD_FILE=/path` | Set vault password |
| `--vault-password-file /path` | Specify vault password |
| `--vault-id 'id1:pass1,id2:pass2'` | Multiple vault passwords |

## Review Questions

1. What is the purpose of Jinja2 templates in Ansible?

2. How do you create an encrypted vault file using Ansible Vault?

3. What is the difference between `copy` module and `template` module?

4. How do you use Ansible Vault to encrypt an existing file?

5. What command do you use to view the contents of an encrypted vault file?

6. How do you specify a vault password when running a playbook?

7. What is the purpose of the `validate` parameter in the `template` module?

8. How do you re-encrypt a vault file with a new password?

9. What Jinja2 filter would you use to provide a default value for a variable?

10. How do you create a vault file with multiple passwords?

## Answers

1. Jinja2 templates enable dynamic configuration generation by allowing variable substitution, loops, conditionals, and other template features. They produce configuration files that are customized based on playbook variables and facts, making automation more flexible and reusable.

2. Use `ansible-vault create filename.yml` to create an encrypted vault file. This command prompts for a vault password and encrypts the file. The file is created with the `.yml` extension and is encrypted by default.

3. The `copy` module deploys static files from the control node to managed nodes without processing. The `template` module deploys Jinja2 templates that are processed with variable substitution before writing to the managed node. Use `copy` for static content and `template` for dynamic content.

4. Use `ansible-vault encrypt filename.yml` to encrypt an existing file. This command prompts for a vault password and encrypts the file in place. The file becomes encrypted and can only be read by decrypting it with the vault password.

5. Use `ansible-vault view filename.yml` to view the contents of an encrypted vault file. This command prompts for the vault password and displays the decrypted contents to the terminal. The content is not saved, only displayed.

6. Use the `--vault-password-file` parameter when running a playbook: `ansible-playbook playbook.yml --vault-password-file /path/to/password`. Alternatively, set the `ANSIBLE_VAULT_PASSWORD_FILE` environment variable to point to a password file.

7. The `validate` parameter runs a command to check the syntax or validity of a template before it is deployed. The `%s` placeholder is replaced with the path to a temporary file. This prevents invalid configurations from being deployed.

8. Use `ansible-vault rekey filename.yml` to re-encrypt a vault file with a new password. This prompts for the old password and then the new password. The file is re-encrypted with the new password while maintaining the same content.

9. Use the `default` filter: `{{ variable | default('default_value') }}`. This provides the default value if the variable is undefined or null. For example: `{{ app_port | default(8080) }}`.

10. Use the `--vault-id` parameter with multiple password specifications: `ansible-vault create file.yml --vault-id 'id1:password1,id2:password2'`. This creates a vault file that can be decrypted with either password, identified by the IDs.
