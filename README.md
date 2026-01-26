# Containerized Web Dev Bootstrap

A bootstrapping tool for setting up a containerized web development environment using Ansible.

## Prerequisites

- **WSL Version:** 2.6.3.0
- **OS:** Ubuntu 24.04 LTS (Noble Numbat)
- **Ansible:** 2.16.3 (core)

## Quick Start

```bash
# 1. Install Ansible
sudo apt update && sudo apt install -y ansible

# 2. Install required collections
ansible-galaxy collection install -r requirements.yml

# 3. Run setup
ansible-playbook playbooks/setup.yml

# 4. Activate docker group (or log out/in)
newgrp docker

# 5. Configure AWS CLI
aws configure

# 6. Pull an image
ansible-playbook playbooks/deploy.yml \
  -e ecr_alias=docker \
  -e ecr_repo=library/node \
  -e image_tag=21-alpine
```

## Installation

### 1. Install Ansible

```bash
sudo apt update
sudo apt install -y ansible
```

### 2. Install Ansible Collections

```bash
ansible-galaxy collection install -r requirements.yml
```

This installs:
- `community.docker` - Docker container management
- `community.general` - General utilities

### 3. Run the Setup Playbook

```bash
ansible-playbook playbooks/setup.yml
```

This will install:
- Node.js and npm (dev only)
- AWS CLI v2
- Docker CE (core engine, CLI, containerd)
- Python Docker SDK (for community.docker)
- Adds your user to the `docker` group

> **Note:** After first install, log out and back in (or run `newgrp docker`) for docker group membership to take effect.

For production (skips npm):

```bash
ansible-playbook playbooks/setup.yml -e env=prod
```

### 4. Configure AWS CLI (Local/WSL Only)

After running setup, configure AWS CLI:

```bash
aws configure
```

You'll be prompted for:
- AWS Access Key ID
- AWS Secret Access Key
- Default region (e.g., `us-east-1`)
- Default output format (e.g., `json`)

> **Note:** On EC2 instances, use IAM roles instead - no manual configuration needed.

## Deployment

### ECR Login Only

```bash
ansible-playbook playbooks/deploy.yml
```

ECR tokens expire after 12 hours. Re-run to refresh.

#### WSL Credential Store Fix

If Docker Desktop is installed on Windows, WSL may fail to store ECR credentials. Fix:

```bash
cat > ~/.docker/config.json << 'EOF'
{
  "auths": {},
  "credsStore": ""
}
EOF
```

### Pull an Image

```bash
ansible-playbook playbooks/deploy.yml \
  -e ecr_alias=docker \
  -e ecr_repo=library/node \
  -e image_tag=21-alpine
```

Output:
```
✓ Logged into ECR Public

✓ Pulled image
  Repository: public.ecr.aws/docker/library/node
  Tag:        21-alpine
  Size:       46.69 MB
```

### Run a Container

Add `-e run_container=true` to also start the container:

```bash
ansible-playbook playbooks/deploy.yml \
  -e ecr_alias=docker \
  -e ecr_repo=library/node \
  -e image_tag=21-alpine \
  -e run_container=true
```

Output:
```
✓ Container running
  Name:   node
  Image:  public.ecr.aws/docker/library/node:21-alpine
  Port:   3000:3000
  Status: running
```

### Deploy Options

| Variable | Description | Default |
|----------|-------------|---------|
| `ecr_alias` | ECR public alias | - |
| `ecr_repo` | Repository name | - |
| `image_tag` | Image tag | `latest` |
| `run_container` | Start container after pull | `false` |
| `container_name` | Override container name | repo basename |
| `container_port` | Port mapping | `3000:3000` |
| `container_command` | Override command | - |
| `env_file` | Path to env file | - |
| `restart_policy` | Restart policy | `no` |
| `force_recreate` | Stop and recreate existing container | `false` |

### Using Environment Files

For apps that need environment variables, create an env file:

```bash
# vars/my-app.env
NODE_ENV=production
API_KEY=secret123
DATABASE_URL=postgres://localhost/mydb
```

Then deploy with:

```bash
ansible-playbook playbooks/deploy.yml \
  -e ecr_alias=your-alias \
  -e ecr_repo=myapp/backend \
  -e run_container=true \
  -e env_file=vars/my-app.env
```

### Push Images

```bash
# Tag for ECR
docker tag myapp:v1.0.0 public.ecr.aws/your-alias/myapp:v1.0.0

# Push
docker push public.ecr.aws/your-alias/myapp:v1.0.0
```

> **Note:** Create ECR Public repositories via AWS Console first.

## Example Walkthrough

Complete example: Pull and run a Node.js server.

```bash
# 1. Setup (first time only)
ansible-playbook playbooks/setup.yml
newgrp docker
aws configure

# 2. Pull and run Node.js
ansible-playbook playbooks/deploy.yml \
  -e ecr_alias=docker \
  -e ecr_repo=library/node \
  -e image_tag=21-alpine \
  -e run_container=true \
  -e container_command="node -e \"require('http').createServer((req, res) => res.end('Hello!')).listen(3000)\""

# 3. Test it
curl http://localhost:3000

# 4. Cleanup
docker stop node && docker rm node
```

## Versions at Time of Writing

| Tool | Version |
|------|---------|
| npm | 9.2.0 |

## Project Structure

```
.
├── ansible.cfg              # Ansible configuration
├── requirements.yml         # Ansible collection dependencies
├── playbooks/
│   ├── setup.yml            # Install base tools + Docker
│   └── deploy.yml           # ECR login + pull/run container
├── roles/
│   ├── base/
│   │   └── tasks/
│   │       └── main.yml     # Node.js, npm (dev only), AWS CLI
│   └── docker/
│       └── tasks/
│           └── main.yml     # Docker installation
└── README.md
```

## Ansible Collections Used

- **community.docker** - Docker container, image, and network management
- **community.general** - General utilities and helpers
