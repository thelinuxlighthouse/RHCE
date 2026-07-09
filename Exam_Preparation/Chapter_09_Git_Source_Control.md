# Chapter 9: Perform Basic Source Control Operations Using Git

## Learning Objectives

By the end of this chapter, you will be able to:

- Understand the purpose of version control and Git
- Clone Git repositories for Ansible playbooks
- Create new Git repositories
- Add files to Git repositories
- Commit changes with meaningful messages
- Push changes to remote repositories
- Pull changes from remote repositories
- Create and manage Git branches
- Resolve basic Git conflicts
- Configure Git for Ansible development
- Use Git in VS Code for Ansible development
- Run playbooks using Ansible development containers

## Concept Overview

Version control systems track changes to files over time, allowing you to revert to previous versions, collaborate with others, and maintain an audit trail. Git is the most popular distributed version control system, making it ideal for Ansible automation.

### Why Git Matters for Ansible

Without Git:

- No version history of playbooks
- Cannot revert failed changes
- No collaboration support
- No audit trail of changes
- Difficult to track who made what changes
- No rollback capability

### How Git Works

```
Git Workflow:
─────────────────────────────────────────────────────────────────
1. Local Repository
   │
   ├── Working Directory (uncommitted files)
   │
   ├── Staging Area (git add)
   │
   └── Local Repository (git commit)
         │
         └── .git directory (object database)

2. Remote Repository (GitHub, GitLab, Bitbucket)
   │
   ├── Origin/main (default branch)
   │
   └── Branches and Tags

3. Git Operations
   │
   ├── git clone - Create local copy from remote
   ├── git pull - Fetch and merge from remote
   ├── git push - Upload to remote
   ├── git add - Stage files for commit
   ├── git commit - Create snapshot
   └── git branch - Manage branches
```

## Ansible Fundamentals Required

### Git Repository Structure

```
project/
├── .git/                    # Git metadata (hidden)
├── ansible.cfg              # Ansible configuration
├── inventory                # Host inventory
├── playbooks/               # Playbook directory
│   ├── setup.yml
│   ├── configure.yml
│   └── deploy.yml
├── roles/                   # Ansible roles
│   ├── webserver/
│   │   ├── tasks/
│   │   ├── handlers/
│   │   └── templates/
│   └── database/
├── templates/               # Jinja2 templates
├── vars/                    # Variable files
├── requirements.yml         # Role/collection requirements
└── README.md
```

### Git Configuration

```bash
# Configure Git user
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Configure Git editor
git config --global core.editor "code --wait"  # VS Code

# Configure Git to use SSH for remote
git config --global url."ssh://git@github.com/".insteadOf "https://github.com/"
```

## Commands and Tools

```bash
# Clone repository
git clone https://github.com/user/repo.git
git clone git@github.com:user/repo.git
git clone --depth 1 https://github.com/user/repo.git  # Shallow clone

# Create repository
mkdir project
cd project
git init
git init --bare

# Add files
git add .
git add playbook.yml
git add -A
git add -u  # Update tracked files

# Commit changes
git commit -m "Add initial playbook"
git commit -m "Fix typo in playbook"
git commit -am "Commit with add"

# View status
git status
git status --short

# View history
git log
git log --oneline
git log --graph --oneline --all
git log -p  # Show patches

# Push to remote
git push origin main
git push origin main --force  # Force push (use with caution)
git push origin main --set-upstream  # First push

# Pull from remote
git pull origin main
git pull --rebase  # Rebase instead of merge

# Create branches
git branch feature-name
git branch -d feature-name  # Delete branch
git branch -D feature-name  # Force delete

# Switch branches
git checkout feature-name
git checkout -b feature-name  # Create and switch

# Merge branches
git merge feature-name
git merge --no-ff feature-name  # Create merge commit

# View diff
git diff  # Staged changes
git diff HEAD  # All changes
git diff file.yml  # File diff

# Stash changes
git stash
git stash pop
git stash list

# Remote operations
git remote add origin https://github.com/user/repo.git
git remote -v
git remote set-url origin https://github.com/user/newrepo.git
git fetch origin
git branch -r  # Remote branches

# VS Code integration
code .  # Open in VS Code
# Use Source Control panel in VS Code

# Ansible development container
# Dockerfile for Ansible dev environment
# Use with VS Code Dev Containers extension
```

