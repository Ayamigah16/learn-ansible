# Ansible Learning Project

Production-ready Ansible automation with roles, vault, CI/CD, and cloud integration.

## Project Structure

```
learn-ansible/
├── ansible.cfg                 # Main configuration
├── requirements.yml            # Galaxy role dependencies
├── site.yml                   # Main playbook using roles
├── nginx.yml                  # Legacy monolithic playbook
├── docker.yml                 # Docker container management
├── rolling-deploy.yml         # Zero-downtime deployments
├── aws_ec2.yml               # Dynamic AWS inventory
├── inventory/
│   └── dev/
│       ├── hosts.ini         # Static inventory
│       └── group_vars/
│           ├── all.yml       # Global variables
│           ├── all/
│           │   └── vault.yml.example  # Vault secrets template
│           └── web.yml       # Web tier variables
├── roles/
│   └── nginx/                # Custom nginx role
│       ├── tasks/
│       ├── handlers/
│       ├── templates/
│       ├── defaults/
│       └── vars/
├── templates/                # Global templates
└── .github/
    └── workflows/
        └── deploy.yml        # CI/CD pipeline
```

## Quick Start

### 1. Install Dependencies

```bash
# Install Ansible
sudo apt update && sudo apt install -y ansible

# Install Galaxy roles
ansible-galaxy role install -r requirements.yml

# Install AWS dynamic inventory dependencies (optional)
pip install boto3 botocore
```

### 2. Configure SSH Access

Update [inventory/dev/group_vars/all.yml](inventory/dev/group_vars/all.yml):
```yaml
ansible_user: ubuntu
ansible_ssh_private_key_file: ~/.ssh/your-key.pem
```

### 3. Run Playbooks

```bash
# Basic deployment
ansible-playbook -i inventory/dev/hosts.ini site.yml

# Check mode (dry-run)
ansible-playbook -i inventory/dev/hosts.ini site.yml --check

# Diff mode (show changes)
ansible-playbook -i inventory/dev/hosts.ini site.yml --diff

# Rolling deployment
ansible-playbook -i inventory/dev/hosts.ini rolling-deploy.yml

# Docker management
ansible-playbook -i inventory/dev/hosts.ini docker.yml
```

## Ansible Vault (Secrets Management)

### Create encrypted vault:
```bash
ansible-vault create inventory/dev/group_vars/all/vault.yml
```

### Edit vault:
```bash
ansible-vault edit inventory/dev/group_vars/all/vault.yml
```

### Run with vault:
```bash
ansible-playbook site.yml --ask-vault-pass

# Or use password file
echo "your-vault-password" > .vault_pass
chmod 600 .vault_pass
ansible-playbook site.yml --vault-password-file .vault_pass
```

## Dynamic Inventory (AWS)

### Prerequisites:
```bash
# Configure AWS credentials
aws configure

# Or use environment variables
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
```

### Test dynamic inventory:
```bash
# List all discovered instances
ansible-inventory -i aws_ec2.yml --graph

# List specific group
ansible-inventory -i aws_ec2.yml --graph @role_web

# Target by AWS tags
ansible role_web -i aws_ec2.yml -m ping
ansible env_dev -i aws_ec2.yml -m ping
```

## CI/CD with GitHub Actions

### Setup GitHub Secrets:
1. Go to repository Settings → Secrets and variables → Actions
2. Add these secrets:
   - `SSH_PRIVATE_KEY`: Your SSH private key
   - `ANSIBLE_VAULT_PASSWORD`: Vault password
   - `HOST_IP`: Target host IP (optional for ssh-keyscan)

### Workflow triggers:
- Push to `main` branch
- Pull request to `main` branch
- Manual dispatch from Actions tab

## Advanced Features

### Ad-hoc Commands

```bash
# Ping all hosts
ansible all -i inventory/dev/hosts.ini -m ping

# Gather facts
ansible web -i inventory/dev/hosts.ini -m setup

# Check disk space
ansible all -i inventory/dev/hosts.ini -m shell -a "df -h"

# Install package
ansible web -i inventory/dev/hosts.ini -m apt -a "name=htop state=present" --become
```

### Performance & Safety

```bash
# Syntax check
ansible-playbook site.yml --syntax-check

# List tasks
ansible-playbook site.yml --list-tasks

# List hosts
ansible-playbook site.yml --list-hosts

# Step-by-step execution
ansible-playbook site.yml --step

# Start at specific task
ansible-playbook site.yml --start-at-task="Install Nginx"
```

### Debugging

```bash
# Verbose output
ansible-playbook site.yml -v    # verbose
ansible-playbook site.yml -vv   # more verbose
ansible-playbook site.yml -vvv  # very verbose (connection debugging)

# Check variable values
ansible web -i inventory/dev/hosts.ini -m debug -a "var=nginx_port"
```

## Production Best Practices

✅ **Implemented:**
- Role-based structure for reusability
- Ansible Vault for secrets
- Dynamic inventory for cloud resources
- CI/CD pipeline integration
- Rolling deployments for zero-downtime
- Check and diff modes for safety
- Error handling with blocks/rescue/always

📝 **Recommended:**
- Use tags for selective execution
- Implement proper logging and monitoring
- Version control all playbooks
- Test in staging before production
- Document role dependencies
- Rotate vault passwords regularly

## Troubleshooting

### Connection Issues
```bash
# Test SSH connectivity
ansible all -i inventory/dev/hosts.ini -m ping -vvv

# Test with different user
ansible all -i inventory/dev/hosts.ini -m ping -u ubuntu
```

### Inventory Issues
```bash
# List all hosts
ansible-inventory -i inventory/dev/hosts.ini --list

# Check group membership
ansible-inventory -i inventory/dev/hosts.ini --graph
```

### Role Issues
```bash
# Install missing roles
ansible-galaxy role install -r requirements.yml --force

# List installed roles
ansible-galaxy role list
```

## Resources

- [Ansible Documentation](https://docs.ansible.com/)
- [Ansible Galaxy](https://galaxy.ansible.com/)
- [Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)

## License

MIT
