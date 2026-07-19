# Chapter 10: Be Familiar with Visual Studio Code (VS Code) for Ansible Development


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


## Added Review: VS Code Is an Editor, Not the Source of Truth

VS Code helps you write cleaner YAML, but the exam result is still judged by files and system state.

Use VS Code for:

- YAML indentation visibility;
- Ansible extension hints;
- integrated terminal commands;
- Git staging and commits;
- development-container workflow practice;
- editing `ansible-navigator.yml`.

Still verify from the terminal:

```bash
ansible-playbook --syntax-check playbook.yml
ansible-navigator run playbook.yml -m stdout
ansible-navigator config -m stdout
```

Do not depend on a GUI-only workflow. Be able to complete the same task in a shell.


## Learning Objectives

By the end of this chapter, you will be able to:

- Install and configure Visual Studio Code for Ansible development
- Create and edit Ansible playbooks in VS Code
- Use VS Code extensions for Ansible development
- Configure ansible-navigator in VS Code
- Run playbooks using VS Code integrated terminals
- Use VS Code Source Control for Git integration
- Push playbooks to Git repositories from VS Code
- Run playbooks using Ansible development containers
- Configure VS Code settings for Ansible
- Debug playbooks using VS Code
- Use VS Code IntelliSense for Ansible YAML

## Concept Overview

Visual Studio Code is a lightweight, powerful code editor that has become the de facto standard for Ansible development. Its rich extension ecosystem, integrated terminal, and Git integration make it ideal for creating and managing Ansible playbooks.

### Why VS Code for Ansible

Without VS Code:

- Manual YAML syntax checking
- No integrated terminal for Ansible commands
- No built-in Git integration
- No IntelliSense for YAML
- No ansible-navigator integration
- Difficult debugging workflow

### How VS Code Integrates with Ansible

```
VS Code Integration:
─────────────────────────────────────────────────────────────────
1. Editor
   │
   ├── YAML IntelliSense
   │
   ├── Syntax highlighting
   │
   └── Error detection

2. Terminal
   │
   ├── Integrated terminal
   │
   ├── Run ansible-playbook
   │
   └── Run ansible-navigator

3. Source Control
   │
   ├── Git integration
   │
   ├── Commit changes
   │
   └── Push to remote

4. Extensions
   │
   ├── Ansible extension
   │
   ├── ansible-navigator
   │
   └── Docker (for dev containers)

5. Dev Containers
   │
   ├── Docker containers
   │
   ├── Pre-configured Ansible
   │
   └── Consistent environment
```

## Ansible Fundamentals Required

### VS Code Extensions for Ansible

| Extension | Purpose |
|---|---|
| **Ansible** | YAML validation, syntax checking, linting |
| **ansible-navigator** | Run ansible-navigator from VS Code |
| **Docker** | Manage Docker containers |
| **Remote - Containers** | Run VS Code inside containers |
| **GitLens** | Enhanced Git integration |
| **YAML** | YAML support and validation |

### VS Code Workspace Structure

```
ansible-project/
├── .devcontainer/
│   ├── devcontainer.json
│   ├── Dockerfile
│   └── docker-compose.yml
├── .vscode/
│   ├── settings.json
│   └── launch.json
├── ansible.cfg
├── inventory
├── playbooks/
│   └── playbook.yml
└── README.md
```

## Commands and Tools

