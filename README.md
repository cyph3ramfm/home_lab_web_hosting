
# Ansible Home Lab Deployment

Automated Ansible playbook for deploying a home-lab stack (Docker containers) with Traefik, Pi-hole, Portainer, CrowdSec and lightweight dashboards.

Project Structure

`deploy_home_lab_web_hosting_playbook.yml`    # Main deployment playbook
`group_vars/`
- `main.yml`                    # Default configuration variables (feature flags, paths, domain prefixes)
- `vault.yml`                   # Sensitive data (credentials, API keys) — encrypted at runtime
- `vault.yml.template`          # Template for creating `vault.yml`
`inventory/`
- `hosts`                       # Target machine definitions
`roles/`
- `base_network_watchtower/`    # Network setup and Watchtower (container updates)
	- `tasks/` and `templates/`
- `core_infrastructure/`        # Core services (Traefik, Pi-hole, Portainer, CrowdSec)
	- `tasks/` and `templates/`
- `dashboards/`                 # Monitoring (Homepage, Glances, other dashboards)
	- `tasks/` and `templates/`

Key Features

- Core Infrastructure
	- Traefik: Reverse proxy with automatic TLS using Cloudflare DNS-01 (requires Cloudflare API token/email in vault)
	- Pi-hole + cloudflared: Local DNS and ad-blocking; cloudflared used for DNS over HTTPS
	- Portainer: Container management UI
	- CrowdSec: Optional security/bouncer integration
- Dashboards
	- Homepage: Customizable service dashboard (templates under `roles/dashboards/templates`)
	- Glances: System monitoring

Prerequisites

- Ubuntu / Debian-based host (playbook uses systemd and manipulates `systemd-resolved`)
- Docker Engine (20.x+ recommended) and Docker Compose v2 (or Docker CLI with compose support)
- Python 3 and Ansible (Ansible 2.10+ recommended)
- `community.docker` Ansible collection:

```bash
ansible-galaxy collection install community.docker
```

- A registered domain and Cloudflare account (for Traefik DNS-01). No-IP support can be added externally if needed.
- Sudo / become access for the deployment user on target hosts (playbook uses privilege escalation for system-level changes)

Getting Started

Clone the repository and change directory:

```bash
git clone <repository-url>
cd home_lab_web_hosting
```

Create and encrypt your vault (sensitive variables):

```bash
# Copy template and edit before encrypting, or create directly
cp group_vars/vault.yml.template group_vars/vault.yml

# Create a secure vault password file (keep safe)
openssl rand -base64 32 > ~/.vault_pass.txt
chmod 600 ~/.vault_pass.txt

# Encrypt the vault using the password file
ansible-vault encrypt --vault-password-file ~/.vault_pass.txt group_vars/vault.yml

# Edit the vault later (decrypts, opens editor, re-encrypts)
ansible-vault edit --vault-password-file ~/.vault_pass.txt group_vars/vault.yml
```

Configure deployment:

- Edit `inventory/hosts` to specify target hosts
- Update `group_vars/main.yml` for non-sensitive configuration (paths, domain prefixes, feature toggles)

Run the deployment (interactive brief):

```bash
ansible-playbook -i inventory/hosts \
	--vault-password-file ~/.vault_pass.txt \
	--ask-become \
	deploy_home_lab_web_hosting_playbook.yml
```

Dry-run with diffs (checks only):

```bash
ansible-playbook -i inventory/hosts \
	--ask-vault-pass --check --diff \
	deploy_home_lab_web_hosting_playbook.yml
```

Notes & Repo-specific gotchas

- Traefik dashboard password: `vault_traefik_dashboard_password` in `group_vars/vault.yml.template` expects a BASE-20 formatted password. If the value contains `$`, you must escape each `$` by doubling it (e.g., `$$`) to avoid login issues.
- systemd-resolved: The playbook disables `DNSStubListener` and reconfigures `/etc/resolv.conf` so Pi-hole can bind to port 53. Do not run on non-systemd systems without adapting the tasks.
- Template render pattern: Many tasks render templates to `/tmp/<role>/...` and then call `community.docker.docker_compose_v2` against those rendered files. When `debug_mode: true` (in `group_vars/main.yml`) the `/tmp` files are preserved for inspection.

Troubleshooting

- Permission errors creating `/var/docker_data/...`: run the playbook with `--ask-become` so Ansible can escalate privileges to create system directories.
- Traefik certificate issues: verify `vault_cf_api_email` and `vault_cf_dns_api_token` exist and are correct in `group_vars/vault.yml`.
- DNS / network issues: ensure Docker networks exist (the playbook will create `proxy` network named by `proxy_network_name` if missing). Ensure Pi-hole is healthy before switching system resolver (playbook waits for DNS on localhost).
- Debug rendered compose files:

```bash
ls /tmp/* | grep core_infrastructure || true
# Inspect rendered compose files if debug_mode=true
```

Development / Extending

- Add a new service by creating role tasks and a Jinja2 compose template in the appropriate `roles/<role>/` folder and expose config variables in `group_vars/main.yml`.
- Follow existing patterns: check for `<service>_info` via `community.docker.*_info` modules, create directories with `become: true`, render templates to `/tmp/<role>`, then deploy with `docker_compose_v2` and guard with `when:` conditions to keep idempotency.

License

This project is provided under the MIT License. See the `LICENSE` file for details.

**CrowdSec Traefik bouncer registration**

- Start the full stack (all containers) first so Traefik, CrowdSec and the traefik-bouncer container are running.
- On the host where the `crowdsec` container runs, register the Traefik bouncer and capture the API key:

```bash
sudo docker exec crowdsec cscli bouncers add bouncer-traefik
```

- The command prints a bouncer key in the console. Copy that key and insert it into your `group_vars/vault.yml` as the value for `vault_crowdsec_bouncer_api_key`.

- After updating `group_vars/vault.yml` (or the encrypted vault), re-encrypt if needed, then redeploy the infrastructure so the traefik-bouncer is recreated with the registered key. Example (interactive run):

```bash
# Edit vault (or copy template and set value)
ansible-vault edit --vault-password-file ~/.vault_pass.txt group_vars/vault.yml

# Delete the traefik-bouncer container and re-run the playbook to apply the new bouncer key
ansible-playbook -i inventory/hosts \
	--vault-password-file ~/.vault_pass.txt \
	--ask-become \
	deploy_home_lab_web_hosting_playbook.yml
```

- Important: if the traefik-bouncer is not registered with CrowdSec (missing or wrong `vault_crowdsec_bouncer_api_key`), Traefik may not correctly receive CrowdSec decisions which can break networking (including internet access) for services behind the proxy. Registering the bouncer and redeploying with the correct key is required for proper networking.
