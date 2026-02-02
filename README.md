# Ansible Playbook for BAR Live Services

This is an [Ansible](https://en.wikipedia.org/wiki/Ansible_(software)) playbook for setting up the BAR Live Services server (bar-rts.com).

## Architecture

The playbook deploys the following containerized services using Podman Quadlet:

- **PostgreSQL** - Database for replay metadata, maps, and user data
- **Redis** - Caching and session storage
- **bar-db** - REST API backend serving replay data
- **bar-db-processor** - Background service processing replays and maps
- **bar-live-services** - Nuxt.js frontend application
- **Caddy** - Reverse proxy with automatic HTTPS

Additionally:
- **SFTP** - Secure upload endpoint for replay files from SPADS servers
- **Backup** - Scheduled PostgreSQL backups with rclone to remote storage
- **Monitoring** - Optional integration with central monitoring server

## Prerequisites

- Ansible 2.15+
- Target server running Debian 12+ (Trixie recommended)
- SSH access to target server

## Usage

### Production Deployment

1. Configure inventory in `inventory/` directory
2. Create and encrypt vault secrets:
   ```bash
   cp group_vars/prod/vault.yml.example group_vars/prod/vault.yml
   # Edit vault.yml with your secrets
   ansible-vault encrypt group_vars/prod/vault.yml
   ```
3. Install required collections:
   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```
4. Run the playbook:
   ```bash
   ansible-playbook -l prod play.yml --ask-vault-pass
   ```

### Local Testing

We use [Incus](https://linuxcontainers.org/incus/) for local testing. Make sure you have it installed and initialized following [the official getting started docs](https://linuxcontainers.org/incus/docs/main/tutorial/first_steps/).

#### Setup

Create a new container and initialize it via cloud-init:

```bash
touch .incus-integration-on && \
chmod 0600 test.ssh.key && \
incus launch images:debian/trixie/cloud bar-live-srv-test < test.incus.yml && \
incus exec bar-live-srv-test -- cloud-init status --wait
```

Test that ansible can connect:

```bash
ansible dev -m ping
```

Add local domain to `/etc/hosts`:

```bash
echo "$(incus list bar-live-srv-test -f csv -c 4 | grep eth | cut -d' ' -f1) live-srv.bar-mon.local" | sudo tee -a /etc/hosts
```

#### Run Playbook

```bash
ansible-playbook -l dev play.yml
```

#### Access Services

- Frontend: https://live-srv.bar-mon.local
- API: https://live-srv.bar-mon.local/api/

#### SSH Access

```bash
ssh -i test.ssh.key ansible@$(incus list bar-live-srv-test -f csv -c 4 | grep eth | cut -d' ' -f1)
```

Or enter container shell directly:

```bash
incus exec bar-live-srv-test -- /bin/bash
```

#### Cleanup

```bash
incus stop bar-live-srv-test && incus delete bar-live-srv-test
```

## Configuration

### Required Secrets (Production)

| Variable | Description |
|----------|-------------|
| `vault_postgres_password` | PostgreSQL database password |
| `vault_sftp_upload_password` | SFTP password for replay uploads |
| `vault_bar_db_github_token` | GitHub token for balance change processing |
| `vault_bar_db_lobby_username` | Teiserver bot username |
| `vault_bar_db_lobby_password` | Teiserver bot password |
| `vault_bar_db_google_sheets_id` | Google Sheets ID for map lists |
| `vault_bar_db_google_sheets_api_key` | Google Sheets API key |
| `vault_backup_*` | Backup storage credentials |

### Configurable Variables

See `group_vars/all.yml` for all available configuration options.

## SFTP Replay Uploads

SPADS servers upload replays via SFTP to port 21344. The upload script should use:

```bash
sshpass -p "$SFTP_PASSWORD" sftp -P 21344 replay-upload@bar-rts.com <<EOF
cd /demos
put "$REPLAY_FILE"
EOF
```

## Related Repositories

- [bar-db](https://github.com/beyond-all-reason/bar-db) - Backend source code
- [bar-live-services](https://github.com/beyond-all-reason/bar-live-services) - Frontend source code
- [ansible-monitoring](https://github.com/beyond-all-reason/ansible-monitoring) - Central monitoring server