```bash
# Install VS Code
# macOS
brew install --cask visual-studio-code

# Linux (deb)
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
wget https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
sudo apt-get install apt-transport-https
curl -sSL https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
sudo apt-get install visual-studio-code

# Windows
# Download from https://code.visualstudio.com/

# Install extensions in VS Code
# Extensions: Ctrl+Shift+X
# - Ansible by Red Hat
# - ansible-navigator by Red Hat
# - Docker by Microsoft
# - Remote - Containers by Microsoft

# Open workspace in VS Code
code .
code --new-window .

# Run ansible-playbook from VS Code terminal
ansible-playbook playbooks/playbook.yml

# Run ansible-navigator from VS Code terminal
ansible-navigator run playbooks/playbook.yml

# Check Git status from VS Code
git status

# Commit from VS Code Source Control panel
git add .
git commit -m "message"

# Push from VS Code Source Control panel
git push origin main

# Configure ansible-navigator in VS Code
# .vscode/settings.json
{
    "ansible-navigator.environment": true
}

# Run in dev container
# Press F1, type "Remote-Containers"
# Select "Reopen in Container"

# View Ansible lint errors
# Ansible extension shows errors in Problems panel
```

## Playbook Examples

### Example 1: VS Code Configuration for Ansible

```yaml
---
- name: Configure VS Code for Ansible development
  hosts: localhost
  connection: local
  become: true
  tasks:
    - name: Create .vscode directory
      file:
        path: /home/ansible/.vscode
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create VS Code settings.json
      copy:
        content: |
          {
            "ansible-lint.enabled": true,
            "ansible-lint.maxNumberOfProblems": 100,
            "ansible-navigator.environment": true,
            "editor.formatOnSave": true,
            "editor.rulers": [80, 120],
            "files.eol": "\n",
            "files.trimTrailingWhitespace": true,
            "files.insertFinalNewline": true,
            "files.trimFinalNewlines": true,
            "files.autoSave": "afterDelay",
            "yaml.schemas": {
              "https://raw.githubusercontent.com/ansible/ansible/dev-docs/rst/ansible-playbook.rst": "playbook.yml",
              "https://raw.githubusercontent.com/ansible/ansible/dev-docs/rst/ansible-role.rst": "roles/*.yml"
            },
            "editor.codeActionsOnSave": {
              "source.fixAll.eslint": "explicit"
            },
            "editor.quickSuggestions": {
              "strings": true
            },
            "files.exclude": {
              "**/.git": true,
              "**/.gitignore": true,
              "**/.vscode": false,
              "**/__pycache__": true,
              "**/*.pyc": true,
              "**/.ansible": true,
              "**/ansible.log": true,
              "**/ansible-tmp": true,
              "**/logs": true,
              "**/*.log": true
            },
            "editor.wordWrap": "on",
            "editor.tabSize": 2,
            "editor.insertSpaces": true,
            "files.trimFinalNewlines": true
          }
        dest: "/home/ansible/.vscode/settings.json"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create ansible-navigator configuration
      copy:
        content: |
          {
            "ansible-navigator.environment": true,
            "ansible-navigator.diff": true,
            "ansible-navigator.inventory": [
              "/home/ansible/inventory"
            ],
            "ansible-navigator.playbook": "",
            "ansible-navigator.mode": "stdout"
          }
        dest: "/home/ansible/.vscode/ansible-navigator.json"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create extensions.json for recommended extensions
      copy:
        content: |
          {
            "recommendations": [
              "ms-azuretools.vscode-docker",
              "redhat.ansible",
              "redhat.vscode-ansible-navigator",
              "ms-vscode.vscode-typescript-next",
              "ms-python.python",
              "ms-vscode.cpptools"
            ],
            "features": [
              "github",
              "docker",
              "remote-containers"
            ]
          }
        dest: "/home/ansible/.vscode/extensions.json"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create launch.json for debugging
      copy:
        content: |
          {
            "version": "0.2.0",
            "configurations": [
              {
                "name": "Run Playbook",
                "type": "bash",
                "request": "launch",
                "program": "${workspaceFolder}/playbooks/playbook.yml",
                "args": ["-vvv"],
                "cwd": "${workspaceFolder}"
              },
              {
                "name": "Run Playbook with Check",
                "type": "bash",
                "request": "launch",
                "program": "${workspaceFolder}/playbooks/playbook.yml",
                "args": ["--check", "--diff", "-vvv"],
                "cwd": "${workspaceFolder}"
              },
              {
                "name": "Run Playbook with Tags",
                "type": "bash",
                "request": "launch",
                "program": "${workspaceFolder}/playbooks/playbook.yml",
                "args": ["--tags", "packages"],
                "cwd": "${workspaceFolder}"
              }
            ]
          }
        dest: "/home/ansible/.vscode/launch.json"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create launch.json for ansible-navigator
      copy:
        content: |
          {
            "version": "0.2.0",
            "configurations": [
              {
                "name": "Run ansible-navigator",
                "type": "bash",
                "request": "launch",
                "program": "ansible-navigator",
                "args": ["run", "${workspaceFolder}/playbooks/playbook.yml"],
                "cwd": "${workspaceFolder}"
              }
            ]
          }
        dest: "/home/ansible/.vscode/launch-navigator.json"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Verify VS Code configuration
      stat:
        path: "/home/ansible/.vscode/settings.json"
      register: settings_stat

    - name: Display configuration status
      debug:
        msg: "VS Code configuration {{ 'created' if settings_stat.stat.exists else 'not created' }}"
```

