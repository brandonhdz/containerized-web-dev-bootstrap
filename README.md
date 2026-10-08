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
- Node.js 24 LTS and npm, through [nvm](https://github.com/nvm-sh/nvm), for you (dev only). For an older project, `nvm install 18` and `nvm use 18`, or an `.nvmrc` in it. Ubuntu's own Node.js, if an earlier run installed it, is removed first.
- Terraform through [tfenv](https://github.com/tfutils/tfenv), for you (dev only): the newest as your default, and in a project with a `.terraform-version` file, the version it names (downloaded the first time you run `terraform` there). Every download is checked against HashiCorp's signing key. `terraform` and `tfenv` are linked into `~/.local/bin`, on PATH once you log in again. A `terraform` already there, installed by hand, stops the run with how to move it aside.
- AWS CLI v2
- Docker CE (core engine, CLI, containerd)
- Python Docker SDK (for community.docker)
- Adds your user to the `docker` group
- [dev-scripts](https://github.com/brandonhdz/dev-scripts) (private): bash commands for development (`in-aws`, `log-run`, `tf`, `ans-pbk`), linked into `~/bin`, with tab completion. As you, not root.

> **Note:** dev-scripts is cloned over SSH, so this machine's SSH key has to be on GitHub first: `ssh-keygen -t ed25519`, then `gh ssh-key add ~/.ssh/id_ed25519.pub` (or github.com/settings/keys), and check with `ssh -T git@github.com`. A clone that's already there is left alone; update it with `git pull`.

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
| nvm | 0.40.8 |
| Node.js | 24.21.0 |
| npm | 11.19.0 |
| tfenv | 3.2.2 |
| Terraform | 1.16.5 |
| AWS CLI | 2.37.10 |

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
│   │   ├── defaults/
│   │   │   └── main.yml     # nvm, Node.js, tfenv, and Terraform versions
│   │   └── tasks/
│   │       ├── main.yml     # Base packages, AWS CLI
│   │       ├── node.yml     # Node.js through nvm (dev only)
│   │       └── terraform.yml # Terraform through tfenv (dev only)
│   ├── docker/
│   │   └── tasks/
│   │       └── main.yml     # Docker installation
│   └── dev_scripts/
│       ├── defaults/
│       │   └── main.yml     # Where dev-scripts comes from and goes
│       └── tasks/
│           └── main.yml     # Clone dev-scripts, run its install.sh
└── README.md
```

## Ansible Collections Used

- **community.docker** - Docker container, image, and network management
- **community.general** - General utilities and helpers
