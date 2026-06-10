# Centralized Logging & Audit Platform — Production Build Plan (10 Days)

> **Scope:** Real infrastructure, real pitfalls, real commands. This is not a tutorial — it is a build plan. Every day has a primary goal, exact steps, known failure modes, and validation gates. You do not move to the next day until the gate passes.

---

## Pre-Day-0 Checklist (Do This Before Day 1)

These are not Day 1 tasks. Resolve them in advance or you will lose a full day.

- [ ] Decide on OS versions NOW and commit — do not mix Ubuntu 22.04 and 24.04 without awareness of systemd/auditd differences
- [ ] Know your network topology: can the logging server reach all clients on port 3100 (Loki)? Firewall rules before you start
- [ ] DNS or `/etc/hosts` entries for all nodes — Ansible inventory breaks without stable hostnames
- [ ] SSH key-based auth from Ansible control node to all targets — no password prompts
- [ ] Decide on Loki version NOW — v2.x and v3.x have breaking config differences (see Day 2)
- [ ] Allocate disk for Loki: minimum 20GB separate volume for log storage, not `/`
- [ ] Time sync: if NTP is not already consistent across all nodes, logs will arrive out of order and correlations will be wrong
- [ ] Create a non-root `ansible` user with sudo NOPASSWD on all nodes before Day 1

---

## Architecture (Realistic)

```
┌──────────────────────────────────────────────────────────────┐
│  LOGGING SERVER                                              │
│                                                              │
│  Loki :3100  (ingest + query)                                │
│  Grafana :3000  (dashboards)                                 │
│  nginx :443  (reverse proxy, TLS termination)  ← add Day 5  │
└──────────────────────────────────────────────────────────────┘
           ▲               ▲               ▲
     Promtail          Promtail        Promtail
     Client 01         Client 02       Client 03
     (PostgreSQL)      (Nginx)         (App)
     port 3100         port 3100       port 3100
```

**Why no Kafka/message queue?** At lab scale (3 clients), Promtail → Loki direct push is fine. At 20+ nodes or high-volume logs, you would add an intermediate buffer. Document this decision.

---

## Label Strategy (Decide Now, Enforce Consistently)

Bad labels destroy LogQL queries and cause Loki cardinality issues. Define this before writing a single Promtail config.

| Label | Values | Notes |
|---|---|---|
| `hostname` | `server01`, `server02`, `server03` | From `__host__` or `/etc/hostname` |
| `env` | `lab`, `staging`, `prod` | Set in Ansible inventory vars |
| `service` | `ssh`, `sudo`, `postgresql`, `nginx`, `audit`, `system` | Per scrape job |
| `log_type` | `auth`, `audit`, `app`, `web`, `db` | Broad category |
| `severity` | `info`, `warn`, `error`, `critical` | Optional — only if you parse it |

**Rules:**
- Never use a label for high-cardinality values like IP addresses, usernames, or request IDs — those go in the log line, queried with `|=` or regex
- Every Promtail `scrape_config` must produce the same label set — gaps cause broken filters in Grafana

---

## Day 1 — Infrastructure & Connectivity

### Goal
All servers exist, are reachable, and have stable identities.

### Tasks

**1.1 — Provision servers**

Minimum specs:
- Logging server: 2 vCPU, 4GB RAM, 40GB disk (20GB separate for Loki data)
- Clients: 1 vCPU, 1GB RAM, 20GB disk

**1.2 — Set hostnames (on each node)**
```bash
hostnamectl set-hostname logging-server   # or client01, client02, client03
```

**1.3 — /etc/hosts entries (on all nodes)**
```
192.168.1.10  logging-server
192.168.1.11  client01
192.168.1.12  client02
192.168.1.13  client03
```

**1.4 — Firewall rules (logging server)**
```bash
# If using ufw
ufw allow 22/tcp
ufw allow 3100/tcp    # Loki ingest from clients
ufw allow 3000/tcp    # Grafana (restrict to your IP in prod)
ufw enable
```

**1.5 — Firewall rules (clients)**
```bash
# Clients only need outbound 3100 to logging-server
# If stateful firewall, outbound is usually allowed by default
# Verify with: nc -zv logging-server 3100
```

**1.6 — Ansible inventory**
```ini
# inventory/hosts.ini
[logging]
logging-server ansible_host=192.168.1.10

[clients]
client01 ansible_host=192.168.1.11
client02 ansible_host=192.168.1.12
client03 ansible_host=192.168.1.13

[clients:vars]
ansible_user=ansible
ansible_become=true
loki_url=http://192.168.1.10:3100

[all:vars]
env=lab
```

**1.7 — Verify Ansible connectivity**
```bash
ansible all -i inventory/hosts.ini -m ping
```

