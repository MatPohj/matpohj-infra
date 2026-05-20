# mng

Ansible playbook for provisioning the MatPohj VPS.

## Logging and SIEM pipeline

This repository now includes a `logging` role that can ship security-relevant logs to a remote Loki instance with Promtail.

Log sources:

- `/var/log/nginx/matpohj.access.log`
- `/var/log/nginx/matpohj.error.log`
- `/var/log/auth.log`
- `/var/log/fail2ban.log`
- `/var/log/ufw.log`

The Nginx access log is emitted as JSON so Grafana/Loki queries can easily filter by:

- client IP
- request method
- request path
- status code
- referrer
- user agent

### Required inventory variables

Set these in inventory or vaulted group vars before running the playbook:

```yaml
logging_enabled: true
logging_loki_url: "https://logs.example.com/loki/api/v1/push"
logging_loki_username: "tenant-or-user"
logging_loki_password: "replace-me"
```

Optional GeoIP enrichment in Promtail requires a MaxMind GeoLite2 City database already present on the target host:

```yaml
logging_geoip_enabled: true
logging_geoip_db_path: /etc/promtail/GeoLite2-City.mmdb
```

### Example questions to answer in Grafana/Loki

- Top source IPs hitting the site:
  - `topk(10, sum by (remote_addr) (count_over_time({job="nginx_access"} | json [24h])))`
- Suspicious probes for `.env`, `.git`, `wp-login.php`, `admin`:
  - `{job="nginx_access"} | json | request_path=~".*(\\.env|\\.git|wp-login|admin).*"`
- Repeated 401/403/404 responses:
  - `{job="nginx_access"} | json | status=~"401|403|404"`
- Fail2ban bans over time:
  - `{job="fail2ban"} |= " Ban "`
- SSH brute-force activity:
  - `{job="auth"} |= "Failed password"`
- Top source countries after GeoIP enrichment:
  - `topk(10, sum by (geoip_country_name) (count_over_time({job="nginx_access"} | json [24h])))`

### Notes

- The playbook installs Promtail from Grafana's APT repository.
- The JSON Nginx log format is always installed so the access/error logs are ready for analysis.
- Promtail log shipping activates when `logging_enabled` is `true`.
- For low-resource VPS hosts, ship logs off-box to a separate Loki/Grafana deployment instead of hosting the full analytics stack on the web server.