### Example 2: Create Ansible Playbook in VS Code

```yaml
---
- name: Create Ansible playbook for VS Code
  hosts: localhost
  connection: local
  become: true
  tasks:
    - name: Create playbooks directory
      file:
        path: /home/ansible/projects/ansible-playbooks/playbooks
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create basic playbook
      copy:
        content: |
          ---
          - name: Configure basic system
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

              - name: Check if httpd is installed
                command: rpm -q httpd
                register: httpd_check
                failed_when: false
                changed_when: false

              - name: Install httpd if not present
                dnf:
                  name: httpd
                  state: present
                when: httpd_check.rc != 0

              - name: Start httpd service
                systemd:
                  name: httpd
                  state: started
                  enabled: yes
                when: httpd_check.rc == 0 or ansible_facts.packages.httpd.state == "present"

              - name: Open HTTP port in firewall
                firewalld:
                  port: "80/tcp"
                  permanent: yes
                  immediate: yes
                  state: enabled
                when: ansible_facts.services.firewalld.service is defined

        dest: "/home/ansible/projects/ansible-playbooks/playbooks/basic.yml"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create playbook with tags
      copy:
        content: |
          ---
          - name: Configure web server
            hosts: webservers
            become: true
            gather_facts: yes
            vars:
              app_name: myapp
              app_port: 8080
              log_level: info
            tasks:
              # Package installation
              - name: Update package cache
                dnf:
                  name: "*"
                  state: present
                  update_cache: yes
                tags: packages

              - name: Install httpd
                dnf:
                  name: httpd
                  state: present
                tags: packages

              # Service configuration
              - name: Start httpd
                systemd:
                  name: httpd
                  state: started
                  enabled: yes
                tags: services

              # Firewall configuration
              - name: Open HTTP port
                firewalld:
                  port: "80/tcp"
                  permanent: yes
                  immediate: yes
                  state: enabled
                tags: firewall

              # Verification
              - name: Verify httpd is running
                systemd:
                  name: httpd
                  state: started
                tags: verify

        dest: "/home/ansible/projects/ansible-playbooks/playbooks/webserver.yml"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create playbook with templates
      copy:
        content: |
          ---
          - name: Deploy configuration with templates
            hosts: all
            become: true
            gather_facts: yes
            vars:
              app_name: myapp
              app_port: 8080
              app_user: myapp
            tasks:
              - name: Create application directory
                file:
                  path: /etc/{{ app_name }}
                  state: directory
                  owner: root
                  group: root
                  mode: "0755"

              - name: Deploy configuration template
                template:
                  src: templates/config.j2
                  dest: /etc/{{ app_name }}/config.ini
                  owner: root
                  group: root
                  mode: "0644"
                  validate: "grep -q ^\[main\] %s"

              - name: Deploy systemd service
                template:
                  src: templates/systemd/service.j2
                  dest: /etc/systemd/system/{{ app_name }}.service
                  owner: root
                  group: root
                  mode: "0644"
                  validate: "systemd-analyze verify %s"
                notify: daemon reload

              - name: Create application user
                user:
                  name: "{{ app_user }}"
                  system: yes
                  shell: /sbin/nologin
                  home: /var/lib/{{ app_name }}

        dest: "/home/ansible/projects/ansible-playbooks/playbooks/deploy.yml"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create inventory file
      copy:
        content: |
          [webservers]
          servera.lab.example.com
          serverb.lab.example.com

          [dbservers]
          serverc.lab.example.com

          [production:children]
          webservers
          dbservers

          [all:vars]
          ansible_user=ansible
          ansible_become=yes
        dest: "/home/ansible/projects/ansible-playbooks/inventory"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create ansible.cfg
      copy:
        content: |
          [defaults]
          inventory = /home/ansible/projects/ansible-playbooks/inventory
          remote_user = ansible
          host_key_checking = false
          log_path = /home/ansible/projects/ansible-playbooks/logs/ansible.log
          forks = 10

          [privilege_escalation]
          become = true
          become_method = sudo
          become_user = root
          become_ask_pass = false

          [ssh_connection]
          pipelining = true
          ssh_args = -o ControlMaster=auto -o ControlPersist=60s

          [ansible_navigator]
          mode = stdout
          diff = true
          inventory = /home/ansible/projects/ansible-playbooks/inventory
        dest: "/home/ansible/projects/ansible-playbooks/ansible.cfg"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create .gitignore
      copy:
        content: |
          # Ansible
          .ansible/
          ansible_facts_cache/
          inventory.~*~
          ansible.log
          ansible-tmp/
          logs/

          # Python
          __pycache__/
          *.py[cod]
          *.pyo
          .coverage

          # OS
          .DS_Store
          Thumbs.db

          # Environment
          .env
          env/
        dest: "/home/ansible/projects/ansible-playbooks/.gitignore"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create README.md
      copy:
        content: |
          # Ansible Playbooks

          VS Code Ansible development project.

          ## Setup

          ```bash
          ansible-playbook playbooks/basic.yml
          ansible-playbook playbooks/webserver.yml
          ```

          ## VS Code Extensions

          - Ansible by Red Hat
          - ansible-navigator by Red Hat
          - Docker by Microsoft

          ## Git

          ```bash
          git add .
          git commit -m "message"
          git push origin main
          ```

        dest: "/home/ansible/projects/ansible-playbooks/README.md"
        owner: ansible
        group: ansible
        mode: "0644"
```