### Validation Gate
- [ ] `ansible all -m ping` returns `pong` for all 4 hosts
- [ ] `nc -zv logging-server 22` succeeds from each client
- [ ] `/etc/hostname` correct on all nodes
- [ ] Separate disk mounted at `/var/lib/loki` on logging server

### Known Pitfalls
- **SELinux/AppArmor** on clients may block Promtail reading log files — check `getenforce` or `/etc/apparmor.d/` and handle in Ansible
- **Cloud firewall vs OS firewall** — AWS Security Groups, GCP firewall rules, etc. block traffic before `ufw` sees it. Verify at both layers
- If you use DHCP, IPs will change. Use static IPs or DNS. Inventory with changing IPs will silently break Ansible

---

## Day 2 — Install & Configure Loki

### Goal
Loki running on logging server, accessible on port 3100, with persistent storage on the separate volume.

### Tasks

**2.1 — Install Loki (binary method — more control than package)**
```bash
LOKI_VERSION="2.9.8"   # Pin a version. Do NOT use "latest" in prod
cd /tmp
wget https://github.com/grafana/loki/releases/download/v${LOKI_VERSION}/loki-linux-amd64.zip
unzip loki-linux-amd64.zip
mv loki-linux-amd64 /usr/local/bin/loki
chmod +x /usr/local/bin/loki
```

**2.2 — Create directories and user**
```bash
useradd --system --no-create-home --shell /bin/false loki
mkdir -p /etc/loki /var/lib/loki/{index,chunks,wal}
chown -R loki:loki /var/lib/loki /etc/loki
```

**2.3 — Loki config (`/etc/loki/loki.yaml`)**

> ⚠️ Loki v3.x changed the `schema_config` period format and removed some legacy storage backends. If using v3, adjust `object_store: filesystem` and `store: tsdb`. This config targets v2.9.x.

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096
  log_level: warn   # NOT debug in production — debug fills disk fast

common:
  path_prefix: /var/lib/loki
  storage:
    filesystem:
      chunks_directory: /var/lib/loki/chunks
      rules_directory: /var/lib/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01     # Must be a past date, never future
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /var/lib/loki/index
    cache_location: /var/lib/loki/boltdb-cache
    shared_store: filesystem
  filesystem:
    directory: /var/lib/loki/chunks

compactor:
  working_directory: /var/lib/loki/compactor
  shared_store: filesystem
  retention_enabled: true

limits_config:
  retention_period: 30d       # Adjust per compliance requirements
  ingestion_rate_mb: 16
  ingestion_burst_size_mb: 32
  max_streams_per_user: 10000  # Increase if you hit "max streams" errors
  reject_old_samples: true
  reject_old_samples_max_age: 168h   # 7 days — reject logs older than this

chunk_store_config:
  max_look_back_period: 30d

table_manager:
  retention_deletes_enabled: true
  retention_period: 30d
```

**2.4 — Systemd unit (`/etc/systemd/system/loki.service`)**
```ini
[Unit]
Description=Loki Log Aggregation
After=network.target
Documentation=https://grafana.com/docs/loki/

[Service]
User=loki
Group=loki
ExecStart=/usr/local/bin/loki -config.file=/etc/loki/loki.yaml
Restart=on-failure
RestartSec=5s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now loki
```

**2.5 — Verify**
```bash
systemctl status loki
curl -s http://localhost:3100/ready   # Should return "ready"
curl -s http://localhost:3100/loki/api/v1/labels   # Should return empty or JSON
```

### Validation Gate
- [ ] `curl http://localhost:3100/ready` returns `ready`
- [ ] No errors in `journalctl -u loki -n 50`
- [ ] `/var/lib/loki` directory is on the separate volume (`df -h /var/lib/loki`)
- [ ] Loki process running as `loki` user, not root

### Known Pitfalls
- **`from` date in schema_config must be in the past.** If it's in the future, Loki starts but rejects all writes silently
- **Disk full = data loss.** Loki does not handle a full disk gracefully. Set up a cron or alert on disk usage before it hits 80%
- **Log level debug in production** will fill your disk overnight. Use `warn`
- **Port 3100 exposed to internet** = anyone can push logs to your Loki. In production, bind to internal interface or put behind nginx with auth

---

## Day 3 — Install Grafana & Connect to Loki

### Goal
Grafana accessible at port 3000, Loki datasource configured and queryable.

### Tasks

**3.1 — Install Grafana (OSS)**
```bash
apt-get install -y apt-transport-https software-properties-common
wget -q -O /usr/share/keyrings/grafana.key https://apt.grafana.com/gpg.key
echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com stable main" \
  > /etc/apt/sources.list.d/grafana.list
apt-get update
apt-get install -y grafana
```