## Playbook Examples

### Example 1: Clone Git Repository

```yaml
---
- name: Clone Git repository for Ansible playbooks
  hosts: localhost
  connection: local
  become: yes
  tasks:
    - name: Create directory for repository
      file:
        path: /home/ansible/ansible-playbooks
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Clone Ansible playbook repository
      command: git clone {{ git_repo_url }} {{ git_repo_dir }}
      args:
        creates: "{{ git_repo_dir }}/.git"
      environment:
        GIT_LFS_SKIP_SMUDGE: "1"

    - name: Configure Git user for clone
      lineinfile:
        path: /home/ansible/.gitconfig
        line: "{{ item }}"
        create: yes
      loop:
        - [user.name = "Ansible User", regex="^user.name"]
        - [user.email = ansible@example.com, regex="^user.email"]

    - name: Verify repository clone
      stat:
        path: "{{ git_repo_dir }}/.git"
      register: repo_stat

    - name: Display clone status
      debug:
        msg: "Repository {{ 'cloned successfully' if repo_stat.stat.exists else 'clone failed' }}"
```

### Example 2: Create and Configure Git Repository

```yaml
---
- name: Create and configure Git repository
  hosts: localhost
  connection: local
  become: yes
  vars:
    project_name: ansible-automation
    project_dir: /home/ansible/projects/{{ project_name }}
  tasks:
    - name: Create project directory
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
      changed_when: false

    - name: Configure Git user
      lineinfile:
        path: /home/ansible/.gitconfig
        line: "{{ item }}"
        create: yes
      loop:
        - [user.name = "Ansible User", regex="^user.name"]
        - [user.email = ansible@example.com, regex="^user.email"]

    - name: Create .gitignore file
      copy:
        content: |
          # Ansible
          ansible-lint-cache/
          ansible-playbook-cache/
          .ansible/
          ansible_facts_cache/
          inventory.~*~
          ansible.log
          ansible-tmp/

          # Python
          __pycache__/
          *.py[cod]
          *.pyo
          .coverage
          htmlcov/

          # OS
          .DS_Store
          Thumbs.db

          # Logs
          *.log
          logs/

          # Environment
          .env
          env/
          venvenv/
          venv/
        dest: "{{ project_dir }}/.gitignore"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create ansible.cfg
      copy:
        content: |
          [defaults]
          inventory = inventory
          remote_user = ansible
          host_key_checking = false
          log_path = logs/ansible.log
          forks = 10

          [privilege_escalation]
          become = true
          become_method = sudo
          become_user = root
          become_ask_pass = false

          [ssh_connection]
          pipelining = true
          ssh_args = -o ControlMaster=auto -o ControlPersist=60s
        dest: "{{ project_dir }}/ansible.cfg"
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
        dest: "{{ project_dir }}/inventory"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create playbooks directory
      file:
        path: "{{ project_dir }}/playbooks"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create initial playbook
      copy:
        content: |
          ---
          - name: Initial playbook
            hosts: all
            become: yes
            gather_facts: yes
            tasks:
              - name: Display hostname
                debug:
                  msg: "Running on {{ inventory_hostname }}"

              - name: Ping all hosts
                ping:
        dest: "{{ project_dir }}/playbooks/initial.yml"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Add all files to Git
      command: git add .
      args:
        chdir: "{{ project_dir }}"

    - name: Commit initial files
      command: git commit -m "Initial commit: Add project structure"
      args:
        chdir: "{{ project_dir }}"

    - name: Verify Git repository
      stat:
        path: "{{ project_dir }}/.git"
      register: git_stat

    - name: Display Git status
      command: git status
      args:
        chdir: "{{ project_dir }}"
      register: git_status
      changed_when: false

    - name: Display commit history
      command: git log --oneline
      args:
        chdir: "{{ project_dir }}"
      register: git_log
      changed_when: false
```

### Example 3: Add and Commit Changes

