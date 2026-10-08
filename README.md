# matpohj-infra

Ansible playbook that turns a fresh Ubuntu VPS into a hardened web server and deploys [matpohj.fi](https://matpohj.fi) to it.

The whole server is described in code: create a VPS, point DNS at it, run one command, and you get the same locked-down machine serving the site over HTTPS every time. Nothing is configured by hand, so the server can be thrown away and rebuilt in minutes. The site itself lives in a separate repository, [MatPohj/PersonalWebsite](https://github.com/MatPohj/PersonalWebsite), and this playbook pulls it onto the server.

It is my personal infrastructure, but the roles are parameterised, so you can use it for your own static site, or read it as a small example of a hardening and deployment playbook.

## What you get

| Role | What it does |
| --- | --- |
| `hardening` | UFW firewall, `sysadmin` user with SSH key and passwordless sudo, root login and SSH password login disabled |
| `monitoring` | fail2ban with the jail config in `roles/monitoring/templates/jail.local.j2` |
| `deploy` | nginx, certbot, clones the site repository and serves it over HTTPS |

After a run the server:

- accepts incoming traffic only on SSH (22), HTTP (80) and HTTPS (443), and denies everything else
- accepts SSH logins only with a key, and only as `sysadmin`, never as root
- bans an IP for 1 hour after 3 failed SSH logins within 10 minutes (fail2ban, enforced through UFW)
- serves the site with nginx using a Let's Encrypt certificate, renewed automatically by the certbot package's systemd timer

## Requirements

- **Server:** a fresh Ubuntu VPS (tested on Ubuntu 26.04) with root SSH access using a key. The playbook relies on apt, UFW and cloud-init, so other distributions will not work without changes.
- **Control machine:** Ansible (tested with ansible-core 2.20) and the collections in `requirements.yml`:

  ```sh
  ansible-galaxy collection install -r requirements.yml
  ```

- **Domain:** DNS records you can point at the server. certbot needs them before it can issue the certificate.

## Using it for your own server

> **Replace `id_ed25519.pub` with your own public key before the first run.**
> The `hardening` role installs this file as the only login key for `sysadmin` and then disables root login and password login. If you run it with my key, you will be locked out of your server.

Then change the settings that point at my setup:

- `hosts.ini`: your server's IP address.
- Site variables in `roles/deploy/defaults/main.yml`:

  | Variable | Meaning | Default |
  | --- | --- | --- |
  | `domain` | Primary domain; also used for the vhost file name, web root and certificate path | `matpohj.fi` |
  | `domain_aliases` | Extra names served by the same site and included in the certificate | `[www.matpohj.fi]` |
  | `site_repo` | Git repository with the static site | `https://github.com/MatPohj/PersonalWebsite.git` |
  | `site_version` | Branch, tag or commit to deploy | `main` |
  | `web_root` | Directory nginx serves | `/var/www/{{ domain }}/html` |
  | `certbot_email` | Let's Encrypt contact address; empty registers without one | `""` |

  Edit the file, set them in the inventory, or pass them on the command line:

  ```sh
  ansible-playbook setup.yml \
    -e domain=example.com \
    -e '{"domain_aliases": ["www.example.com"]}' \
    -e site_repo=https://github.com/you/your-site.git \
    -e certbot_email=you@example.com
  ```

The site repository is served as is: there is no build step, so it must contain the finished static files (an `index.html` at the top level).

## New server

1. Create the VPS with your SSH key for `root`.
2. Point the DNS A records for your domain and its aliases (here `matpohj.fi` and `www.matpohj.fi`) at it. certbot needs this to issue the certificate.
3. Put the new IP in `hosts.ini`.
4. Connect once with `ssh root@<ip>`, check the host key fingerprint against the one your provider shows, and accept it. Host key checking is on, so Ansible will not connect to a server it does not recognise.
5. First run, as root. The `sysadmin` user does not exist yet:

   ```sh
   ansible-playbook setup.yml -e ansible_user=root
   ```

   After this run root login and SSH password login are disabled.
6. Run again as `sysadmin` (the default). This should report no changes except possibly the website:

   ```sh
   ansible-playbook setup.yml
   ```

## Publishing the website

Push to `main` in the site repository, then run only the deploy role:

```sh
ansible-playbook setup.yml --tags deploy
```

Other tags: `hardening`, `monitoring`.

## Upgrading packages

Security updates are installed automatically by unattended-upgrades. To apply
all other pending updates, run the separate upgrade playbook. It reboots the
server only if an update requires it:

```sh
ansible-playbook upgrade.yml
```

## Design notes

Some problems only show up on a brand-new server. The playbook handles them so the first run works without manual fixes:

- **cloud-init holds the apt lock.** On first boot, cloud-init and unattended-upgrades are still installing packages, which makes the first apt task fail. The playbook waits for `cloud-init status --wait` before doing anything else.
- **Cloud images re-enable password login.** Many images ship `/etc/ssh/sshd_config.d/50-cloud-init.conf` with `PasswordAuthentication yes`. sshd uses the first value it reads, so that file silently overrides changes to `sshd_config`. The playbook adds a drop-in that sorts first (`00-hardening.conf`), then checks the settings sshd actually uses with `sshd -T` and fails the run if hardening did not apply.
- **The certificate does not exist yet.** nginx refuses to start with a config that points at missing certificate files. On the first run the playbook installs an HTTP-only vhost, gets the certificate with certbot, and switches to the full HTTPS vhost on later runs.
- **The firewall is set up before nginx is installed.** It opens ports 80 and 443 directly, because UFW's `Nginx Full` profile only exists after nginx is installed.

Trade-off: `sysadmin` has passwordless sudo. The account has no password and can only be reached with the SSH key, so this is what lets Ansible's `become` keep working after password login is disabled. Treat that key as root access.

## License

GPL-3.0. See [LICENSE](LICENSE).
