# mng

Ansible playbook that provisions, hardens and deploys matpohj.fi on a VPS.

| Role | What it does |
| --- | --- |
| `hardening` | UFW firewall, `sysadmin` user with SSH key and passwordless sudo, root login and SSH password login disabled |
| `monitoring` | fail2ban with the jail config in `roles/monitoring/templates/jail.local.j2` |
| `deploy` | nginx, certbot, clones `MatPohj/PersonalWebsite` (`main`) and serves it over HTTPS |

## Setup on the control machine

```sh
ansible-galaxy collection install -r requirements.yml
```

## New server

1. Create the VPS with your SSH key for `root`.
2. Point the DNS A records for `matpohj.fi` and `www.matpohj.fi` at it. certbot needs this to issue the certificate.
3. Put the new IP in `hosts.ini`.
4. First run, as root. The `sysadmin` user does not exist yet:

   ```sh
   ansible-playbook setup.yml -e ansible_user=root
   ```

   After this run root login and SSH password login are disabled.
5. Run again as `sysadmin` (the default). This should report no changes except possibly the website:

   ```sh
   ansible-playbook setup.yml
   ```

## Publishing the website

Push to `main` in `MatPohj/PersonalWebsite`, then run only the deploy role:

```sh
ansible-playbook setup.yml --tags deploy
```

Other tags: `hardening`, `monitoring`.