```yaml
---
- name: Add and commit changes to Git repository
  hosts: localhost
  connection: local
  become: yes
  vars:
    project_dir: /home/ansible/projects/ansible-automation
  tasks:
    - name: Create playbook to add
      copy:
        content: |
          ---
          - name: Configure web servers
            hosts: webservers
            become: yes
            tasks:
              - name: Install httpd
                dnf:
                  name: httpd
                  state: present

              - name: Start httpd
                systemd:
                  name: httpd
                  state: started
                  enabled: yes
        dest: "{{ project_dir }}/playbooks/webserver.yml"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create variable file
      copy:
        content: |
          app_name: myapp
          app_port: 8080
          log_level: info
        dest: "{{ project_dir }}/vars/webserver.yml"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Add files to Git staging area
      command: git add {{ item }}
      args:
        chdir: "{{ project_dir }}"
      loop:
        - "playbooks/webserver.yml"
        - "vars/webserver.yml"

    - name: Show staged changes
      command: git status --short
      args:
        chdir: "{{ project_dir }}"
      register: staged_status
      changed_when: false

    - name: Commit changes
      command: git commit -m "Add webserver playbook and variables"
      args:
        chdir: "{{ project_dir }}"

    - name: Verify commit
      command: git log --oneline -1
      args:
        chdir: "{{ project_dir }}"
      register: commit_info
      changed_when: false

    - name: Display commit message
      debug:
        var: commit_info.stdout
```

### Example 4: Push to Remote Repository

```yaml
---
- name: Push changes to remote Git repository
  hosts: localhost
  connection: local
  become: yes
  vars:
    project_dir: /home/ansible/projects/ansible-automation
    remote_repo: https://github.com/username/ansible-automation.git
  tasks:
    - name: Add remote origin
      lineinfile:
        path: "{{ project_dir }}/.git/config"
        line: "{{ item }}"
        create: yes
      loop:
        - [url = {{ remote_repo }}, regex="^\[remote \"origin\"\"\]"]
        - [fetch = {{ remote_repo }}, regex="^\s*fetch"]
        - [push = {{ remote_repo }}, regex="^\s*push"]

    - name: Verify remote configuration
      command: git remote -v
      args:
        chdir: "{{ project_dir }}"
      register: remote_config
      changed_when: false

    - name: Pull latest changes
      command: git pull origin main
      args:
        chdir: "{{ project_dir }}"
      register: pull_result
      failed_when: false

    - name: Push to remote repository
      command: git push origin main
      args:
        chdir: "{{ project_dir }}"
      environment:
        GIT_ASKPASS: /usr/bin/true  # For testing (remove in production)

    - name: Verify push
      command: git log --oneline --all
      args:
        chdir: "{{ project_dir }}"
      register: push_log
      changed_when: false

    - name: Display push result
      debug:
        msg: "Push {{ 'successful' if pull_result.rc == 0 else 'failed' }}"
```

### Example 5: Git Branch Management

```yaml
---
- name: Manage Git branches
  hosts: localhost
  connection: local
  become: yes
  vars:
    project_dir: /home/ansible/projects/ansible-automation
  tasks:
    - name: List all branches
      command: git branch -a
      args:
        chdir: "{{ project_dir }}"
      register: branches
      changed_when: false

    - name: Create feature branch
      command: git checkout -b feature-webserver
      args:
        chdir: "{{ project_dir }}"
      register: branch_created
      failed_when: false

    - name: Switch to feature branch
      command: git checkout feature-webserver
      args:
        chdir: "{{ project_dir }}"
      register: branch_switched
      failed_when: false

    - name: Create feature file
      copy:
        content: |
          # Feature file for webserver
          This is a feature branch file
        dest: "{{ project_dir }}/playbooks/feature-webserver.yml"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Add feature file to Git
      command: git add playbooks/feature-webserver.yml
      args:
        chdir: "{{ project_dir }}"

    - name: Commit feature changes
      command: git commit -m "Add feature webserver playbook"
      args:
        chdir: "{{ project_dir }}"

    - name: List branches again
      command: git branch -a
      args:
        chdir: "{{ project_dir }}"
      register: branches_after
      changed_when: false

    - name: Switch back to main branch
      command: git checkout main
      args:
        chdir: "{{ project_dir }}"
      register: back_to_main
      failed_when: false

    - name: Merge feature branch
      command: git merge feature-webserver -m "Merge feature-webserver into main"
      args:
        chdir: "{{ project_dir }}"
      register: merge_result
      failed_when: false

    - name: List final branches
      command: git branch -a
      args:
        chdir: "{{ project_dir }}"
      register: final_branches
      changed_when: false

    - name: Display branch operations result
      debug:
        msg: |
          Branches created: {{ branch_created.rc | default(0) }}
          Branch switched: {{ branch_switched.rc | default(0) }}
          Merge result: {{ merge_result.rc | default(0) }}
```