**3.2 — Grafana config (`/etc/grafana/grafana.ini` — key settings)**
```ini
[server]
http_port = 3000
root_url = http://logging-server:3000   # Change to HTTPS URL when you add TLS

[security]
admin_password = CHANGE_THIS_NOW   # Do not leave as 'admin'
secret_key = a_long_random_string_here   # Used for cookie signing

[users]
allow_sign_up = false   # Never allow self-registration in production

[auth.anonymous]
enabled = false
```

```bash
systemctl enable --now grafana-server
```

**3.3 — Provision Loki datasource via YAML (preferred over clicking)**

Create `/etc/grafana/provisioning/datasources/loki.yaml`:
```yaml
apiVersion: 1
datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://localhost:3100
    isDefault: true
    editable: false   # Prevent accidental modification via UI
    jsonData:
      maxLines: 5000
      timeout: 60
```

```bash
systemctl restart grafana-server
```

**3.4 — Test the datasource**

In Grafana → Explore → Select Loki → Run query: `{job="varlogs"}`

It will return no results yet — that is correct. The important thing is no connection error.

### Validation Gate
- [ ] Grafana login page accessible at `:3000`
- [ ] Default password changed
- [ ] Loki datasource shows green "Data source connected" in Grafana
- [ ] No `permission denied` errors in `journalctl -u grafana-server`

### Known Pitfalls
- **Grafana connects to Loki via `localhost` not the external IP** — if you put `http://192.168.1.10:3100` as the datasource URL and Loki is on the same machine, both work, but `localhost` is simpler and avoids firewall confusion
- **Provisioned datasources cannot be deleted via UI** — that is intentional. If you need to change them, edit the YAML and restart
- **Grafana stores dashboards in SQLite by default** — fine for a lab, but SQLite is not suitable if you add a second Grafana instance later. Document this limitation

---

## Day 4 — Promtail Agent & System Log Shipping

### Goal
System logs (`/var/log/syslog`, `/var/log/auth.log`) from all clients flowing into Loki with correct labels.

### Tasks

**4.1 — Install Promtail on clients (Ansible role)**

Create `ansible/roles/logging-agent/tasks/main.yml`:
```yaml
- name: Create promtail user
  user:
    name: promtail
    system: yes
    shell: /bin/false
    groups: adm   # adm group can read /var/log on Ubuntu

- name: Download promtail binary
  get_url:
    url: "https://github.com/grafana/loki/releases/download/v{{ promtail_version }}/promtail-linux-amd64.zip"
    dest: /tmp/promtail.zip

- name: Extract promtail
  unarchive:
    src: /tmp/promtail.zip
    dest: /usr/local/bin/
    remote_src: yes
    creates: /usr/local/bin/promtail-linux-amd64

- name: Symlink promtail
  file:
    src: /usr/local/bin/promtail-linux-amd64
    dest: /usr/local/bin/promtail
    state: link

- name: Create config directory
  file:
    path: /etc/promtail
    state: directory
    owner: promtail
    group: promtail

- name: Deploy promtail config
  template:
    src: promtail.yaml.j2
    dest: /etc/promtail/promtail.yaml
    owner: promtail
    group: promtail
    mode: '0640'
  notify: restart promtail

- name: Deploy systemd unit
  template:
    src: promtail.service.j2
    dest: /etc/systemd/system/promtail.service
  notify:
    - reload systemd
    - restart promtail

- name: Enable promtail
  systemd:
    name: promtail
    enabled: yes
    state: started
```

**4.2 — Promtail config template (`ansible/roles/logging-agent/templates/promtail.yaml.j2`)**
```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0
  log_level: warn

positions:
  filename: /var/lib/promtail/positions.yaml   # Tracks read position per file

clients:
  - url: http://{{ loki_url }}/loki/api/v1/push
    backoff_config:
      min_period: 500ms
      max_period: 5m
      max_retries: 10
    timeout: 10s

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: system
          hostname: "{{ inventory_hostname }}"
          env: "{{ env }}"
          service: system
          log_type: system
          __path__: /var/log/syslog

  - job_name: auth
    static_configs:
      - targets:
          - localhost
        labels:
          job: auth
          hostname: "{{ inventory_hostname }}"
          env: "{{ env }}"
          service: ssh
          log_type: auth
          __path__: /var/log/auth.log
```

**4.3 — Position file directory**
```bash
mkdir -p /var/lib/promtail
chown promtail:promtail /var/lib/promtail
```

**4.4 — Deploy via Ansible**
```bash
ansible-playbook -i inventory/hosts.ini ansible/deploy-agents.yml
```

**4.5 — Verify in Grafana Explore**

Query: `{hostname="client01"} |= ""`

You should see log lines within 30 seconds of Promtail starting.