### Example 3: Push Playbooks to Git from VS Code

```yaml
---
- name: Push playbooks to Git repository from VS Code
  hosts: localhost
  connection: local
  become: true
  vars:
    project_dir: /home/ansible/projects/ansible-playbooks
    remote_repo: https://github.com/username/ansible-playbooks.git
  tasks:
    - name: Create Git repository
      file:
        path: "{{ project_dir }}"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Initialize Git repository
      command: git init
      args:
        chdir: "{{ project_dir }}"
      when: not (stat(path="{{ project_dir }}/.git").stat.exists)
      changed_when: false

    - name: Configure Git user
      lineinfile:
        path: /home/ansible/.gitconfig
        line: "{{ item }}"
        create: yes
      loop:
        - [user.name = "Ansible User", regex="^user.name"]
        - [user.email = ansible@example.com, regex="^user.email"]

    - name: Add remote origin
      lineinfile:
        path: "{{ project_dir }}/.git/config"
        line: "{{ item }}"
        create: yes
      loop:
        - [url = {{ remote_repo }}, regex="^\[remote \"origin\"\"\]"]
        - [fetch = {{ remote_repo }}, regex="^\s*fetch"]
        - [push = {{ remote_repo }}, regex="^\s*push"]

    - name: Add all files to Git
      command: git add .
      args:
        chdir: "{{ project_dir }}"

    - name: Commit changes
      command: git commit -m "Add initial Ansible playbooks"
      args:
        chdir: "{{ project_dir }}"

    - name: Pull latest changes
      command: git pull origin main
      args:
        chdir: "{{ project_dir }}"
      register: pull_result
      failed_when: false

    - name: Push to remote repository
      command: git push -u origin main
      args:
        chdir: "{{ project_dir }}"

    - name: Verify Git push
      command: git log --oneline --all
      args:
        chdir: "{{ project_dir }}"
      register: git_log
      changed_when: false

    - name: Display Git status
      command: git status
      args:
        chdir: "{{ project_dir }}"
      register: git_status
      changed_when: false

    - name: Display Git remote
      command: git remote -v
      args:
        chdir: "{{ project_dir }}"
      register: git_remote
      changed_when: false

    - name: Display push result
      debug:
        msg: |
          Repository: {{ project_dir }}
          Remote: {{ remote_repo }}
          Commits: {{ git_log.stdout_lines | length }}
          Push: {{ 'successful' if pull_result.rc == 0 else 'failed' }}
```