### Example 6: Complete Git Workflow

```yaml
---
- name: Complete Git workflow for Ansible playbooks
  hosts: localhost
  connection: local
  become: yes
  vars:
    project_dir: /home/ansible/projects/ansible-automation
    remote_repo: https://github.com/username/ansible-automation.git
  tasks:
    # Initialize repository
    - name: Initialize Git repository
      command: git init
      args:
        chdir: "{{ project_dir }}"
      when: not (stat(path="{{ project_dir }}/.git").stat.exists)

    # Configure Git user
    - name: Configure Git user
      lineinfile:
        path: /home/ansible/.gitconfig
        line: "{{ item }}"
        create: yes
      loop:
        - [user.name = "Ansible User", regex="^user.name"]
        - [user.email = ansible@example.com, regex="^user.email"]

    # Create project structure
    - name: Create project structure
      file:
        path: "{{ item }}"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"
      loop:
        - "{{ project_dir }}/playbooks"
        - "{{ project_dir }}/roles"
        - "{{ project_dir }}/templates"
        - "{{ project_dir }}/vars"
        - "{{ project_dir }}/inventory"

    # Create initial files
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

          [dbservers]
          serverb.lab.example.com

          [all:vars]
          ansible_user=ansible
        dest: "{{ project_dir }}/inventory"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Create initial playbook
      copy:
        content: |
          ---
          - name: Initial playbook
            hosts: all
            become: yes
            tasks:
              - name: Ping all hosts
                ping:
        dest: "{{ project_dir }}/playbooks/initial.yml"
        owner: ansible
        group: ansible
        mode: "0644"

    # First commit
    - name: Add initial files
      command: git add .
      args:
        chdir: "{{ project_dir }}"

    - name: Initial commit
      command: git commit -m "Initial commit: Project structure"
      args:
        chdir: "{{ project_dir }}"

    # Create feature branch
    - name: Create feature branch
      command: git checkout -b feature-webserver
      args:
        chdir: "{{ project_dir }}"

    - name: Add webserver playbook
      copy:
        content: |
          ---
          - name: Configure web servers
            hosts: webservers
            become: yes
            tasks:
              - name: Install httpd
                dnf:
                  name: httpd
                  state: present
              - name: Start httpd
                systemd:
                  name: httpd
                  state: started
                  enabled: yes
        dest: "{{ project_dir }}/playbooks/webserver.yml"
        owner: ansible
        group: ansible
        mode: "0644"

    - name: Add webserver playbook
      command: git add playbooks/webserver.yml
      args:
        chdir: "{{ project_dir }}"

    - name: Commit webserver changes
      command: git commit -m "Add webserver playbook"
      args:
        chdir: "{{ project_dir }}"

    # Switch back to main
    - name: Switch to main branch
      command: git checkout main
      args:
        chdir: "{{ project_dir }}"

    # Merge feature
    - name: Merge feature branch
      command: git merge feature-webserver -m "Merge feature-webserver"
      args:
        chdir: "{{ project_dir }}"

    # Push to remote
    - name: Add remote origin
      lineinfile:
        path: "{{ project_dir }}/.git/config"
        line: "{{ item }}"
        create: yes
      loop:
        - [url = {{ remote_repo }}, regex="^\[remote \"origin\"\"\]"]
        - [fetch = {{ remote_repo }}, regex="^\s*fetch"]
        - [push = {{ remote_repo }}, regex="^\s*push"]

    - name: Push to remote
      command: git push origin main
      args:
        chdir: "{{ project_dir }}"

    # Display summary
    - name: Display Git history
      command: git log --oneline --all --graph
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

    - name: Display workflow summary
      debug:
        msg: |
          Repository initialized: YES
          Initial commit: COMPLETED
          Feature branch: CREATED
          Merge: COMPLETED
          Push to remote: COMPLETED
          Total commits: {{ git_log.stdout_lines | length }}
```

