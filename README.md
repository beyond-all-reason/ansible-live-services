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

- Ansible 2.17+
- Target server running Debian 13 (Trixie)
- SSH access to target server (production only; local testing uses Incus)

## Usage

### Production Deployment

First-time setup:

```bash
# Install required Ansible collections
ansible-galaxy collection install -r requirements.yml

# Create and encrypt vault secrets (see group_vars/all.yml for required variables)
ansible-vault create group_vars/prod/vault.yml
```

Run the playbook:

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

Add local domain to `/etc/hosts`. This is required because Caddy binds to the host's ports 80/443 and uses SNI to route traffic. Connecting directly via container IP is not sufficient since Caddy needs the correct hostname for TLS certificate provisioning and virtual host routing:

```bash
ansible localhost -b -m lineinfile -a "path=/etc/hosts regexp='live-srv\.bar-mon\.local' line='$(incus list bar-live-srv-test -f csv -c 4 | grep eth | cut -d' ' -f1) live-srv.bar-mon.local api.live-srv.bar-mon.local'"
```

#### Run Playbook

```bash
ansible-playbook -l dev play.yml
```

#### Access Services

- Frontend: https://live-srv.bar-mon.local
- API: https://api.live-srv.bar-mon.local

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

See `group_vars/all.yml` for defaults and `group_vars/prod/vars.yml` for production overrides.

## SFTP Replay Uploads

SPADS servers upload replays via SFTP. The upload script should use:

```bash
sshpass -p "$SFTP_PASSWORD" sftp replay-upload@bar-rts.com <<EOF
cd /demos
put "$REPLAY_FILE"
EOF
```

## Related Repositories

- [bar-db](https://github.com/beyond-all-reason/bar-db) - Backend source code
- [bar-live-services](https://github.com/beyond-all-reason/bar-live-services) - Frontend source code
- [ansible-monitoring](https://github.com/beyond-all-reason/ansible-monitoring) - Central monitoring server