## Explanation of Each Playbook

### Playbook 1: VS Code Configuration

This playbook creates the VS Code configuration files needed for Ansible development. It creates `settings.json` with Ansible-specific settings, `ansible-navigator.json` for navigator configuration, `extensions.json` for recommended extensions, and `launch.json` for debugging. These files enable VS Code to work optimally with Ansible playbooks.

### Playbook 2: Create Playbooks

This playbook demonstrates creating various Ansible playbooks in VS Code: basic playbooks with simple tasks, playbooks with tags for selective execution, and playbooks with templates for dynamic configuration. It also creates the inventory file and `ansible.cfg` for the project.

### Playbook 3: Push to Git

This playbook demonstrates the complete Git workflow from VS Code: initializing a repository, configuring Git user, adding remote, committing changes, and pushing to a remote repository. It shows how VS Code's integrated Git integration works.

## Variables and Templates

### Using Variables in VS Code Configuration

```yaml
---
- name: VS Code configuration with variables
  hosts: localhost
  connection: local
  become: true
  vars:
    project_name: ansible-playbooks
    project_dir: /home/ansible/projects/{{ project_name }}
    vs_code_dir: "{{ project_dir }}/.vscode"
  tasks:
    - name: Create .vscode directory
      file:
        path: "{{ vs_code_dir }}"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create settings.json
      copy:
        content: |
          {
            "ansible-lint.enabled": true,
            "ansible-navigator.environment": true,
            "editor.formatOnSave": true
          }
        dest: "{{ vs_code_dir }}/settings.json"
```

### Using Templates for VS Code Configuration

```yaml
# Template: .vscode/settings.json.j2
{
  "ansible-lint.enabled": {{ ansible_lint_enabled | default(true) | lower }},
  "ansible-navigator.environment": {{ ansible_navigator_env | default(true) | lower }},
  "editor.formatOnSave": {{ editor_format_on_save | default(true) | lower }},
  "files.eol": "\n",
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "yaml.schemas": {
    "https://raw.githubusercontent.com/ansible/ansible/dev-docs/rst/ansible-playbook.rst": "playbook.yml"
  }
}
```

```yaml
# Playbook using template
---
- name: Create settings.json with template
  template:
    src: templates/.vscode/settings.json.j2
    dest: "{{ project_dir }}/.vscode/settings.json"
    owner: ansible
    group: ansible
    mode: "0644"
```

## Verification Procedures

### Verify VS Code Installation

```bash
# Check VS Code version
code --version

# Verify VS Code is in PATH
which code

# Check installed extensions
code --list-extensions
```

### Verify Playbook Syntax in VS Code

```bash
# Syntax check
ansible-playbook playbook.yml --syntax-check

# Lint playbook
ansible-lint playbook.yml
```

### Verify Git Integration

