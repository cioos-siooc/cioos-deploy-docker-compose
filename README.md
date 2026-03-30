# CIOOS Deploy Docker Compose

A GitHub composite action that deploys a Docker Compose project to a remote server via SSH. Optionally connects through a WireGuard VPN and injects secrets from 1Password before deployment.

- [What it does](#what-it-does)
- [Usage](#usage)
- [Inputs](#inputs)
- [Deploy summary](#deploy-summary)
- [Host setup](#host-setup)

## What it does

1. (Optional) Connects to a WireGuard VPN
2. Syncs the repository to the remote host via SSH + Git
3. (Optional) Injects 1Password secrets into specified files on the remote host
4. Runs `docker compose build --pull`, optionally `down --remove-orphans`, then `up -d --remove-orphans`
5. Prunes dangling Docker images
6. Writes a deploy summary to the GitHub Actions job summary (including failure details on error)
7. (Optional) Tears down WireGuard on completion

## Usage

### Basic deployment to a public server

```yaml
steps:
  - uses: actions/checkout@v4

  - uses: cioos-siooc/cioos-deploy-docker-compose@main
    with:
      ssh_host: "203.0.113.10"
      ssh_username: "github-deploy"
      ssh_key: ${{ secrets.SSH_KEY }}
      deploy_path: "/opt/myapp"
      stack_name: "myapp"
```

### Deployment over WireGuard VPN

```yaml
steps:
  - uses: actions/checkout@v4

  - uses: cioos-siooc/cioos-deploy-docker-compose@main
    with:
      wg_config: ${{ secrets.WG_CONFIG }}
      ssh_host: "10.0.0.1"
      ssh_username: "github-deploy"
      ssh_key: ${{ secrets.SSH_KEY }}
      deploy_path: "/opt/myapp"
      stack_name: "myapp"
```

### Full example with 1Password, multiple compose files, and down before up

```yaml
steps:
  - uses: actions/checkout@v4

  - uses: cioos-siooc/cioos-deploy-docker-compose@main
    with:
      wg_config: ${{ secrets.WG_CONFIG }}
      ssh_host: "10.0.0.1"
      ssh_username: "github-deploy"
      ssh_key: ${{ secrets.SSH_KEY }}
      deploy_path: "/opt/myapp"
      stack_name: "myapp"
      compose_files: "docker-compose.yml docker-compose.prod.yml"
      op_secret_files: ".env config/secrets.yml"
      op_service_account_token: ${{ secrets.OP_SERVICE_ACCOUNT_TOKEN }}
      docker_compose_down: "true"
```

### 1Password secret injection with file renaming

Use `source:destination` syntax to inject secrets from a template file into a differently named output file. This is useful when your repo contains `.tpl` templates that should produce plain config files on the server.

```yaml
op_secret_files: ".env.tpl:.env config/secrets.yml.tpl:config/secrets.yml"
```

You can mix in-place and renamed entries:

```yaml
op_secret_files: ".env.tpl:.env config/app.yml"
```

### Deploying multiple instances on the same host

Use different `deploy_path` and `stack_name` values per branch/environment:

```yaml
# Branch: staging
- uses: cioos-siooc/cioos-deploy-docker-compose@main
  with:
    ssh_host: "10.0.0.1"
    ssh_username: "github-deploy"
    ssh_key: ${{ secrets.SSH_KEY }}
    deploy_path: "/opt/myapp/staging"
    stack_name: "myapp-staging"
```

```yaml
# Branch: production
- uses: cioos-siooc/cioos-deploy-docker-compose@main
  with:
    ssh_host: "10.0.0.1"
    ssh_username: "github-deploy"
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
| `compose_files` | No | `docker-compose.yml` | Space-separated list of compose files |
| `docker_compose_down` | No | `false` | Run `docker compose down` before `up` |
| `wg_config` | No | | WireGuard config file content (skipped if empty) |
| `wireguard_interface` | No | `wg0` | WireGuard interface name |
| `op_secret_files` | No | | Space-separated list of files to inject 1Password secrets into. Use `source:destination` to rename (e.g. `.env.tpl:.env`), or just `file` to inject in place. |
| `op_service_account_token` | No | | 1Password service account token |

## Deploy summary

Every run writes a summary table to the GitHub Actions job summary, visible on the workflow run page. The summary includes the stack name, host, branch, compose files, timestamps, and commit link.

On failure, the summary includes a **Failure Details** section with container status from the remote host to help diagnose issues without digging through logs.

## Host setup

### Create a github-deploy user

```bash
sudo useradd -m -s /bin/bash github-deploy
sudo usermod -aG docker github-deploy
```

The user must be in the `docker` group to run `docker compose` commands without `sudo`.

### Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo systemctl enable --now docker
```

### Install Git

Git must be installed on the remote host for the repository sync step:

```bash
sudo apt-get install -y git
```

### Install 1Password CLI (optional)

If you plan to use 1Password secret injection, install the `op` CLI on the host:

```bash
curl -sS https://downloads.1password.com/linux/keys/1password.asc | \
  sudo gpg --dearmor --output /usr/share/keyrings/1password-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/1password-archive-keyring.gpg] https://downloads.1password.com/linux/debian/$(dpkg --print-architecture) stable main" | \
  sudo tee /etc/apt/sources.list.d/1password-cli.list
sudo apt-get update && sudo apt-get install -y 1password-cli
```

Verify the installation:

```bash
op --version
```

The `OP_SERVICE_ACCOUNT_TOKEN` is passed to the host at runtime via the action — no need to persist it on the server.
