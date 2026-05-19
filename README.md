# mng

Single-host provisioning for a VPS: SSH hardening, UFW, Fail2ban, Nginx + Certbot, and a static site deploy.

## Requirements
- Ansible installed on the control machine
- Target host has Python 3 available at `/usr/bin/python3`
- Collections:
	- community.general
	- ansible.posix

Install collections:
```
ansible-galaxy collection install community.general ansible.posix
```

## Setup
- Put your host in `hosts.ini`
- Ensure you can SSH as `sysadmin`
- Place your SSH public key at `id_ed25519.pub` in the repo root, or override `ssh_public_key_path`

## Run
```
ansible-playbook setup.yml
```

## Variables
- `ssh_public_key_path`: Path to the public key file used for `sysadmin` (default: `{{ playbook_dir }}/id_ed25519.pub`)