```bash
# Check Git status
git status

# Check Git remote
git remote -v

# Check Git history
git log --oneline
```

## Troubleshooting

### VS Code Extension Issues

**Problem: Extensions not loading**

```bash
# Solution: Check VS Code extensions
code --list-extensions

# Solution: Install extensions manually
code --install-extension ms-azuretools.vscode-docker
code --install-extension redhat.ansible
```

### Git Integration Issues

**Problem: Git authentication fails**

```bash
# Solution: Configure Git credentials
git config --global credential.helper store

# Solution: Use SSH keys
ssh-keygen -t ed25519
ssh-copy-id git@github.com
```

### ansible-navigator Issues

**Problem: ansible-navigator not found in VS Code**

```bash
# Solution: Install ansible-navigator
dnf install ansible-navigator

# Solution: Verify installation
ansible-navigator --version
```

## Real-World Automation Examples

### Complete VS Code Ansible Development Setup

```yaml
---
- name: Complete VS Code Ansible development setup
  hosts: localhost
  connection: local
  become: true
  vars:
    project_name: ansible-playbooks
    project_dir: /home/ansible/projects/{{ project_name }}
  tasks:
    # Create project directory
    - name: Create project directory
      file:
        path: "{{ project_dir }}"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    # Create .vscode directory
    - name: Create .vscode directory
      file:
        path: "{{ project_dir }}/.vscode"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    # Create .devcontainer directory
    - name: Create .devcontainer directory
      file:
        path: "{{ project_dir }}/.devcontainer"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    # Create VS Code settings
    - name: Create VS Code settings.json
      copy:
        content: |
          {
            "ansible-lint.enabled": true,
            "ansible-lint.maxNumberOfProblems": 100,
            "ansible-navigator.environment": true,
            "ansible-navigator.diff": true,
            "editor.formatOnSave": true,
            "editor.rulers": [80, 120],
            "files.eol": "\n",
            "files.trimTrailingWhitespace": true,
            "files.insertFinalNewline": true,
            "files.trimFinalNewlines": true,
            "files.autoSave": "afterDelay",
            "yaml.schemas": {
              "https://raw.githubusercontent.com/ansible/ansible/dev-docs/rst/ansible-playbook.rst": "playbook.yml"
            },
            "editor.codeActionsOnSave": {
              "source.fixAll.eslint": "explicit"
            },
            "editor.quickSuggestions": {
              "strings": true
            },
            "files.exclude": {
                "**/.git": true,
                "**/.gitignore": true,
                "**/__pycache__": true,
                "**/*.pyc": true,
                "**/.ansible": true,
                "**/ansible.log": true,
                "**/logs": true,
                "**/*.log": true
            },
            "editor.wordWrap": "on",
            "editor.tabSize": 2,
            "editor.insertSpaces": true
          }
        dest: "{{ project_dir }}/.vscode/settings.json"
        owner: ansible
        group: ansible
        mode: "0644"

    # Create .devcontainer configuration
    - name: Create devcontainer.json
      copy:
        content: |
          {
            "name": "Ansible Development Container",
            "build": {
              "dockerfile": "Dockerfile",
              "context": ..
            },
            "customizations": {
              "vscode": {
                "extensions": [
                  "ms-azuretools.vscode-docker",
                  "redhat.ansible",
                  "redhat.vscode-ansible-navigator"
                ]
              }
            },
            "features": {
              "git": "latest"
            },
            "remoteUser": "ansible"
          }
        dest: "{{ project_dir }}/.devcontainer/devcontainer.json"
        owner: ansible
        group: ansible
        mode: "0644"

    # Create Dockerfile for dev container
    - name: Create Dockerfile
      copy:
        content: |
          FROM registry.access.redhat.com/ubi9/ansible:latest

          USER root

          # Install VS Code dependencies
          RUN dnf install -y code

          # Create ansible user
          RUN useradd -m -s /bin/bash ansible

          # Set permissions
          RUN chown -R ansible:ansible /home/ansible

          USER ansible

          WORKDIR /home/ansible
        dest: "{{ project_dir }}/.devcontainer/Dockerfile"
        owner: ansible
        group: ansible
        mode: "0644"

    # Create project files
    - name: Create ansible.cfg
      copy:
        content: |
          [defaults]
          inventory = inventory
          remote_user = ansible
          host_key_checking = false
          forks = 10

          [privilege_escalation]
          become = true
          become_method = sudo
          become_user = root
          become_ask_pass = false

          [ssh_connection]
          pipelining = true
        dest: "{{ project_dir }}/ansible.cfg"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create inventory
      copy:
        content: |
          [webservers]
          servera.lab.example.com

          [all:vars]
          ansible_user=ansible
        dest: "{{ project_dir }}/inventory"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create playbook
      copy:
        content: |
          ---
          - name: Ansible development playbook
            hosts: all
            become: true
            gather_facts: yes
            tasks:
              - name: Display hostname
                debug:
                  msg: "Running on {{ inventory_hostname }}"

              - name: Ping all hosts
                ping:
        dest: "{{ project_dir }}/playbooks/playbook.yml"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create .gitignore
      copy:
        content: |
          # Ansible
          .ansible/
          ansible_facts_cache/
          inventory.~*~
          ansible.log
          ansible-tmp/
          logs/

          # Python
          __pycache__/
          *.pyc

          # OS
          .DS_Store
          Thumbs.db
        dest: "{{ project_dir }}/.gitignore"
        owner: ansible
        group: ansible
        mode: "0644"

    # Display summary
    - name: Display setup summary
      debug:
        msg: |
          Project: {{ project_name }}
          Directory: {{ project_dir }}
          VS Code Config: CREATED
          Dev Container: CREATED
          Playbook: CREATED
          Git: READY
```

