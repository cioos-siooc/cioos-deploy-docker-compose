# CIOOS Deploy Docker Compose

A GitHub composite action that deploys a Docker Compose project to a remote server via SSH. Optionally connects through a WireGuard VPN and injects secrets from 1Password before deployment.

## What it does

1. (Optional) Connects to a WireGuard VPN
2. Copies files from the runner to the remote host via SCP
3. (Optional) Injects 1Password secrets into specified files on the remote host
4. Runs `docker compose build --pull`, optionally `down --remove-orphans`, then `up -d --remove-orphans`
5. Prunes dangling Docker images
6. (Optional) Tears down WireGuard on completion

## Prerequisites

- The remote host must have Docker and Docker Compose installed
- If using 1Password, the `op` CLI must be installed on the remote host

## Usage

### Basic deployment to a public server

```yaml
steps:
  - uses: actions/checkout@v4

  - uses: cioos/cioos-deploy-docker-compose@main
    with:
      ssh_host: "203.0.113.10"
      ssh_username: "deploy"
      ssh_key: ${{ secrets.SSH_KEY }}
      deploy_path: "/opt/myapp"
      stack_name: "myapp"
```

### Deployment over WireGuard VPN

```yaml
steps:
  - uses: actions/checkout@v4

  - uses: cioos/cioos-deploy-docker-compose@main
    with:
      wg_config: ${{ secrets.WG_CONFIG }}
      ssh_host: "10.0.0.1"
      ssh_username: "deploy"
      ssh_key: ${{ secrets.SSH_KEY }}
      deploy_path: "/opt/myapp"
      stack_name: "myapp"
```

### Full example with 1Password, multiple compose files, and down before up

```yaml
steps:
  - uses: actions/checkout@v4

  - uses: cioos/cioos-deploy-docker-compose@main
    with:
      wg_config: ${{ secrets.WG_CONFIG }}
      ssh_host: "10.0.0.1"
      ssh_username: "deploy"
      ssh_key: ${{ secrets.SSH_KEY }}
      ssh_passphrase: ${{ secrets.SSH_PASSPHRASE }}
      source_dir: "."
      deploy_path: "/opt/myapp"
      stack_name: "myapp"
      compose_files: "docker-compose.yml docker-compose.prod.yml"
      op_secret_files: ".env config/secrets.yml"
      op_service_account_token: ${{ secrets.OP_SERVICE_ACCOUNT_TOKEN }}
      docker_compose_down: "true"
```

### Deploying multiple instances on the same host

Use different `deploy_path` and `stack_name` values per branch/environment:

```yaml
# Branch: staging
- uses: cioos/cioos-deploy-docker-compose@main
  with:
    ssh_host: "10.0.0.1"
    ssh_username: "deploy"
    ssh_key: ${{ secrets.SSH_KEY }}
    deploy_path: "/opt/myapp/staging"
    stack_name: "myapp-staging"
```

```yaml
# Branch: production
- uses: cioos/cioos-deploy-docker-compose@main
  with:
    ssh_host: "10.0.0.1"
    ssh_username: "deploy"
    ssh_key: ${{ secrets.SSH_KEY }}
    deploy_path: "/opt/myapp/production"
    stack_name: "myapp-production"
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `ssh_host` | Yes | | Remote SSH host |
| `ssh_username` | Yes | | Remote SSH username |
| `ssh_key` | Yes | | SSH private key |
| `deploy_path` | Yes | | Remote path where the project gets deployed |
| `stack_name` | Yes | | Docker Compose project name |
| `ssh_port` | No | `22` | Remote SSH port |
| `ssh_passphrase` | No | | SSH key passphrase |
| `source_dir` | No | `.` | Path on the runner to copy from |
| `compose_files` | No | `docker-compose.yml` | Space-separated list of compose files |
| `docker_compose_down` | No | `false` | Run `docker compose down` before `up` |
| `wg_config` | No | | WireGuard config file content (skipped if empty) |
| `wireguard_interface` | No | `wg0` | WireGuard interface name |
| `op_secret_files` | No | | Space-separated list of files to inject 1Password secrets into |
| `op_service_account_token` | No | | 1Password service account token |