### Validation Gate
- [ ] `systemctl status promtail` shows `active (running)` on all clients
- [ ] Grafana Explore query `{hostname="client01"}` returns results
- [ ] All 3 client hostnames visible in label browser
- [ ] Timestamps in Grafana match server clock (NTP working)
- [ ] `journalctl -u promtail` shows no `connection refused` errors

### Known Pitfalls
- **Promtail must be in `adm` group** on Ubuntu to read `/var/log`. Without this, it starts successfully but reads nothing — no errors, just silence
- **Positions file** (`/var/lib/promtail/positions.yaml`) tracks where Promtail has read to. If this file is lost or corrupted, Promtail re-ships everything from the beginning, flooding Loki with duplicates
- **Loki rejects out-of-order logs.** If two log lines arrive with the same timestamp, Loki may reject one. If system time is wrong on a client, all its logs may be rejected with no clear error message
- **`__path__` is consumed** — it does not become a Loki label. Do not try to query by `__path__`
- **Log rotation** — Promtail handles `logrotate` gracefully IF the positions file is on the same filesystem. Do not put positions on a tmpfs

---

## Day 5 — Authentication & Security Logs

### Goal
SSH failed logins, successful logins, and sudo events reliably searchable in Grafana.

### Tasks

**5.1 — Add auth log scrape jobs to Promtail config**

Extend `promtail.yaml.j2`:
```yaml
  - job_name: sudo
    static_configs:
      - targets: [localhost]
        labels:
          job: sudo
          hostname: "{{ inventory_hostname }}"
          env: "{{ env }}"
          service: sudo
          log_type: auth
          __path__: /var/log/auth.log
    pipeline_stages:
      - match:
          selector: '{service="sudo"}'
          stages:
            - regex:
                expression: '.*COMMAND=(?P<command>.*)'
            - labels:
                command:
```

**5.2 — Ensure `auth.log` is populated**

On Ubuntu 22.04+, `auth.log` is populated by rsyslog. Verify:
```bash
grep "auth,authpriv" /etc/rsyslog.conf
# Should show: auth,authpriv.*   /var/log/auth.log
```

On systems using `journald` only (no rsyslog), you need to either:
- Install rsyslog: `apt-get install -y rsyslog`
- Or configure Promtail to read from journald directly (requires `UseJournal: true` and different Promtail config)

**5.3 — Useful LogQL queries to verify**

```logql
# Failed SSH logins
{log_type="auth"} |= "Failed password"

# Successful SSH logins  
{log_type="auth"} |= "Accepted publickey" or {log_type="auth"} |= "Accepted password"

# Sudo events
{service="sudo"} |= "COMMAND"

# Failed sudo (wrong password)
{log_type="auth"} |= "sudo" |= "authentication failure"

# All auth events from a specific host
{hostname="client01", log_type="auth"}

# SSH events in last 1 hour
{log_type="auth"} |= "sshd" [1h]
```

**5.4 — Generate test events**

```bash
# From a remote machine — wrong password (will log failed attempt)
ssh wronguser@client01

# On client01 — test sudo
sudo ls /root

# On client01 — check what was logged
grep "Failed\|Accepted\|sudo" /var/log/auth.log | tail -20
```

### Validation Gate
- [ ] Failed SSH login appears in Grafana within 60 seconds
- [ ] Sudo event appears in Grafana within 60 seconds
- [ ] Correct hostname label on each event
- [ ] Correct service label (`ssh` vs `sudo`)

### Known Pitfalls
- **Ubuntu 22.04 with `systemd-journald` but no rsyslog** — `auth.log` may not exist or may not be written to. Check `ls -la /var/log/auth.log`. If missing, install rsyslog
- **`sshd` config may suppress logs** — if `SyslogFacility` in `/etc/ssh/sshd_config` is not `AUTH`, SSH events go to a different log file
- **LogQL regex is RE2 syntax**, not PCRE. Some patterns that work in grep will not work in Loki pipeline stages

---

## Day 6 — Auditd Integration

### Goal
Kernel-level audit events (user creation, passwd changes, sensitive file access, privilege escalation) visible in Grafana.

### Tasks

**6.1 — Ansible role for auditd**

`ansible/roles/auditd/tasks/main.yml`:
```yaml
- name: Install auditd
  apt:
    name: [auditd, audispd-plugins]
    state: present

- name: Deploy audit rules
  template:
    src: audit.rules.j2
    dest: /etc/audit/rules.d/logging-platform.rules
    mode: '0640'
  notify: restart auditd

- name: Configure auditd to write to syslog
  lineinfile:
    path: /etc/audit/plugins.d/syslog.conf
    regexp: '^active'
    line: 'active = yes'
  notify: restart auditd
```