## RHCE Exam Notes

### Critical Exam Points

1. **VS Code is the primary editor**: The exam expects you to create playbooks in VS Code.

2. **Git integration is essential**: Know how to use VS Code's Source Control panel for Git operations.

3. **ansible-navigator integration**: Configure ansible-navigator to run from VS Code terminal.

4. **Dev containers**: Understand how to use Ansible development containers for isolated execution.

5. **VS Code extensions**: Know which extensions are needed for Ansible development.

6. **Integrated terminal**: Use VS Code's integrated terminal for running Ansible commands.

7. **VS Code settings**: Configure VS Code for optimal Ansible development.

8. **Git workflow**: Know how to commit and push playbooks from VS Code.

### Exam Workflow

1. Open VS Code with project directory
2. Install required extensions
3. Configure ansible-navigator
4. Create playbooks
5. Run syntax check
6. Commit to Git
7. Push to remote

### Time Management

- 5 minutes: VS Code setup and extensions
- 10 minutes: Create playbooks
- 5 minutes: Git commit and push
- 5 minutes: Verify with ansible-navigator

## Common Mistakes

### Mistake 1: Not Installing Required Extensions

```bash
# WRONG - missing Ansible extension
code .

# CORRECT - install Ansible extension first
code --install-extension redhat.ansible
code .
```

### Mistake 2: Not Configuring Git User

```bash
# WRONG - commits will fail
git commit -m "message"

# CORRECT - configure user first
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
git commit -m "message"
```

### Mistake 3: Not Using Integrated Terminal