## Explanation of Each Playbook

### Playbook 1: Clone Repository

This playbook demonstrates cloning a Git repository using the `command` module. It creates the target directory, runs `git clone`, and verifies the repository was created. The `creates` parameter prevents re-cloning if the repository already exists.

### Playbook 2: Create and Configure Repository

This comprehensive playbook creates a complete Git repository from scratch. It initializes Git, configures the user, creates `.gitignore` for Ansible, sets up `ansible.cfg`, creates inventory, and writes an initial playbook. Each step is committed to Git, demonstrating the complete workflow.

### Playbook 3: Add and Commit Changes

This playbook demonstrates adding new files to an existing repository and committing them. It creates a new playbook and variable file, stages them with `git add`, and commits with a descriptive message.

### Playbook 4: Push to Remote

This playbook configures a remote repository and pushes changes. It adds the remote origin, pulls the latest changes, and pushes the current branch. This demonstrates the complete push workflow.

### Playbook 5: Branch Management

This playbook demonstrates Git branching operations: creating a feature branch, switching branches, making changes, and merging back to main. It shows how to isolate changes in feature branches.

### Playbook 6: Complete Workflow

This comprehensive playbook demonstrates the entire Git workflow: initialization, configuration, project structure creation, initial commit, feature branch creation, changes, merge, and push. It provides a complete example of Git version control for Ansible playbooks.

## Variables and Templates

### Using Variables in Git Operations

```yaml
---
- name: Git operations with variables
  hosts: localhost
  connection: local
  become: yes
  vars:
    project_name: ansible-automation
    project_dir: /home/ansible/projects/{{ project_name }}
    remote_repo: https://github.com/username/{{ project_name }}.git
    branch_name: feature-webserver
  tasks:
    - name: Clone repository
      command: git clone {{ remote_repo }} {{ project_dir }}

    - name: Add files
      command: git add .
      args:
        chdir: "{{ project_dir }}"

    - name: Commit
      command: git commit -m "Add {{ project_name }} files"
      args:
        chdir: "{{ project_dir }}"

    - name: Create branch
      command: git checkout -b {{ branch_name }}
      args:
        chdir: "{{ project_dir }}"
```

### Using Templates for Git Configuration

```yaml
# Template: .gitignore.j2
# Git ignore for {{ project_name }}

# Ansible
{{ ansible_ignore_patterns | default('*.ansible', sep='\n') }}
.ansible/
ansible_facts_cache/

# Python
__pycache__/
*.pyc

# OS
.DS_Store
Thumbs.db

# Logs
*.log
```

```yaml
# Playbook using template
---
- name: Create .gitignore
  template:
    src: templates/.gitignore.j2
    dest: "{{ project_dir }}/.gitignore"
    owner: ansible
    group: ansible
    mode: "0644"
```

## Verification Procedures

### Verify Git Installation

```bash
# Check Git version
git --version

# Verify Git is in PATH
which git
```

### Verify Repository Status

```bash
# Check repository status
git status

# Check Git directory exists
ls -la .git/

# Check configured remotes
git remote -v

# Check branches
git branch -a

# Check commit history
git log --oneline
```

### Verify Git Operations

```bash
# Verify clone
ls -la .git/

# Verify add
git status --short

# Verify commit
git log --oneline -1

# Verify push
git log --oneline --all

# Verify remote
git remote -v
```

## Troubleshooting

### Git Clone Failures

**Problem: Clone fails with authentication error**

```bash
# Solution: Use SSH keys instead of HTTPS
git clone git@github.com:user/repo.git

# Solution: Configure Git credentials
git config --global credential.helper store

# Solution: Use SSH key authentication
ssh-keygen -t ed25519
ssh-copy-id git@github.com
```