**6.2 — Audit rules template (`audit.rules.j2`)**
```bash
# Delete existing rules first
-D

# Increase buffer — prevents dropped events under load
-b 8192

# Failure mode: 1=silent, 2=panic. Use 1 in production
-f 1

# Monitor passwd and shadow changes
-w /etc/passwd -p wa -k user_modification
-w /etc/shadow -p wa -k user_modification
-w /etc/group -p wa -k group_modification
-w /etc/gshadow -p wa -k group_modification
-w /etc/sudoers -p wa -k sudoers_modification
-w /etc/sudoers.d/ -p wa -k sudoers_modification

# Monitor user/group management commands
-w /usr/sbin/useradd -p x -k user_mgmt
-w /usr/sbin/userdel -p x -k user_mgmt
-w /usr/sbin/usermod -p x -k user_mgmt
-w /usr/sbin/groupadd -p x -k user_mgmt
-w /usr/sbin/groupdel -p x -k user_mgmt
-w /usr/sbin/passwd -p x -k user_mgmt

# Monitor systemd service changes
-w /etc/systemd/system/ -p wa -k service_modification
-w /usr/lib/systemd/system/ -p wa -k service_modification

# Monitor SSH config
-w /etc/ssh/sshd_config -p wa -k ssh_config

# Monitor cron
-w /etc/cron.d/ -p wa -k cron_modification
-w /var/spool/cron/ -p wa -k cron_modification

# Privileged commands (setuid binaries)
-a always,exit -F path=/usr/bin/sudo -F perm=x -F auid>=1000 -F auid!=-1 -k priv_cmd
-a always,exit -F path=/usr/bin/su -F perm=x -F auid>=1000 -F auid!=-1 -k priv_cmd

# Make rules immutable — requires reboot to change (comment out during testing)
# -e 2
```

**6.3 — Add auditd log scrape to Promtail**

```yaml
  - job_name: audit
    static_configs:
      - targets: [localhost]
        labels:
          job: audit
          hostname: "{{ inventory_hostname }}"
          env: "{{ env }}"
          service: audit
          log_type: audit
          __path__: /var/log/audit/audit.log
    pipeline_stages:
      - match:
          selector: '{log_type="audit"}'
          stages:
            - regex:
                expression: '.*key="(?P<audit_key>[^"]+)"'
            - labels:
                audit_key:
```

> ⚠️ The `promtail` user needs to be in the `adm` group AND `/var/log/audit/audit.log` needs to be readable. By default, audit.log is mode 600, root:root. You must either add promtail to the `audit` group (which requires creating it) or use `auditd` → `syslog` plugin and read from `/var/log/syslog` instead. The syslog approach (6.1) is simpler.

### Validation Gate
- [ ] `auditctl -l` shows rules loaded on clients
- [ ] `useradd testuser` on client generates event visible in Grafana within 60s
- [ ] Audit events have `audit_key` label extracted correctly
- [ ] `audit_key="user_modification"` returns results for passwd changes

### Known Pitfalls
- **`-e 2` (immutable rules)** will prevent any rule changes until reboot. Do NOT enable this during initial setup
- **Auditd log rotation** — auditd has its own log rotation separate from logrotate. Configure `max_log_file_action = ROTATE` and `num_logs = 5` in `/etc/audit/auditd.conf`
- **High audit volume** under load — the `-a always,exit` rules for syscalls generate enormous volume. Only add syscall rules for specific security requirements, not wholesale

---

## Day 7 — Application Logs (PostgreSQL + Nginx)

### Goal
PostgreSQL query logs and Nginx access/error logs visible in Grafana with separate filtering.

### Tasks

**7.1 — PostgreSQL logging config**

On client01 (PostgreSQL server), edit `/etc/postgresql/*/main/postgresql.conf`:
```
log_destination = 'stderr'
logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d.log'
log_rotation_age = 1d
log_rotation_size = 100MB
log_min_duration_statement = 1000   # Log queries taking > 1 second
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_connections = on
log_disconnections = on
log_checkpoints = on
log_lock_waits = on
```

```bash
systemctl reload postgresql
```

**7.2 — Promtail scrape for PostgreSQL**
```yaml
  - job_name: postgresql
    static_configs:
      - targets: [localhost]
        labels:
          job: postgresql
          hostname: "{{ inventory_hostname }}"
          env: "{{ env }}"
          service: postgresql
          log_type: db
          __path__: /var/log/postgresql/postgresql-*.log
```

> ⚠️ Promtail `__path__` supports globs. The `*` in `postgresql-*.log` will match dated files. However, when a new day's file is created, Promtail detects it automatically — verify this works when you cross midnight.

**7.3 — Nginx log format and scrape**