```bash
# WRONG - running in separate terminal
ansible-playbook playbook.yml

# CORRECT - use VS Code integrated terminal
# Terminal: Ctrl+` or View > Terminal
ansible-playbook playbook.yml
```

## Chapter Summary

This chapter covered VS Code for Ansible development:

- **VS Code installation** and configuration for Ansible development.
- **Extensions** for Ansible including ansible-lint, ansible-navigator, and Docker.
- **Playbook creation** with VS Code IntelliSense and syntax highlighting.
- **Git integration** using VS Code Source Control panel for commit and push.
- **ansible-navigator configuration** for integrated playbook execution.
- **Development containers** for isolated Ansible execution environments.
- **VS Code settings** for optimal Ansible development workflow.
- **Debugging** playbooks using VS Code launch configurations.

## Quick Reference

| VS Code Action | Shortcut/Command |
|---|---|
| Open workspace | `code .` |
| Extensions panel | `Ctrl+Shift+X` |
| Integrated terminal | `Ctrl+` ` or View > Terminal` |
| Source Control panel | `Ctrl+Shift+G` |
| Run task | `Ctrl+Shift+P` > "Tasks: Run Task" |
| Debug configuration | `F5` |
| Reopen in container | `F1` > "Remote-Containers: Reopen in Container" |
| Git commit | `Ctrl+Shift+G` > Commit input |
| Git push | `Ctrl+Shift+G` > Push button |
| ansible-navigator | `Ctrl+Shift+P` > "ansible-navigator: Run" |

## Review Questions

1. What VS Code extension is required for Ansible YAML validation?

2. How do you open a workspace in VS Code from the command line?

3. What is the purpose of the `.vscode/settings.json` file?

4. How do you run ansible-navigator from VS Code?

5. What is the difference between using the integrated terminal and a separate terminal for running Ansible commands?

6. How do you create a new Git branch in VS Code?

7. What is the purpose of Ansible development containers?

8. How do you configure Git user information for VS Code?

9. What VS Code feature provides YAML IntelliSense for Ansible playbooks?

10. How do you push changes to a remote repository from VS Code?

## Answers

1. The required VS Code extension for Ansible YAML validation is the **Ansible** extension by Red Hat. This extension provides YAML validation, syntax checking, linting, and Ansible-specific IntelliSense.

2. Use `code .` from the command line to open the current directory in VS Code. The `.` represents the current directory. You can also specify a path: `code /path/to/workspace`.

3. The `.vscode/settings.json` file contains VS Code workspace-specific settings. For Ansible development, it configures ansible-lint, ansible-navigator, editor formatting, YAML schemas, and file exclusions. These settings apply only to the workspace where the file is located.

4. You can run ansible-navigator from VS Code using the integrated terminal: type `ansible-navigator run playbook.yml`. You can also use VS Code commands: `Ctrl+Shift+P` > "ansible-navigator: Run" or use the ansible-navigator extension if installed.

5. The integrated terminal is part of VS Code and shares the same environment as the editor, making it easier to manage workflows and copy-paste code. A separate terminal runs independently and may have different environments. The integrated terminal also integrates with VS Code features like terminal tabs and shared history.

6. To create a new Git branch in VS Code: (1) Open the Source Control panel (`Ctrl+Shift+G`), (2) Click the "Create new branch" button or use the branch dropdown, (3) Enter the branch name, (4) Switch to the new branch. VS Code provides a visual interface for branch management.

7. Ansible development containers provide isolated execution environments with pre-configured Ansible, dependencies, and tools. They ensure consistent behavior across different development machines by running VS Code inside a Docker container with the exact same environment as production.

8. Configure Git user information using: `git config --global user.name "Your Name"` and `git config --global user.email "your.email@example.com"`. These settings apply globally to all Git repositories on the system. In VS Code, you can also configure these in the Source Control panel.

9. VS Code provides **YAML IntelliSense** for Ansible playbooks, powered by the Ansible extension. This includes syntax highlighting, error detection, auto-completion for Ansible modules, and validation of playbook structure. The YAML extension also provides basic YAML support.

10. To push changes from VS Code: (1) Open the Source Control panel (`Ctrl+Shift+G`), (2) Stage changes (if not already staged), (3) Click the "Push" button, or (4) Use the command palette: `Ctrl+Shift+P` > "Git: Push". VS Code handles the Git push command automatically.


---

# Chapter 10 — Reviewed Edition Closing Checkpoint

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
