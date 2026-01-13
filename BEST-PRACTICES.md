# Ansible Best Practices Guide

A comprehensive guide to production-grade Ansible automation based on real-world implementation.

---

## Table of Contents

1. [Project Structure](#project-structure)
2. [Inventory Management](#inventory-management)
3. [Secrets & Security](#secrets--security)
4. [Roles & Reusability](#roles--reusability)
5. [Variables & Configuration](#variables--configuration)
6. [Playbook Best Practices](#playbook-best-practices)
7. [Error Handling](#error-handling)
8. [CI/CD Integration](#cicd-integration)
9. [Service Discovery](#service-discovery)
10. [Testing & Validation](#testing--validation)
11. [Performance & Optimization](#performance--optimization)
12. [Documentation](#documentation)

---

## Project Structure

### ✅ Standard Directory Layout

```
ansible-project/
├── ansible.cfg                 # Main configuration
├── requirements.yml            # Galaxy role dependencies
├── site.yml                   # Main playbook
├── inventory/
│   ├── production/
│   │   ├── hosts.ini
│   │   └── group_vars/
│   │       ├── all.yml
│   │       └── all/
│   │           └── vault.yml
│   └── staging/
├── roles/
│   └── custom_role/
│       ├── tasks/
│       ├── handlers/
│       ├── templates/
│       ├── files/
│       ├── defaults/
│       ├── vars/
│       └── meta/
├── playbooks/                 # Specific playbooks
├── templates/                 # Global templates
├── .github/
│   └── workflows/
│       └── deploy.yml
├── .gitignore
└── README.md
```

### 🔑 Key Principles

- **One playbook per purpose** - Don't create monolithic playbooks
- **Environment separation** - Use separate inventory directories
- **Version control everything** - Except secrets (use vault)
- **Use relative paths** - Makes playbooks portable

---

## Inventory Management

### ✅ Use Dynamic Inventory for Cloud

**Why:**
- No hardcoded IPs
- Auto-discovers new instances
- Works with auto-scaling
- Reduces manual maintenance

**Implementation:**

```yaml
# aws_ec2.yml
plugin: aws_ec2
regions:
  - eu-west-1

keyed_groups:
  - key: tags.Role
    prefix: role
  - key: tags.Environment
    prefix: env

filters:
  instance-state-name: running

hostnames:
  - ip-address
  - private-ip-address

compose:
  ansible_host: public_ip_address | default(private_ip_address)
```

### ✅ AWS Tagging Strategy

**Required tags for all instances:**
- `Role` - Purpose (web, app, db)
- `Environment` - Lifecycle (dev, staging, prod)
- `Name` - Human-readable identifier
- `Project` - Project/application name

**Example:**
```
Role: web
Environment: prod
Name: web-server-01
Project: ecommerce
```

### ✅ Flexible Host Patterns

Use multiple group patterns for compatibility:

```yaml
# Works with static OR dynamic inventory
hosts: app:role_app

# Pattern matching
hosts: web*              # Matches web1, web2, webserver
hosts: web:&prod         # Intersection: web AND prod
hosts: web:!staging      # Exclusion: web but NOT staging
```

### ❌ Anti-patterns

- ❌ Hardcoding IPs in playbooks
- ❌ No separation between environments
- ❌ Mixing production and development inventory
- ❌ Using default inventory location

---

## Secrets & Security

### ✅ Ansible Vault Best Practices

**1. Separate vault files per environment**

```
inventory/
├── dev/
│   └── group_vars/
│       └── all/
│           └── vault.yml      # Dev secrets
└── prod/
    └── group_vars/
        └── all/
            └── vault.yml      # Prod secrets (different password!)
```

**2. Vault variable naming convention**

```yaml
# vault.yml - Encrypted
vault_db_password: "SuperSecret123"
vault_api_key: "secret-key"

# all.yml - Not encrypted (references vault)
db_password: "{{ vault_db_password }}"
api_key: "{{ vault_api_key }}"
```

**Why:** Separates sensitive from non-sensitive, makes refactoring easier

**3. Never commit unencrypted secrets**

```bash
# .gitignore
*.pem
*.key
.vault_pass
vault.yml
!vault.yml.example
```

**4. Vault password management**

```bash
# Development: Use prompt
ansible-playbook site.yml --ask-vault-pass

# CI/CD: Use password file
echo "$VAULT_PASSWORD" > .vault_pass
ansible-playbook site.yml --vault-password-file .vault_pass
rm .vault_pass  # Always cleanup
```

**5. Rotate vault passwords regularly**

```bash
# Change vault password
ansible-vault rekey inventory/prod/group_vars/all/vault.yml
```

### ✅ SSH Key Management

```yaml
# Use dedicated deployment keys
ansible_user: ubuntu
ansible_ssh_private_key_file: ~/.ssh/ansible-deploy.pem

# In CI/CD
- name: Setup SSH key
  run: |
    mkdir -p ~/.ssh
    echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/deploy.pem
    chmod 600 ~/.ssh/deploy.pem
```

### ❌ Security Anti-patterns

- ❌ Committing passwords to git
- ❌ Sharing vault passwords in Slack/Email
- ❌ Using root user directly
- ❌ Storing secrets in plaintext
- ❌ Same vault password for all environments

---

## Roles & Reusability

### ✅ Role Structure

**Use ansible-galaxy to generate:**

```bash
ansible-galaxy init roles/nginx
```

**Results in:**
```
roles/nginx/
├── tasks/
│   └── main.yml          # Main task list
├── handlers/
│   └── main.yml          # Restart/reload handlers
├── templates/
│   └── nginx.conf.j2     # Jinja2 templates
├── files/
│   └── index.html        # Static files
├── defaults/
│   └── main.yml          # Default variables (lowest precedence)
├── vars/
│   └── main.yml          # Internal variables (high precedence)
└── meta/
    └── main.yml          # Dependencies
```

### ✅ Role Best Practices

**1. One role = One responsibility**

```yaml
# ✅ Good - Single purpose
roles:
  - nginx
  - nodejs
  - postgresql

# ❌ Bad - Monolithic
roles:
  - web_stack  # Does everything
```

**2. Use defaults for user-configurable values**

```yaml
# roles/nginx/defaults/main.yml
nginx_port: 80
nginx_worker_processes: auto
nginx_worker_connections: 1024
```

**3. Use vars for internal logic**

```yaml
# roles/nginx/vars/main.yml
nginx_package: nginx
nginx_service: nginx
nginx_config_dir: /etc/nginx
```

**4. Make roles idempotent**

```yaml
# ✅ Idempotent - Can run multiple times safely
- name: Install nginx
  apt:
    name: nginx
    state: present  # ← Same result every time

# ❌ Not idempotent
- name: Append to config
  shell: echo "setting=value" >> /etc/app.conf
```

**5. Use tags for selective execution**

```yaml
- name: Install nginx
  apt:
    name: nginx
    state: present
  tags:
    - nginx
    - packages
    - install
```

Run specific tags:
```bash
ansible-playbook site.yml --tags "nginx"
ansible-playbook site.yml --skip-tags "install"
```

### ✅ Using Galaxy Roles

**requirements.yml:**
```yaml
roles:
  - name: geerlingguy.docker
    version: 7.4.1
  - name: geerlingguy.mysql
    version: 4.3.5
```

**Install:**
```bash
ansible-galaxy role install -r requirements.yml
```

**Use in playbook:**
```yaml
- hosts: app
  roles:
    - geerlingguy.docker  # Galaxy role
    - myapp              # Custom role
```

### ❌ Role Anti-patterns

- ❌ Roles with hardcoded values
- ❌ Roles that depend on specific inventory
- ❌ Roles that modify global state
- ❌ Kitchen-sink roles (do everything)

---

## Variables & Configuration

### ✅ Variable Precedence (Lowest to Highest)

1. `role defaults` (roles/*/defaults/main.yml)
2. `inventory file` or `script group vars`
3. `inventory group_vars/all`
4. `inventory group_vars/*`
5. `inventory host_vars/*`
6. `playbook group_vars/all`
7. `playbook group_vars/*`
8. `playbook host_vars/*`
9. `host facts / cached set_facts`
10. `play vars`
11. `play vars_files`
12. `role vars` (roles/*/vars/main.yml)
13. `block vars` (in tasks)
14. `task vars`
15. `include_vars`
16. `set_facts` / `registered vars`
17. `role params`
18. `include params`
19. `extra vars` (`-e` in CLI)

### ✅ Variable Naming Conventions

```yaml
# ✅ Good - Prefixed, clear
nginx_port: 80
app_db_host: localhost
vault_api_key: secret

# ❌ Bad - Generic, collision risk
port: 80
host: localhost
key: secret
```

### ✅ Variable Organization

```yaml
# group_vars/all.yml - Global settings
ansible_user: ubuntu
ansible_ssh_private_key_file: ~/.ssh/deploy.pem

# group_vars/web.yml - Web tier specific
nginx_port: 8080
nginx_worker_processes: 4

# group_vars/all/vault.yml - Secrets
vault_db_password: encrypted_value
```

### ❌ Variable Anti-patterns

- ❌ Using generic variable names (port, host, user)
- ❌ Hardcoding values in tasks
- ❌ Mixing environment-specific and generic vars
- ❌ Not documenting complex variable structures

---

## Playbook Best Practices

### ✅ Playbook Structure

```yaml
---
- name: Configure Web Servers  # Descriptive name
  hosts: web:role_web         # Flexible patterns
  become: yes                 # Privilege escalation
  
  vars:
    app_version: "1.2.3"     # Play-level vars
  
  pre_tasks:                  # Run before roles
    - name: Update apt cache
      apt:
        update_cache: yes
  
  roles:
    - nginx
    - app
  
  tasks:
    - name: Deploy application
      copy:
        src: app.jar
        dest: /opt/app/
      notify: restart app
  
  post_tasks:                 # Run after everything
    - name: Verify deployment
      uri:
        url: http://localhost/health
        status_code: 200
  
  handlers:
    - name: restart app
      systemd:
        name: app
        state: restarted
```

### ✅ Task Best Practices

**1. Always name your tasks**

```yaml
# ✅ Good
- name: Install nginx web server
  apt:
    name: nginx
    state: present

# ❌ Bad
- apt:
    name: nginx
    state: present
```

**2. Use modules over commands**

```yaml
# ✅ Good - Idempotent
- name: Install package
  apt:
    name: nginx
    state: present

# ❌ Bad - Not idempotent
- name: Install package
  command: apt-get install -y nginx
```

**3. Use check mode for safety**

```yaml
# Supports --check mode
- name: Create directory
  file:
    path: /opt/app
    state: directory
    
# Document when check mode isn't supported
- name: Run migration
  command: /opt/app/migrate.sh
  check_mode: no  # Can't simulate
```

**4. Add changed_when and failed_when**

```yaml
- name: Check service status
  command: systemctl is-active nginx
  register: service_status
  changed_when: false         # Never reports changed
  failed_when: service_status.rc not in [0, 3]  # 0=active, 3=inactive
```

### ✅ Using Handlers

```yaml
# handlers/main.yml
- name: restart nginx
  systemd:
    name: nginx
    state: restarted

- name: reload nginx
  systemd:
    name: nginx
    state: reloaded

# tasks/main.yml
- name: Update nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify:
    - reload nginx  # Only runs if config changed
```

**Handler rules:**
- Run at end of play
- Run only once even if notified multiple times
- Don't run if play fails (unless using `meta: flush_handlers`)

### ❌ Playbook Anti-patterns

- ❌ Using shell/command when module exists
- ❌ Not using handlers for service restarts
- ❌ Unnamed tasks
- ❌ Ignoring errors without reason
- ❌ Not using blocks for error handling

---

## Error Handling

### ✅ Block/Rescue/Always

```yaml
- name: Deploy application
  block:
    - name: Stop service
      systemd:
        name: app
        state: stopped
    
    - name: Deploy new version
      copy:
        src: app-v2.jar
        dest: /opt/app/app.jar
    
    - name: Start service
      systemd:
        name: app
        state: started
  
  rescue:
    - name: Rollback to previous version
      copy:
        src: /opt/app/backup/app.jar
        dest: /opt/app/app.jar
    
    - name: Start service with backup
      systemd:
        name: app
        state: started
    
    - name: Notify team
      debug:
        msg: "Deployment failed. Rolled back to previous version."
  
  always:
    - name: Log deployment attempt
      lineinfile:
        path: /var/log/deployments.log
        line: "{{ ansible_date_time.iso8601 }} - Deployment attempted"
```

### ✅ Graceful Failures

```yaml
# Continue on specific errors
- name: Check optional service
  service:
    name: optional-service
    state: started
  ignore_errors: yes

# Conditional failure
- name: Verify deployment
  uri:
    url: http://localhost/health
  register: health_check
  failed_when: 
    - health_check.status != 200
    - health_check.status != 503  # Maintenance mode OK
```

### ✅ Max Failure Percentage

```yaml
- name: Rolling update
  hosts: web
  serial: 2
  max_fail_percentage: 20  # Abort if >20% fail
  
  tasks:
    - name: Update server
      # ...
```

---

## CI/CD Integration

### ✅ GitHub Actions Workflow

```yaml
name: Deploy with Ansible

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      # 1. Checkout
      - uses: actions/checkout@v4
      
      # 2. Setup
      - name: Install dependencies
        run: |
          sudo apt update
          sudo apt install -y ansible python3-boto3
      
      # 3. Configure AWS
      - name: Configure AWS credentials
        run: |
          mkdir -p ~/.aws
          cat > ~/.aws/credentials << EOF
          [default]
          aws_access_key_id = ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws_secret_access_key = ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          EOF
          chmod 600 ~/.aws/credentials
      
      # 4. Install roles
      - name: Install Galaxy roles
        run: ansible-galaxy role install -r requirements.yml
      
      # 5. Setup SSH
      - name: Setup SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/deploy.pem
          chmod 600 ~/.ssh/deploy.pem
      
      # 6. Vault password
      - name: Create vault password
        run: |
          echo "${{ secrets.ANSIBLE_VAULT_PASSWORD }}" > .vault_pass
          chmod 600 .vault_pass
      
      # 7. Syntax check
      - name: Syntax check
        run: ansible-playbook site.yml --syntax-check
      
      # 8. Dry run
      - name: Check mode (dry-run)
        run: |
          ansible-playbook -i aws_ec2.yml site.yml --check --vault-password-file .vault_pass
      
      # 9. Deploy
      - name: Deploy
        run: |
          ansible-playbook -i aws_ec2.yml site.yml --vault-password-file .vault_pass
      
      # 10. Cleanup
      - name: Cleanup secrets
        if: always()
        run: |
          rm -f ~/.ssh/deploy.pem .vault_pass
          rm -rf ~/.aws
```

### ✅ Required GitHub Secrets

```
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
SSH_PRIVATE_KEY
ANSIBLE_VAULT_PASSWORD
```

### ✅ CI/CD Best Practices

1. **Always run syntax check first**
2. **Use check mode (dry-run) before actual deployment**
3. **Cleanup secrets after run** (`if: always()`)
4. **Use minimal IAM permissions** (least privilege)
5. **Tag deployments** for rollback capability
6. **Separate workflows** for different environments

---

## Service Discovery

### ✅ Layered Discovery Architecture

**Layer 1: Infrastructure Discovery (Ansible + AWS)**
```yaml
# Discovers infrastructure at deployment time
ansible-playbook -i aws_ec2.yml site.yml
```

**Layer 2: Runtime Discovery (Consul/etcd)**
```python
# Apps query for current service locations
import consul
c = consul.Consul()
db = c.health.service('postgres', passing=True)
```

### ✅ When to Use Each

**Use AWS Dynamic Inventory:**
- ✅ Infrastructure provisioning
- ✅ Configuration management
- ✅ Initial deployments
- ✅ Bootstrap operations

**Add Consul/Service Mesh:**
- ✅ Microservices architecture
- ✅ Auto-scaling applications
- ✅ Frequent IP changes
- ✅ Real-time health checks
- ✅ Service-to-service communication

### ✅ Hybrid Approach

```yaml
# Ansible deploys Consul cluster
- name: Deploy Consul
  hosts: all
  tasks:
    - name: Install Consul
      # AWS inventory discovers hosts
    
    - name: Configure Consul
      template:
        src: consul.hcl.j2
        dest: /etc/consul.d/consul.hcl
      vars:
        # Use discovered server IPs
        retry_join: "{{ groups['role_consul'] | map('extract', hostvars, 'ansible_host') | list }}"
```

---

## Testing & Validation

### ✅ Pre-deployment Checks

```bash
# Syntax validation
ansible-playbook site.yml --syntax-check

# List tasks
ansible-playbook site.yml --list-tasks

# List hosts
ansible-playbook site.yml --list-hosts

# Check mode (dry-run)
ansible-playbook site.yml --check

# Diff mode (show changes)
ansible-playbook site.yml --diff --check
```

### ✅ Testing in Stages

```yaml
# Stage 1: Syntax
- name: Syntax check
  run: ansible-playbook site.yml --syntax-check

# Stage 2: Dry run
- name: Check mode
  run: ansible-playbook site.yml --check

# Stage 3: Single host
- name: Deploy to one host
  run: ansible-playbook site.yml --limit web1

# Stage 4: Small batch
- name: Deploy to staging
  run: ansible-playbook -i staging site.yml

# Stage 5: Production
- name: Deploy to production
  run: ansible-playbook -i production site.yml
```

### ✅ Post-deployment Validation

```yaml
- name: Verify deployment
  hosts: web
  tasks:
    - name: Check service is running
      systemd:
        name: nginx
        state: started
      check_mode: yes
      register: service_check
    
    - name: Check HTTP endpoint
      uri:
        url: http://localhost
        status_code: 200
      retries: 3
      delay: 5
    
    - name: Verify configuration
      command: nginx -t
      changed_when: false
```

---

## Performance & Optimization

### ✅ Ansible Configuration

```ini
# ansible.cfg
[defaults]
# Parallel execution
forks = 20

# Fact caching
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600

# SSH optimization
[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
pipelining = True
control_path = /tmp/ansible-ssh-%%h-%%p-%%r
```

### ✅ Performance Tips

**1. Use serial for controlled rollouts**
```yaml
- hosts: web
  serial: 2  # Update 2 at a time
```

**2. Disable fact gathering when not needed**
```yaml
- hosts: all
  gather_facts: no
```

**3. Use async for long-running tasks**
```yaml
- name: Run long task
  command: /opt/app/migrate.sh
  async: 300  # 5 minutes
  poll: 10    # Check every 10 seconds
```

**4. Use strategy plugins**
```yaml
- hosts: all
  strategy: free  # Don't wait for all hosts
```

---

## Documentation

### ✅ Document Everything

**README.md:**
- Project overview
- Prerequisites
- Quick start
- Usage examples
- Troubleshooting

**Playbook comments:**
```yaml
---
# Purpose: Deploy web application
# Requirements: Ubuntu 22.04, sudo access
# Usage: ansible-playbook -i inventory/prod site.yml

- name: Configure Web Tier
  hosts: web
  become: yes
  
  tasks:
    # Update nginx config to use new backend servers
    - name: Update nginx config
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: reload nginx
```

**Role documentation:**
```yaml
# roles/nginx/meta/main.yml
galaxy_info:
  description: Install and configure nginx
  min_ansible_version: 2.9
  
  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy

dependencies: []
```

---

## Quick Reference Checklist

### ✅ Before Committing

- [ ] Secrets encrypted with vault
- [ ] No hardcoded IPs or passwords
- [ ] All tasks have names
- [ ] Syntax check passes
- [ ] Added to .gitignore: *.pem, .vault_pass, *.key

### ✅ Before Deploying

- [ ] Tested in staging environment
- [ ] Check mode (dry-run) successful
- [ ] Backup strategy in place
- [ ] Rollback plan documented
- [ ] Team notified

### ✅ Production Deployment

- [ ] Deploy during maintenance window
- [ ] Use serial/rolling updates
- [ ] Monitor logs and metrics
- [ ] Verify health checks pass
- [ ] Document changes

---

## Common Pitfalls to Avoid

### ❌ Don't

1. **Use shell/command when modules exist**
   - Use `apt` not `shell: apt-get install`
   
2. **Store secrets in plaintext**
   - Always use Ansible Vault

3. **Hardcode environment-specific values**
   - Use inventory variables

4. **Deploy to production without testing**
   - Always test in staging first

5. **Use root user directly**
   - Use `become: yes` with sudo

6. **Ignore errors silently**
   - Use proper error handling

7. **Create monolithic playbooks**
   - Break into roles and includes

8. **Forget to tag resources in AWS**
   - Tags enable dynamic inventory

9. **Mix secrets and configuration**
   - Separate vault files

10. **Skip documentation**
    - Future you will thank you

---

## Additional Resources

- [Ansible Documentation](https://docs.ansible.com/)
- [Ansible Galaxy](https://galaxy.ansible.com/)
- [Best Practices Guide](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [AWS EC2 Plugin](https://docs.ansible.com/ansible/latest/collections/amazon/aws/aws_ec2_inventory.html)
- [Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html)

---

**Last Updated:** January 13, 2026  
**Version:** 1.0  
**Maintainer:** DevOps Team
