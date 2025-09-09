# Ansible Podman Container Deployment

An Ansible playbook for deploying and managing Podman containers across different environments using host-specific variables.

## Overview

This playbook provides a declarative approach to deploy Podman containers using Ansible. Define your containers, networks, volumes, and configurations in host_vars files and let Ansible handle the deployment.

## Features

- **Simple Configuration**: Define containers in YAML host_vars
- **Idempotent Deployment**: Safe to run multiple times
- **Network Management**: Automatic network creation
- **Directory Setup**: Automatic directory creation with proper permissions
- **Health Checks**: Wait for containers to be ready
- **Quadlet Support**: Optional systemd integration via Quadlet
- **Environment-Specific**: Use different configs for dev/staging/prod

## Prerequisites

- Ansible >= 2.9
- Podman installed on target hosts
- Python podman module: `pip install podman`
- Ansible Podman collection: `ansible-galaxy collection install containers.podman`

## Quick Start

### 1. Clone the Repository

```bash
git clone <repository-url>
cd ansible-podman-deploy
```

### 2. Create Your Inventory

Create an inventory file `inventory.ini`:

```ini
[development]
dev ansible_host=localhost ansible_connection=local

[staging]
staging-server ansible_host=10.0.1.50

[production]
prod-server ansible_host=10.0.1.100
```

### 3. Configure Your Environment

Create a host_vars file for your target host. For example, `host_vars/dev.yml`:

```yaml
# host_vars/dev.yml
---
# Container management settings
pull_images: true
container_restart_policy: "on-failure"

# Directories to create
create_directories:
  - path: "/opt/openwebui/dev/ollama"
    mode: "0755"
  - path: "/opt/openwebui/dev" 
    mode: "0755"

# Container definitions
containers:
  - name: "ollama-dev"
    image: "docker.io/ollama/ollama:latest"
    state: "started"
    ports:
      - "21434:11434"
    volumes:
      - "/opt/openwebui/dev/ollama:/root/.ollama:Z"
    networks:
      - "openwebui-network-dev"
    devices:
      - "nvidia.com/gpu=all"
    security_opts:
      - "label=disable"
    environment:
      OLLAMA_SCHED_SPREAD: "true"
      OLLAMA_MAX_LOADED_MODELS: "1"
      OLLAMA_GPU_OVERHEAD: "2147483648"
      OLLAMA_FLASH_ATTENTION: "true"
      OLLAMA_KV_CACHE_TYPE: "f16"

  - name: "openwebui-dev"
    image: "ghcr.io/open-webui/open-webui:main"
    state: "started"
    ports:
      - "4001:4000"
    volumes:
      - "/opt/openwebui/dev:/app/backend/data:Z"
    networks: 
      - "openwebui-network-dev"
    environment:
      PORT: "4000"
      WEBUI_URL: "http://localhost:4001"
      OLLAMA_BASE_URL: "http://ollama-dev:21434"
```

### 4. Deploy

Run the playbook:

```bash
# Deploy to development
ansible-playbook -i inventory.ini deploy.yml -l development

# Deploy with specific tags
ansible-playbook -i inventory.ini deploy.yml -l development --tags containers

# Check what would be changed (dry run)
ansible-playbook -i inventory.ini deploy.yml -l development --check
```

## Adding New Hosts to Existing Inventory

### Step 1: Add Host to Inventory

Edit your `inventory.ini` file and add the new host under the appropriate group:

```ini
[development]
dev ansible_host=localhost ansible_connection=local
dev2 ansible_host=10.0.1.20  # New development server

[staging]
staging-server ansible_host=10.0.1.50
staging-server2 ansible_host=10.0.1.51  # New staging server

[production]
prod-server ansible_host=10.0.1.100
prod-server2 ansible_host=10.0.1.101  # New production server
prod-server3 ansible_host=10.0.1.102  # Another production server
```

### Step 2: Create Host Variables File

Create a new file in the `host_vars` directory matching the hostname from inventory:

```bash
# Create new host_vars file
touch host_vars/dev2.yml
```

