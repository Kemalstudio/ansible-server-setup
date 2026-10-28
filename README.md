# ansible-server-setup

![Lint](https://github.com/Kemalstudio/ansible-server-setup/actions/workflows/lint.yml/badge.svg)

Ansible playbook that turns a fresh **Ubuntu Server** into a secured, ready-to-use machine with one command.

```bash
ansible-playbook site.yml
```

## What it does

| Role | Result |
| :--- | :--- |
| `common` | System upgrade, base packages, timezone, automatic security updates |
| `users` | Admin user with sudo and SSH key login |
| `ssh` | Hardened SSH: no root login, no passwords, key-only access, limited auth tries |
| `firewall` | UFW: deny all incoming except SSH, HTTP, HTTPS; SSH rate limiting |
| `fail2ban` | Bans IPs after repeated failed SSH logins |
| `docker` | Docker Engine + Compose plugin from the official repository |
| `nginx` | Nginx with hidden version and security headers |

Every role can be run separately with tags, and Docker / Nginx can be switched off in `group_vars/all.yml`.

## Safety measures

- **No lockout:** the SSH role checks that the admin user's key is installed before disabling password login
- **Config validation:** SSH config is checked with `sshd -t`, sudoers with `visudo` before being applied
- **SSH stays open:** the firewall allows the SSH port before enabling UFW
- **No secrets in git:** the real inventory with server IPs (`inventory/hosts.ini`) is ignored
- **CI:** every push runs `yamllint`, a syntax check and `ansible-lint` (production profile)

## Project structure

```
ansible-server-setup/
├── site.yml                  # main playbook
├── ansible.cfg
├── requirements.yml          # required Ansible collections
├── inventory/
│   └── hosts.ini.example     # copy to hosts.ini
├── group_vars/
│   └── all.yml               # all settings in one place
├── roles/
│   ├── common/  users/  ssh/  firewall/
│   └── fail2ban/  docker/  nginx/
└── .github/workflows/lint.yml
```

## Quick start

### 1. Control machine
Ansible runs from Linux: WSL on Windows, a Linux VM, or Kali.

```bash
sudo apt install -y ansible
ansible-galaxy collection install -r requirements.yml
```

You need an SSH key on the control machine:
```bash
ssh-keygen -t ed25519
```

### 2. Inventory
```bash
cp inventory/hosts.ini.example inventory/hosts.ini
```
Put your server address and current user into `hosts.ini`.

### 3. First run
On a fresh server you still log in with a password:
```bash
ansible-playbook site.yml --ask-pass --ask-become-pass
```

After this run the server only accepts SSH keys for the admin user (`devops` by default).
Change `ansible_user` in `hosts.ini` to `devops` for the next runs:
```bash
ansible-playbook site.yml
```

### Useful commands
```bash
ansible-playbook site.yml --check --diff     # dry run: show what would change
ansible-playbook site.yml --tags ssh,firewall  # run only selected roles
ansible servers -m ping                      # test connection
```

## Verify the result

```bash
ssh devops@<server>              # works with key
ssh root@<server>                # denied
sudo ufw status verbose          # only 22, 80, 443 open
sudo fail2ban-client status sshd # jail is active
docker --version && nginx -v
```

## Configuration

All settings live in `group_vars/all.yml`:

| Variable | Default | Description |
| :--- | :--- | :--- |
| `admin_user` | `devops` | Admin user to create |
| `admin_ssh_public_key` | `~/.ssh/id_ed25519.pub` | Key allowed to log in |
| `ssh_port` | `22` | SSH port |
| `firewall_allowed_tcp_ports` | `22, 80, 443` | Open ports |
| `fail2ban_maxretry` | `5` | Failed logins before ban |
| `install_docker` / `install_nginx` | `true` | Optional components |

## Roadmap

- [ ] Test with Molecule in Docker
- [ ] Role for PostgreSQL
- [ ] Automatic Let's Encrypt certificates
- [ ] Node exporter for Prometheus monitoring

## Supported systems

- Ubuntu Server 22.04 LTS / 24.04 LTS