On client02 (Nginx server), edit `/etc/nginx/nginx.conf`:
```nginx
log_format json_combined escape=json '{'
  '"time":"$time_iso8601",'
  '"remote_addr":"$remote_addr",'
  '"status":$status,'
  '"request":"$request",'
  '"body_bytes_sent":$body_bytes_sent,'
  '"request_time":$request_time,'
  '"http_referer":"$http_referer",'
  '"http_user_agent":"$http_user_agent"'
'}';

access_log /var/log/nginx/access.log json_combined;
error_log /var/log/nginx/error.log warn;
```

> Using JSON format makes Promtail parsing trivial and removes ambiguity in field extraction. If you cannot change the log format (existing app), you must write a regex pipeline stage — plan for 2–4 hours of debugging.

```yaml
  - job_name: nginx-access
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx
          hostname: "{{ inventory_hostname }}"
          env: "{{ env }}"
          service: nginx
          log_type: web
          __path__: /var/log/nginx/access.log
    pipeline_stages:
      - json:
          expressions:
            status: status
            remote_addr: remote_addr
      - labels:
          status:

  - job_name: nginx-error
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx-error
          hostname: "{{ inventory_hostname }}"
          env: "{{ env }}"
          service: nginx
          log_type: web
          __path__: /var/log/nginx/error.log
```

### Validation Gate
- [ ] PostgreSQL log lines visible in Grafana with `{service="postgresql"}`
- [ ] Nginx access log visible with `{service="nginx"}`
- [ ] HTTP 4xx/5xx errors filterable: `{service="nginx"} | json | status >= 400`
- [ ] Slow query logs (> 1s) visible in PostgreSQL logs

### Known Pitfalls
- **File permissions** — `/var/log/postgresql/` is typically mode 750, owned by `postgres:postgres`. Promtail user cannot read it. Fix: `chmod 755 /var/log/postgresql` or add promtail to the postgres group
- **Nginx log buffer** — by default Nginx buffers access log writes. Under low traffic, lines may not appear in Grafana for 30–60 seconds. Add `access_log /var/log/nginx/access.log json_combined flush=1s;` to reduce latency
- **JSON pipeline stage in Promtail** — extracted fields are NOT labels unless explicitly promoted with a `labels:` stage. High-cardinality fields (`remote_addr`, `request_uri`) should never become labels

---

## Day 8 — Dashboards

### Goal
Four operational dashboards built and saved. Queries work on real data.

### Dashboard Strategy

Provision dashboards as JSON files via Grafana provisioning — do not build them manually in the UI. Manual dashboards are lost if Grafana is reinstalled.

Create `/etc/grafana/provisioning/dashboards/default.yaml`:
```yaml
apiVersion: 1
providers:
  - name: default
    folder: Logging Platform
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

Place dashboard JSON files in `/var/lib/grafana/dashboards/`.

### Dashboard 1 — SSH & Authentication

**Panels:**
1. **Stat panel** — Failed SSH logins (last 24h)
   - Query: `count_over_time({log_type="auth"} |= "Failed password" [24h])`
2. **Time series** — Failed logins over time (1h buckets)
   - Query: `sum(count_over_time({log_type="auth"} |= "Failed password" [$__interval]))`
3. **Logs panel** — Raw failed login events
   - Query: `{log_type="auth"} |= "Failed password"`
4. **Table** — Failed logins by hostname
   - Use `sum by (hostname)` with count_over_time
5. **Logs panel** — Sudo events
   - Query: `{service="sudo"} |= "COMMAND"`

### Dashboard 2 — Audit Events

**Panels:**
1. **Stat** — Audit events (last 24h) by key
2. **Logs panel** — User modification events: `{log_type="audit"} |= "user_modification"`
3. **Logs panel** — Sudoers changes: `{log_type="audit"} |= "sudoers_modification"`
4. **Logs panel** — All audit events with `audit_key` filter variable

### Dashboard 3 — PostgreSQL

**Panels:**
1. **Stat** — Connection events
2. **Logs panel** — Authentication failures: `{service="postgresql"} |= "authentication failed"`
3. **Logs panel** — Slow queries: `{service="postgresql"} |= "duration"`
4. **Logs panel** — Errors: `{service="postgresql"} |= "ERROR"`

### Dashboard 4 — Web (Nginx)

**Panels:**
1. **Time series** — Request rate: `count_over_time({service="nginx", log_type="web"}[$__interval])`
2. **Stat** — 4xx errors: `count_over_time({service="nginx"} | json | status >= 400 and status < 500 [1h])`
3. **Stat** — 5xx errors: `count_over_time({service="nginx"} | json | status >= 500 [1h])`
4. **Logs panel** — Error log: `{service="nginx", job="nginx-error"}`
5. **Logs panel** — 4xx/5xx access log entries

### Dashboard Variables (add to all dashboards)

| Variable | Type | Query |
|---|---|---|
| `hostname` | Query | `label_values(hostname)` |
| `env` | Query | `label_values(env)` |
| `service` | Query | `label_values(service)` |
| `time_range` | Interval | 5m, 15m, 1h, 6h, 24h |

Use `$hostname` in all LogQL queries to make dashboards filterable.

### Validation Gate
- [ ] All 4 dashboards load without errors
- [ ] Hostname dropdown filters data correctly
- [ ] Stat panels show non-zero counts after generating test events
- [ ] Logs panels show raw lines with correct timestamps

---

## Day 9 — Ansible Automation

### Goal
A single `ansible-playbook` command deploys everything on a fresh node.

### Ansible Role Structure

```
ansible/
├── inventory/
│   └── hosts.ini
├── group_vars/
│   ├── all.yml           # Global vars (loki_url, promtail_version, env)
│   └── clients.yml       # Client-specific vars
├── roles/
│   ├── common/           # Base packages, hostname, /etc/hosts
│   ├── chrony/           # NTP configuration
│   ├── auditd/           # Auditd + rules (Day 6)
│   ├── logging-agent/    # Promtail install + config (Day 4)
│   └── loki/             # Loki + Grafana (logging server only)
├── deploy-logging-server.yml
├── deploy-clients.yml
└── site.yml              # Runs everything
```

**`group_vars/all.yml`:**
```yaml
promtail_version: "2.9.8"
loki_version: "2.9.8"
loki_url: "192.168.1.10:3100"
env: "lab"
ntp_servers:
  - 0.pool.ntp.org
  - 1.pool.ntp.org