**Ansible playbook:**
```yaml
- name: Clone with SSH
  command: git clone git@github.com:user/repo.git {{ project_dir }}
  args:
    creates: "{{ project_dir }}/.git"
```

### Git Push Failures

**Problem: Push fails with permission denied**

```bash
# Solution: Check SSH key permissions
chmod 600 ~/.ssh/id_ed25519
chmod 700 ~/.ssh

# Solution: Verify remote URL
git remote -v

# Solution: Use correct branch
git push origin main
```

**Ansible playbook:**
```yaml
- name: Verify SSH key permissions
  file:
    path: "{{ item }}"
    mode: "{{ item.mode }}"
  loop:
    - { path: ~/.ssh/id_ed25519, mode: "0600" }
    - { path: ~/.ssh, mode: "0700" }

- name: Push to remote
  command: git push origin main
  args:
    chdir: "{{ project_dir }}"
```

### Git Merge Conflicts

**Problem: Merge conflict during branch merge**

```bash
# Solution: View conflict
cat file.yml

# Solution: Resolve conflict manually
# Mark conflict sections:
# <<<<<<< HEAD
# Your changes
# =======
# Their changes
# >>>>>>> feature

# Solution: Choose one and remove markers
# git add file.yml
# git commit -m "Resolved conflict"
```

**Ansible playbook:**
```yaml
- name: Check for merge conflicts
  command: git status
  args:
    chdir: "{{ project_dir }}"
  register: conflict_status
  changed_when: false

- name: Display conflict status
  debug:
    msg: "Conflict detected: {{ conflict_status.stdout_lines }}"
```

### Git Repository Issues

**Problem: Repository not found**

```bash
# Solution: Verify remote URL
git remote -v

# Solution: Add correct remote
git remote add origin https://github.com/user/repo.git

# Solution: Fetch remote
git fetch origin
```

## Real-World Automation Examples

### Complete Ansible Project Setup with Git