### Step 3: Define Container Configuration

Edit the new host_vars file with your container definitions:

```yaml
# host_vars/dev2.yml
---
# Container management settings
pull_images: true
container_restart_policy: "on-failure"

# Directories to create
create_directories:
  - path: "/opt/myapp/data"
    mode: "0755"
  - path: "/opt/myapp/config"
    mode: "0755"

# Container definitions
containers:
  - name: "myapp-dev2"
    image: "docker.io/myapp:latest"
    state: "started"
    ports:
      - "8080:80"
    volumes:
      - "/opt/myapp/data:/app/data:Z"
      - "/opt/myapp/config:/app/config:ro"
    networks:
      - "myapp-network"
    environment:
      APP_ENV: "development"
      APP_PORT: "80"
```

### Step 4: Deploy to New Host

```bash
# Deploy to specific new host
ansible-playbook -i inventory.ini deploy.yml -l dev2

# Deploy to all hosts in a group
ansible-playbook -i inventory.ini deploy.yml -l development

# Deploy to multiple specific hosts
ansible-playbook -i inventory.ini deploy.yml -l "dev2,staging-server2"
```

### Step 5: Verify Deployment on New Host

```bash
# Check connectivity to new host
ansible -i inventory.ini dev2 -m ping

# Run ad-hoc command to check containers
ansible -i inventory.ini dev2 -m shell -a "podman ps"

# View deployment info
ansible-playbook -i inventory.ini deploy.yml -l dev2 --tags info
```

## Adding New Containers to Existing Host

To add a new container to an existing host configuration:

### Step 1: Edit Existing Host Variables

Open the host_vars file for your target host:

```yaml
# host_vars/dev.yml
containers:
  # ... existing containers ...
  
  # Add new container
  - name: "redis-dev"
    image: "docker.io/redis:alpine"
    state: "started"
    ports:
      - "6379:6379"
    volumes:
      - "/opt/redis/data:/data:Z"
    networks:
      - "openwebui-network-dev"
    environment:
      REDIS_PASSWORD: "development_password"
```

### Step 2: Add Required Directories

If the new container needs directories, add them to `create_directories`:

```yaml
create_directories:
  # ... existing directories ...
  - path: "/opt/redis/data"
    mode: "0755"
```

### Step 3: Deploy Only the New Container

```bash
# Deploy everything (will skip existing containers if no changes)
ansible-playbook -i inventory.ini deploy.yml -l dev

# Or use check mode first
ansible-playbook -i inventory.ini deploy.yml -l dev --check
```

## Configuration Reference

### Container Definition Options

Each container in the `containers` list supports the following options:

| Option | Description | Required | Example |
|--------|-------------|----------|---------|
| `name` | Container name | Yes | `"ollama-dev"` |
| `image` | Container image | Yes | `"docker.io/ollama/ollama:latest"` |
| `state` | Container state | No | `"started"` (default), `"stopped"`, `"quadlet"` |
| `ports` | Port mappings | No | `["8080:80", "8443:443"]` |
| `volumes` | Volume mounts | No | `["/host/path:/container/path:Z"]` |
| `networks` | Networks to join | No | `["app-network"]` |
| `environment` | Environment variables | No | `{KEY: "value"}` |
| `devices` | Device mappings | No | `["nvidia.com/gpu=all"]` |
| `security_opts` | Security options | No | `["label=disable"]` |
| `restart_policy` | Restart policy | No | `"always"`, `"on-failure"`, `"no"` |
| `quadlet_options` | Quadlet-specific options | No | `["AutoUpdate=registry"]` |

### Directory Creation Options

Each directory in `create_directories` supports:

| Option | Description | Required | Default |
|--------|-------------|----------|---------|
| `path` | Directory path | Yes | - |
| `mode` | Permissions | No | `"0755"` |
| `owner` | Owner user | No | Current user |
| `group` | Owner group | No | Current group |

## Deployment Examples

### Basic Web Application

```yaml
containers:
  - name: "webapp"
    image: "nginx:latest"
    state: "started"
    ports:
      - "8080:80"
    volumes:
      - "/opt/webapp/html:/usr/share/nginx/html:ro"
```