```

**`site.yml`:**
```yaml
---
- import_playbook: deploy-logging-server.yml
- import_playbook: deploy-clients.yml
```

**`deploy-clients.yml`:**
```yaml
---
- name: Deploy logging agents to clients
  hosts: clients
  become: yes
  roles:
    - common
    - chrony
    - auditd
    - logging-agent
```

**Idempotency requirement:** Every task must be safe to re-run. Use Ansible modules (not `shell: echo x >> file`). Test by running the playbook twice — the second run should show zero changes except for `ok`.

**Tags for selective deployment:**
```yaml
- name: Deploy promtail config
  template:
    src: promtail.yaml.j2
    dest: /etc/promtail/promtail.yaml
  tags: [config, promtail]
  notify: restart promtail
```

Run with: `ansible-playbook site.yml --tags config` to push config changes only.

**Onboarding a new server:**
1. Add to `inventory/hosts.ini` under `[clients]`
2. Set `ansible_host` IP
3. Run: `ansible-playbook deploy-clients.yml -l new-server-hostname`

That is the "one command" onboarding goal achieved.

### Validation Gate
- [ ] `ansible-playbook site.yml` completes with zero failures
- [ ] Running it a second time shows no changes (idempotency)
- [ ] A fresh client added to inventory and deployed shows logs in Grafana within 5 minutes
- [ ] All roles have `tasks/main.yml`, `handlers/main.yml`, and `templates/` where needed

### Known Pitfalls
- **Handler ordering** — Ansible handlers run at the end of a play, not immediately. If a config change and a service restart must happen in order within the same play, use `meta: flush_handlers`
- **`notify` on changed only** — handlers only run when the notifying task reports `changed`. If config is already correct, handler will not run. This is correct behavior but confuses people during debugging
- **Template variables not defined** — if a variable in a Jinja2 template is missing from inventory, Ansible will fail with `undefined variable`. Use `{{ var | default('fallback') }}` for optional vars
- **Become password prompts** — if `NOPASSWD` sudo is not configured on a new node, playbook will hang waiting for a password interactively

---

## Day 10 — Validation & Hardening

### Goal
Generate all event types, verify they appear correctly, and close the obvious security gaps.

### 10.1 — Event Generation Matrix

Run these on the appropriate client and verify in Grafana before moving on:

| Event | Command | Expected in Grafana |
|---|---|---|
| Failed SSH login | `ssh baduser@client01` (from remote) | `{log_type="auth"} \|= "Failed password"` |
| Successful SSH login | `ssh ansible@client01` | `{log_type="auth"} \|= "Accepted"` |
| Sudo execution | `sudo ls /root` on client01 | `{service="sudo"} \|= "COMMAND"` |
| User creation | `sudo useradd audittest` on client01 | `{log_type="audit"} \|= "user_modification"` |
| passwd change | `sudo passwd audittest` | `{log_type="audit"} \|= "user_modification"` |
| PostgreSQL auth failure | `psql -U nonexistent -d postgres` | `{service="postgresql"} \|= "authentication failed"` |
| Nginx 404 | `curl http://client02/doesnotexist` | `{service="nginx"} \| json \| status == 404` |
| Nginx 500 | Depends on app config | `{service="nginx"} \| json \| status >= 500` |
| Sensitive file change | `sudo touch /etc/sudoers.d/test` | `{log_type="audit"} \|= "sudoers_modification"` |