```yaml
---
- name: Complete Ansible project setup with Git
  hosts: localhost
  connection: local
  become: yes
  vars:
    project_name: ansible-automation
    project_dir: /home/ansible/projects/{{ project_name }}
    remote_repo: https://github.com/username/{{ project_name }}.git
  tasks:
    # Create project directory
    - name: Create project directory
      file:
        path: "{{ project_dir }}"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    # Initialize Git
    - name: Initialize Git repository
      command: git init
      args:
        chdir: "{{ project_dir }}"
      changed_when: false

    # Configure Git
    - name: Configure Git user
      lineinfile:
        path: /home/ansible/.gitconfig
        line: "{{ item }}"
        create: yes
      loop:
        - [user.name = "Ansible User", regex="^user.name"]
        - [user.email = ansible@example.com, regex="^user.email"]

    # Create .gitignore
    - name: Create .gitignore
      copy:
        content: |
          # Ansible
          .ansible/
          ansible_facts_cache/
          inventory.~*~
          ansible.log
          ansible-tmp/

          # Python
          __pycache__/
          *.py[cod]
          *.pyo
          .coverage

          # OS
          .DS_Store
          Thumbs.db

          # Logs
          *.log
        dest: "{{ project_dir }}/.gitignore"
        owner: ansible
        group: ansible
        mode: "0644"

    # Create project structure
    - name: Create playbooks directory
      file:
        path: "{{ project_dir }}/playbooks"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create roles directory
      file:
        path: "{{ project_dir }}/roles"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create templates directory
      file:
        path: "{{ project_dir }}/templates"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create vars directory
      file:
        path: "{{ project_dir }}/vars"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    - name: Create inventory directory
      file:
        path: "{{ project_dir }}/inventory"
        state: directory
        owner: ansible
        group: ansible
        mode: "0755"

    # Create ansible.cfg
    - name: Create ansible.cfg
      copy:
        content: |
          [defaults]
          inventory = inventory
          remote_user = ansible
          host_key_checking = false
          forks = 10
          collections_paths = {{ project_dir }}/.ansible/collections:/usr/share/ansible/collections

          [privilege_escalation]
          become = true
          become_method = sudo
          become_user = root
          become_ask_pass = false

          [ssh_connection]
          pipelining = true
          ssh_args = -o ControlMaster=auto -o ControlPersist=60s
        dest: "{{ project_dir }}/ansible.cfg"
        owner: ansible
        group: ansible
        mode: "0644"

    # Create inventory
    - name: Create inventory
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
        dest: "{{ project_dir }}/inventory"
        owner: ansible
        group: ansible
        mode: "0644"

    # Create initial playbook
    - name: Create initial playbook
      copy:
        content: |
          ---
          - name: Initial playbook
            hosts: all
            become: yes
            gather_facts: yes
            tasks:
              - name: Display hostname
                debug:
                  msg: "Running on {{ inventory_hostname }}"

              - name: Ping all hosts
                ping:
        dest: "{{ project_dir }}/playbooks/initial.yml"
        owner: ansible
        group: ansible
        mode: "0644"

    # Create README.md
    - name: Create README.md
      copy:
        content: |
          # {{ project_name }}

          Ansible automation project.

          ## Setup

          ```bash
          ansible-playbook playbooks/initial.yml
          ```

          ## Structure

          - playbooks/ - Ansible playbooks
          - roles/ - Ansible roles
          - templates/ - Jinja2 templates
          - vars/ - Variable files
          - inventory/ - Host inventory

          ## Git

          ```bash
          git add .
          git commit -m "message"
          git push origin main
          ```
        dest: "{{ project_dir }}/README.md"
        owner: ansible
        group: ansible
        mode: "0644"

    # Add files to Git
    - name: Add all files to Git
      command: git add .
      args:
        chdir: "{{ project_dir }}"

    # Commit initial files
    - name: Initial commit
      command: git commit -m "Initial commit: Project structure"
      args:
        chdir: "{{ project_dir }}"

    # Add remote
    - name: Add remote origin
      lineinfile:
        path: "{{ project_dir }}/.git/config"
        line: "{{ item }}"
        create: yes
      loop:
        - [url = {{ remote_repo }}, regex="^\[remote \"origin\"\"\]"]
        - [fetch = {{ remote_repo }}, regex="^\s*fetch"]
        - [push = {{ remote_repo }}, regex="^\s*push"]

    # Push to remote
    - name: Push to remote repository
      command: git push -u origin main
      args:
        chdir: "{{ project_dir }}"

    # Display summary
    - name: Display Git summary
      debug:
        msg: |
          Project: {{ project_name }}
          Directory: {{ project_dir }}
          Remote: {{ remote_repo }}
          Status: Repository initialized and pushed to remote
```

## RHCE Exam Notes

### Critical Exam Points

1. **Git is essential for exam**: You will need to create playbooks, commit them, and push to Git.

2. **Know basic Git commands**: `clone`, `init`, `add`, `commit`, `push`, `pull`, `branch`, `checkout`.

3. **Commit messages matter**: Write clear, descriptive commit messages.

4. **VS Code integration**: Configure ansible-navigator in VS Code for development.

5. **Development containers**: Know how to use Ansible development containers.

6. **Git LFS for large files**: Use Git LFS for large binary files.

7. **Remote configuration**: Know how to add and configure remote repositories.

8. **Branch workflow**: Understand feature branches and merging.

### Exam Workflow

1. Create project directory
2. Initialize Git: `git init`
3. Configure Git user
4. Create project files
5. Add files: `git add .`
6. Commit: `git commit -m "message"`
7. Add remote: `git remote add origin url`
8. Push: `git push -u origin main`

### Time Management

- 5 minutes: Git initialization and configuration
- 10 minutes: Create playbooks and files
- 5 minutes: Add and commit
- 5 minutes: Push to remote

## Common Mistakes

### Mistake 1: Not Configuring Git User

```bash
# WRONG - commits will fail without user config
git commit -m "message"

# CORRECT - configure user first
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

### Mistake 2: Forgetting to Add Files

```bash
# WRONG - nothing to commit
git commit -m "message"

# CORRECT - add files first
git add .
git commit -m "message"
```

### Mistake 3: Pushing Without Pulling First

```bash
# WRONG - may lose remote changes
git push origin main