### Database with Persistent Storage

```yaml
containers:
  - name: "postgres-prod"
    image: "postgres:15"
    state: "started"
    ports:
      - "5432:5432"
    volumes:
      - "/opt/postgres/data:/var/lib/postgresql/data:Z"
    environment:
      POSTGRES_DB: "myapp"
      POSTGRES_USER: "myuser"
      POSTGRES_PASSWORD: "secure_password"
```

### Multi-Container Application Stack

```yaml
containers:
  # Backend API
  - name: "api"
    image: "myapp/api:latest"
    state: "started"
    ports:
      - "3000:3000"
    networks:
      - "app-network"
    environment:
      DATABASE_URL: "postgresql://postgres:5432/myapp"
      REDIS_URL: "redis://redis:6379"

  # Redis Cache
  - name: "redis"
    image: "redis:alpine"
    state: "started"
    networks:
      - "app-network"
    volumes:
      - "/opt/redis/data:/data:Z"

  # Frontend
  - name: "frontend"
    image: "myapp/frontend:latest"
    state: "started"
    ports:
      - "80:80"
    networks:
      - "app-network"
    environment:
      API_URL: "http://api:3000"
```

## Available Tags

Use tags to run specific parts of the playbook:

- `directories` - Create required directories
- `images` - Pull container images
- `networks` - Create container networks
- `containers` - Deploy containers
- `health_check` - Wait for containers to be ready
- `verification` - Verify container endpoints
- `quadlet` - Configure Quadlet services
- `info` - Display container information

Example usage:
```bash
# Only pull images and deploy containers
ansible-playbook -i inventory.ini deploy.yml --tags images,containers

# Skip health checks
ansible-playbook -i inventory.ini deploy.yml --skip-tags health_check,verification
```

## Managing Multiple Environments

Create separate host_vars for each environment:

```
host_vars/
├── dev.yml          # Development settings
├── staging.yml      # Staging settings
└── production.yml   # Production settings
```

Each environment can have different:
- Port mappings
- Resource limits
- Environment variables
- Volume mounts
- Network configurations

## Verifying Deployment

After deployment, verify containers are running:

```bash
# Check container status
podman ps

# Check specific container logs
podman logs ollama-dev

# Test container endpoint
curl http://localhost:4001

# For Quadlet deployments
systemctl status ollama-dev.service
```

## Troubleshooting

### Container won't start
- Check logs: `podman logs <container-name>`
- Verify image exists: `podman images`
- Check port conflicts: `netstat -tulpn | grep <port>`

### Network issues
- List networks: `podman network ls`
- Inspect network: `podman network inspect <network-name>`

### Permission issues
- Ensure SELinux labels on volumes (`:Z` or `:z` suffix)
- Check directory ownership and permissions

### Quadlet services - Work in Progress
- Check systemd status: `systemctl status <container-name>.service`
- View systemd logs: `journalctl -u <container-name>.service`
- Reload systemd: `systemctl daemon-reload`

## Best Practices

1. **Environment-Specific Variables**: Keep sensitive data in ansible-vault encrypted files
2. **Version Control**: Pin container image versions for production
3. **Health Checks**: Define health checks for critical services
4. **Resource Limits**: Set memory and CPU limits in production environments
5. **Logging**: Configure appropriate log drivers and retention policies
6. **Backups**: Implement regular backups for volumes containing persistent data
7. **Network Isolation**: Use separate networks for different application stacks
8. **Update Strategy**: Test updates in development before applying to production

## Directory Structure

```
.
├── ansible.cfg
├── inventory.ini
├── deploy.yml
├── host_vars/
│   ├── dev.yml
│   ├── staging.yml
│   └── production.yml
├── group_vars/
│   └── all.yml # Coming soon
└── roles/
    └── container_deployment/
        ├── tasks/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        └── defaults/
            └── main.yml
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

## Support

For issues, questions, or contributions, please [open an issue](https://github.com/your-repo/issues) on GitHub.