### 10.2 — Hardening Checklist

These are minimum steps before calling anything production-adjacent:

**Loki:**
- [ ] Loki binds only to internal interface, not `0.0.0.0` in public environments
- [ ] Disk usage alert or cron job monitoring `/var/lib/loki`
- [ ] Retention policy configured and tested (`retention_period: 30d`)

**Grafana:**
- [ ] Default `admin` password changed
- [ ] `allow_sign_up = false`
- [ ] If accessible from internet: put behind nginx with HTTPS
- [ ] Viewer role accounts for read-only access (do not share admin credentials)

**Promtail:**
- [ ] Runs as non-root `promtail` user
- [ ] Config file mode `0640` (no world-readable — config contains Loki URL)
- [ ] Positions file on persistent volume (not tmpfs)

**Auditd:**
- [ ] `auditctl -l` shows rules on all clients
- [ ] Log rotation configured in `/etc/audit/auditd.conf`
- [ ] `-e 2` (immutable) NOT set yet — only enable after rules are finalized

**Ansible:**
- [ ] No plaintext passwords in inventory or vars — use Ansible Vault for any secrets
- [ ] `ansible` user has minimum required sudo permissions (or NOPASSWD limited to specific commands)

### 10.3 — Performance Baselines

Record these now to detect problems later:

```bash
# Loki disk usage
du -sh /var/lib/loki/

# Loki ingestion rate (from Grafana Explore)
# Query: rate({job=~".+"} [5m])  — shows log lines per second

# Promtail memory per client
ps -o pid,rss,comm -p $(pgrep promtail)

# Number of active streams in Loki
curl -s http://localhost:3100/loki/api/v1/label | python3 -m json.tool
```

### 10.4 — Known Gaps to Document for Phase 2

These are real gaps that exist after 10 days. Write them down:

- No alerting — repeated SSH failures generate no notifications
- No TLS — all log traffic is plaintext HTTP between clients and Loki
- No authentication on Loki push endpoint — any internal host can push fake logs
- Grafana on HTTP, not HTTPS
- SQLite backend on Grafana — not suitable for HA
- No log integrity — logs can be tampered with after ingestion (Loki is not an immutable store)
- No backup of Loki data or Grafana dashboards

---

## Ansible Role Dependency Summary

| Role | Depends On | Applied To |
|---|---|---|
| `common` | none | all nodes |
| `chrony` | common | all nodes |
| `auditd` | common | clients |
| `logging-agent` | common, chrony | clients |
| `loki` | common | logging-server |
| `grafana` | loki | logging-server |

---

## File Reference Map

```
ansible/
├── inventory/hosts.ini
├── group_vars/all.yml
├── site.yml
├── deploy-logging-server.yml
├── deploy-clients.yml
└── roles/
    ├── common/tasks/main.yml
    ├── chrony/tasks/main.yml
    ├── chrony/templates/chrony.conf.j2
    ├── auditd/tasks/main.yml
    ├── auditd/handlers/main.yml
    ├── auditd/templates/audit.rules.j2
    ├── logging-agent/tasks/main.yml
    ├── logging-agent/handlers/main.yml
    └── logging-agent/templates/
        ├── promtail.yaml.j2
        └── promtail.service.j2
```

---

## Critical LogQL Reference

```logql
# All logs from a host
{hostname="client01"}

# Failed SSH (last 1h)
{log_type="auth"} |= "Failed password" [1h]

# Sudo events
{service="sudo"} |= "COMMAND"

# Audit events by key
{log_type="audit", audit_key="user_modification"}

# PostgreSQL errors
{service="postgresql"} |= "ERROR"

# Nginx 5xx errors
{service="nginx"} | json | status >= 500

# Any log line matching a pattern across all services
{env="lab"} |= "error" | __error__ = ""

# Count events (for stat panels)
count_over_time({log_type="auth"} |= "Failed password" [24h])

# Rate of events (for time series)
sum(rate({log_type="auth"} |= "Failed password" [$__interval]))
```

---

## Resume Entry (Revised)

Designed and deployed a centralized logging and security audit platform for a multi-server Linux environment using Grafana, Loki, Promtail, and auditd. Implemented structured log collection for SSH authentication events, sudo activity, kernel audit events, PostgreSQL query logs, and Nginx access logs across three client nodes. Configured Promtail pipeline stages for label extraction and field parsing. Built operational dashboards in Grafana for authentication monitoring, audit event review, database error tracking, and web traffic analysis. Automated full-stack deployment and new-server onboarding using Ansible roles, achieving idempotent configuration management from a single playbook. Established retention policies, basic security hardening, and documented production limitations for future remediation.