# CORRECT - pull first
git pull origin main
git push origin main
```

### Mistake 4: Not Using Descriptive Commit Messages

```bash
# WRONG - unclear message
git commit -m "fix"

# CORRECT - descriptive message
git commit -m "Fix webserver playbook: Add httpd service configuration"
```

## Chapter Summary

This chapter covered Git operations for Ansible playbooks:

- **Git basics**: Version control system for tracking changes to files.
- **Clone**: Download repositories from remote sources.
- **Create repository**: Initialize new repositories with `git init`.
- **Add files**: Stage changes with `git add` for commits.
- **Commit**: Create snapshots with descriptive messages.
- **Push**: Upload changes to remote repositories.
- **Pull**: Fetch and merge changes from remote.
- **Branches**: Create and manage feature branches.
- **VS Code integration**: Configure Git and ansible-navigator in VS Code.
- **Development containers**: Use Docker containers for isolated execution.

## Quick Reference

| Command | Purpose |
|---|---|
| `git clone url` | Clone repository |
| `git init` | Initialize repository |
| `git add .` | Add all files |
| `git commit -m "msg"` | Commit with message |
| `git status` | View status |
| `git log --oneline` | View history |
| `git push origin main` | Push to remote |
| `git pull origin main` | Pull from remote |
| `git branch` | List branches |
| `git checkout -b name` | Create and switch branch |
| `git merge branch` | Merge branch |
| `git remote -v` | View remotes |
| `git fetch origin` | Fetch remote changes |
| `git stash` | Temporarily save changes |
| `git stash pop` | Restore stashed changes |

## Review Questions

1. What is the purpose of version control in Ansible automation?

2. How do you initialize a new Git repository?

3. What is the difference between `git add` and `git commit`?

4. How do you configure Git user information?

5. What is the purpose of `.gitignore` and why is it important for Ansible?

6. How do you push changes to a remote repository for the first time?

7. What is the difference between `git pull` and `git fetch`?

8. How do you create a new feature branch?

9. What is the purpose of `git stash`?

10. How do you open a Git repository in VS Code?

## Answers

1. Version control tracks changes to files over time, allowing you to revert to previous versions, collaborate with others, maintain an audit trail of changes, and rollback failed changes. For Ansible, it provides version history of playbooks, enables team collaboration, and ensures you can always recover from mistakes.

2. Use `git init` to initialize a new Git repository in the current directory. This creates a `.git` directory containing Git metadata and object database. Example: `mkdir project && cd project && git init`.

3. `git add` stages files for the next commit, adding them to the staging area. `git commit` creates a snapshot of the staged files and records it in the Git history with a commit message. You must `add` files before you can `commit` them.

4. Use `git config --global user.name "Your Name"` and `git config --global user.email "your.email@example.com"` to configure Git user information. These settings apply to all repositories on the system.

5. `.gitignore` specifies files and directories that Git should ignore and not track. For Ansible, it's important to exclude temporary files (`.ansible/`), logs (`*.log`), cache files (`ansible_facts_cache/`), and Python bytecode (`__pycache__/`) to keep the repository clean and avoid tracking unnecessary files.

6. To push changes to a remote repository for the first time: (1) Add the remote: `git remote add origin https://github.com/user/repo.git`, (2) Push and set upstream: `git push -u origin main`. The `-u` flag sets the upstream tracking so future pushes use the correct remote branch.

7. `git pull` fetches changes from the remote and automatically merges them into your current branch. `git fetch` only downloads changes from the remote without merging. Use `fetch` first to review changes, then merge manually if needed.

8. Use `git checkout -b feature-name` to create a new branch and switch to it in one command. This creates a copy of the current branch and names it `feature-name`. You can then make changes in isolation and merge back later.

9. `git stash` temporarily saves uncommitted changes without committing them. This allows you to switch branches or work on other tasks without losing your changes. Use `git stash pop` to restore the most recent stashed changes.

10. Use `code .` from the command line to open the current directory in VS Code. VS Code has built-in Git integration with a Source Control panel for viewing changes, staging files, committing, and pushing. You can also configure ansible-navigator to run within VS Code for integrated development.
