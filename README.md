# Centralized Logging & Audit Platform — Final Build Plan

> **Philosophy:** Do it manually first. Verify it works. Then automate only what needs to scale.
> Grafana and Loki are installed once on one server — no automation needed.
> Promtail and auditd go on potentially hundreds of nodes — manual first on one node, then Ansible.
> No passwordless SSH. No passwordless sudo. No extra system users created just for tooling.

---

## What Gets Automated vs What Does Not

| Component | How |
|---|---|
| Loki | Manual install on logging server — one time |
| Grafana | Manual install on logging server — one time |
| Promtail | Manual on first client to verify, then Ansible for all nodes |
| Auditd | Manual on first client to verify, then Ansible for all nodes |
| Firewall rules | Not touched — configure separately, do not let this plan break existing rules |

---

## Node Layout

| Hostname | IP | Role |
|---|---|---|
| `logging-server` | `192.168.1.10` | Loki + Grafana |
| `client01` | `192.168.1.11` | First node — manual verification |
| `client02` | `192.168.1.12` | Ansible deployed |
| `client03` | `192.168.1.13` | Ansible deployed |

Replace these IPs with your actual addresses throughout.

---

## Architecture

```
┌──────────────────────────────────────┐
│  LOGGING SERVER (192.168.1.10)       │
│  Loki    :3100                       │
│  Grafana :3000                       │
└──────────────────────────────────────┘
        ▲            ▲            ▲
   Promtail      Promtail     Promtail
   client01      client02     client03
```

---

## Label Strategy

Define once. Never deviate. Inconsistent labels break every query.

| Label | Values | Set By |
|---|---|---|
| `hostname` | `client01`, `client02`, etc. | Ansible inventory var |
| `env` | `lab`, `staging`, `prod` | Ansible group var |
| `service` | `ssh`, `sudo`, `postgresql`, `nginx`, `audit`, `system` | Per scrape job |
| `log_type` | `auth`, `audit`, `web`, `db`, `system` | Per scrape job |

Never make these into labels: IP addresses, usernames, request IDs, URLs. Those are high-cardinality — they go in the log line and are filtered with `|=`.

---

## Pre-Start Checklist

- [ ] OS version confirmed — Ubuntu 22.04 LTS on all nodes
- [ ] Static IPs on all nodes — DHCP will silently break things
- [ ] `/etc/hosts` or DNS resolving all hostnames
- [ ] NTP synchronized on all nodes: `timedatectl` shows `NTP synchronized: yes`
- [ ] Loki version decided: **2.9.8** — v2.x and v3.x have breaking config differences
- [ ] Separate disk or partition available for Loki data (minimum 20GB)
- [ ] You can SSH into all nodes with your normal user
- [ ] Your user has sudo access (with password) on all nodes

---

## Part 1 — Logging Server Setup (Manual Only)

### Step 1 — Prepare the Loki Data Volume

```bash
# List block devices to identify the separate disk
lsblk

# Format and mount — adjust /dev/sdb to your actual device
mkfs.ext4 /dev/sdb
mkdir -p /var/lib/loki

# Get UUID for stable mounting
blkid /dev/sdb
# Copy the UUID from output

# Add to fstab
echo "UUID=your-uuid-here  /var/lib/loki  ext4  defaults  0  2" >> /etc/fstab
mount -a

# Verify — must show a separate filesystem, not root
df -h /var/lib/loki
```

If `df -h /var/lib/loki` shows the root filesystem, Loki data will eventually fill your OS disk and crash everything. Fix this before continuing.

---

### Step 2 — Install Loki

```bash
cd /tmp
LOKI_VERSION="2.9.8"

wget https://github.com/grafana/loki/releases/download/v${LOKI_VERSION}/loki-linux-amd64.zip
unzip loki-linux-amd64.zip

# Verify binary before installing
./loki-linux-amd64 --version

mv loki-linux-amd64 /usr/local/bin/loki
chmod +x /usr/local/bin/loki
loki --version
```

---

### Step 3 — Configure Loki

```bash
sudo mkdir -p /etc/loki
sudo mkdir -p /var/lib/loki/{index,chunks,wal,compactor,boltdb-cache,rules}

sudo tee /etc/loki/loki.yaml > /dev/null << 'EOF'
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096
  log_level: warn

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
    - from: 2024-01-01
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
  retention_period: 30d
  ingestion_rate_mb: 16
  ingestion_burst_size_mb: 32
  max_streams_per_user: 10000
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  max_query_lookback: 30d

table_manager:
  retention_deletes_enabled: true
  retention_period: 30d
EOF
```

Verify config is valid before creating the service:
```bash
loki -config.file=/etc/loki/loki.yaml -verify-config
# Must return: config is valid
```

> `from: 2024-01-01` must be a past date. A future date causes Loki to silently reject all writes with no useful error.

---

### Step 4 — Loki as a Systemd Service

```bash
sudo tee /etc/systemd/system/loki.service > /dev/null << 'EOF'
[Unit]
Description=Loki Log Aggregation
After=network.target

[Service]
ExecStart=/usr/local/bin/loki -config.file=/etc/loki/loki.yaml
Restart=on-failure
RestartSec=5s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable loki
sudo systemctl start loki
```

**Verify:**
```bash
sudo systemctl status loki

# Must return "ready"
curl -s http://localhost:3100/ready

# Must return valid JSON
curl -s http://localhost:3100/loki/api/v1/labels
```

**Check logs for errors:**
```bash
sudo journalctl -u loki -n 50 --no-pager
# Look for: level=error — there should be none at startup
```

---

### Step 5 — Install Grafana

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget

sudo mkdir -p /usr/share/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | sudo tee /usr/share/keyrings/grafana.key > /dev/null

echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com stable main" \
  | sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install -y grafana

grafana server -v
```

---

### Step 6 — Configure Grafana

Edit `/etc/grafana/grafana.ini`. Find and change these specific lines — do not replace the whole file:

```bash
sudo nano /etc/grafana/grafana.ini
```

Under `[security]`:
```ini
admin_password = something_strong_here
```

Under `[users]`:
```ini
allow_sign_up = false
```

Under `[auth.anonymous]`:
```ini
enabled = false
```

**Provision the Loki datasource:**
```bash
sudo mkdir -p /etc/grafana/provisioning/datasources

sudo tee /etc/grafana/provisioning/datasources/loki.yaml > /dev/null << 'EOF'
apiVersion: 1
datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://localhost:3100
    isDefault: true
    editable: false
    jsonData:
      maxLines: 5000
      timeout: 60
EOF

sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

**Verify:**
```bash
sudo systemctl status grafana-server

# Should return HTTP 200
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/login
```

Open `http://192.168.1.10:3000` in a browser. Log in with `admin` and your password.

Go to **Connections → Data sources → Loki → Test**. Must show green — connected.

Go to **Explore**, select Loki, run `{job="test"}` — no results is correct. No connection error is what you are verifying.

---

### Logging Server Validation

- [ ] `curl http://localhost:3100/ready` returns `ready`
- [ ] `systemctl status loki` — active, running
- [ ] `journalctl -u loki -n 50` — no ERROR lines
- [ ] `df -h /var/lib/loki` — separate volume, not root filesystem
- [ ] Grafana login works with the new password (not `admin`/`admin`)
- [ ] Loki datasource shows green in Grafana

---

## Part 2 — Promtail Agent (Manual on client01 First, Then Ansible)

### Step 7 — Manual Promtail Install on client01

SSH into client01 and run everything in this section as your normal user with sudo.

**Verify the log files you want to ship actually exist:**
```bash
ls -la /var/log/syslog
ls -la /var/log/auth.log
# Both must exist and have recent timestamps

# Verify your user can read them (it should via sudo)
sudo tail -5 /var/log/syslog
sudo tail -5 /var/log/auth.log
```

**Install Promtail:**
```bash
cd /tmp
sudo useradd --system --no-create-home --shell /bin/false -G adm promtail

PROMTAIL_VERSION="2.9.8"

wget https://github.com/grafana/loki/releases/download/v${PROMTAIL_VERSION}/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip

./promtail-linux-amd64 --version

sudo mv promtail-linux-amd64 /usr/local/bin/promtail
sudo chmod +x /usr/local/bin/promtail
promtail --version
```

**Create config directory and positions directory:**
```bash
sudo mkdir -p /etc/promtail /var/lib/promtail
sudo chown -R promtail:promtail /var/lib/promtail

```

**Write Promtail config:**
```bash
sudo tee /etc/promtail/promtail.yaml > /dev/null << 'EOF'
server:
  http_listen_port: 9080
  grpc_listen_port: 0
  log_level: warn

positions:
  filename: /var/lib/promtail/positions.yaml

clients:
  - url: http://192.168.1.10:3100/loki/api/v1/push
    backoff_config:
      min_period: 500ms
      max_period: 5m
      max_retries: 10
    timeout: 10s

scrape_configs:
  - job_name: system
    static_configs:
      - targets: [localhost]
        labels:
          job: system
          hostname: client01
          env: lab
          service: system
          log_type: system
          __path__: /var/log/syslog

  - job_name: auth
    static_configs:
      - targets: [localhost]
        labels:
          job: auth
          hostname: client01
          env: lab
          service: ssh
          log_type: auth
          __path__: /var/log/auth.log
EOF
```

**Verify config syntax:**
```bash
promtail -config.file=/etc/promtail/promtail.yaml -dry-run
# Must exit with no errors
```

**Verify connectivity to Loki before starting the service:**
```bash
curl -s http://192.168.1.10:3100/ready
# Must return: ready
# If this fails, fix connectivity before continuing — the service will just silently fail
```

**Create and start the service:**
```bash
sudo tee /etc/systemd/system/promtail.service > /dev/null << 'EOF'
[Unit]
Description=Promtail Log Shipping Agent
After=network.target

[Service]
User=promtail
Group=promtail
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/promtail.yaml
Restart=on-failure
RestartSec=5s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable promtail
sudo systemctl start promtail
```

**Verify Promtail is running and shipping:**
```bash
sudo systemctl status promtail

# Watch for errors — specifically "connection refused" or "permission denied"
sudo journalctl -u promtail -n 50 --no-pager

# Positions file must exist and have entries — confirms Promtail is reading files
sudo cat /var/lib/promtail/positions.yaml
# Must show entries for /var/log/syslog and /var/log/auth.log with non-zero offsets

# Promtail metrics endpoint — if this returns data, it is alive
curl -s http://localhost:9080/metrics | grep promtail_sent_bytes_total
```

**Verify in Grafana Explore:**
```logql
{hostname="client01"}
```

Results must appear within 30 seconds. If nothing after 60 seconds:
1. `sudo journalctl -u promtail -f` — look for push errors
2. `sudo journalctl -u loki -f` on logging server — look for rejection errors
3. Re-run `curl -s http://192.168.1.10:3100/ready` from client01

---

### Step 8 — Generate Test Events and Verify Capture

Do this on client01 before writing any Ansible automation. If these do not appear in Grafana, something is wrong and automating it will deploy broken agents everywhere.

```bash
# Generate a sudo event
sudo ls /root

# Generate a failed SSH — run this from your workstation
ssh nonexistent@192.168.1.11

# Check the file directly first
grep "COMMAND\|Failed password\|Accepted" /var/log/auth.log | tail -10
```

Then in Grafana Explore:
```logql
{log_type="auth", hostname="client01"} |= "COMMAND"
{log_type="auth"} |= "Failed password"
```

Both must return results within 60 seconds. Only proceed to Ansible after this passes.

---

### Step 9 — Ansible Automation for Promtail (100s of Nodes)

Now that you know the manual steps work, encode them exactly into Ansible. Nothing in these roles should be a surprise — it is the same commands you just ran.

**Ansible inventory structure:**
```
ansible/
├── inventory/
│   └── hosts.ini
├── group_vars/
│   └── all/
│       ├── vars.yml
│       └── vault.yml        ← encrypted with ansible-vault
├── roles/
│   └── logging-agent/
│       ├── tasks/main.yml
│       ├── handlers/main.yml
│       └── templates/
│           ├── promtail.yaml.j2
│           └── promtail.service.j2
└── deploy-agents.yml
```

**`inventory/hosts.ini`:**
```ini
[clients]
client01 ansible_host=192.168.1.11
client02 ansible_host=192.168.1.12
client03 ansible_host=192.168.1.13

[clients:vars]
loki_url=192.168.1.10:3100
env=lab
```

**`group_vars/all/vars.yml`:**
```yaml
promtail_version: "2.9.8"
ansible_user: your_normal_username
ansible_become: true
```

**`group_vars/all/vault.yml` — create then encrypt:**
```yaml
vault_ssh_pass: "your_ssh_password"
vault_become_pass: "your_sudo_password"
```

```bash
ansible-vault encrypt group_vars/all/vault.yml
```

**Reference vault vars in `vars.yml`:**
```yaml
ansible_ssh_pass: "{{ vault_ssh_pass }}"
ansible_become_pass: "{{ vault_become_pass }}"
```

**`roles/logging-agent/tasks/main.yml`:**
```yaml
---
- name: Create promtail config and positions directories
  file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - /etc/promtail
    - /var/lib/promtail

- name: Download promtail archive
  get_url:
    url: "https://github.com/grafana/loki/releases/download/v{{ promtail_version }}/promtail-linux-amd64.zip"
    dest: /tmp/promtail-{{ promtail_version }}.zip
    mode: '0644'

- name: Unarchive promtail
  unarchive:
    src: /tmp/promtail-{{ promtail_version }}.zip
    dest: /tmp/
    remote_src: yes

- name: Install promtail binary
  copy:
    src: /tmp/promtail-linux-amd64
    dest: /usr/local/bin/promtail
    mode: '0755'
    remote_src: yes
  notify: restart promtail

- name: Deploy promtail config
  template:
    src: promtail.yaml.j2
    dest: /etc/promtail/promtail.yaml
    mode: '0640'
  notify: restart promtail

- name: Deploy promtail systemd unit
  template:
    src: promtail.service.j2
    dest: /etc/systemd/system/promtail.service
    mode: '0644'
  notify:
    - reload systemd
    - restart promtail

- name: Enable and start promtail
  systemd:
    name: promtail
    enabled: yes
    state: started
    daemon_reload: yes
```

**`roles/logging-agent/handlers/main.yml`:**
```yaml
---
- name: reload systemd
  systemd:
    daemon_reload: yes

- name: restart promtail
  systemd:
    name: promtail
    state: restarted
```

**`roles/logging-agent/templates/promtail.yaml.j2`:**
```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0
  log_level: warn

positions:
  filename: /var/lib/promtail/positions.yaml

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
      - targets: [localhost]
        labels:
          job: system
          hostname: "{{ inventory_hostname }}"
          env: "{{ env }}"
          service: system
          log_type: system
          __path__: /var/log/syslog

  - job_name: auth
    static_configs:
      - targets: [localhost]
        labels:
          job: auth
          hostname: "{{ inventory_hostname }}"
          env: "{{ env }}"
          service: ssh
          log_type: auth
          __path__: /var/log/auth.log
```

**`roles/logging-agent/templates/promtail.service.j2`:**
```ini
[Unit]
Description=Promtail Log Shipping Agent
After=network.target

[Service]
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/promtail.yaml
Restart=on-failure
RestartSec=5s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

**`deploy-agents.yml`:**
```yaml
---
- name: Deploy Promtail to all clients
  hosts: clients
  become: yes
  roles:
    - logging-agent
```

**Run:**
```bash
ansible-playbook -i inventory/hosts.ini deploy-agents.yml --ask-vault-pass
```

**Idempotency test — must pass before you deploy to 100s of nodes:**
```bash
# First run — expect changes
ansible-playbook -i inventory/hosts.ini deploy-agents.yml --ask-vault-pass

# Second run immediately after — must show changed=0
ansible-playbook -i inventory/hosts.ini deploy-agents.yml --ask-vault-pass
# Every host: ok=N  changed=0  failed=0
```

**Adding a new node:**
```bash
# 1. Add to inventory/hosts.ini under [clients]
# 2. Run targeting only the new host
ansible-playbook -i inventory/hosts.ini deploy-agents.yml --ask-vault-pass -l new-hostname

# 3. Verify in Grafana — {hostname="new-hostname"} should return logs within 5 minutes
```

---

## Part 3 — Auditd (Manual on client01 First, Then Ansible)

### Step 10 — Manual Auditd Setup on client01

**Install auditd:**
```bash
sudo apt-get install -y auditd audispd-plugins

sudo systemctl enable auditd
sudo systemctl start auditd
sudo systemctl status auditd

# Verify audit log exists
ls -la /var/log/audit/audit.log
```

**Write audit rules:**
```bash
sudo tee /etc/audit/rules.d/logging-platform.rules > /dev/null << 'EOF'
# Clear existing rules
-D

# Buffer size — prevents dropped events under load
-b 8192

# Failure mode: 1 = silent, 2 = panic. Use 1 in production.
-f 1

# User and group file changes
-w /etc/passwd -p wa -k user_modification
-w /etc/shadow -p wa -k user_modification
-w /etc/group -p wa -k group_modification
-w /etc/gshadow -p wa -k group_modification
-w /etc/sudoers -p wa -k sudoers_modification
-w /etc/sudoers.d/ -p wa -k sudoers_modification

# User management command execution
-w /usr/sbin/useradd -p x -k user_mgmt
-w /usr/sbin/userdel -p x -k user_mgmt
-w /usr/sbin/usermod -p x -k user_mgmt
-w /usr/sbin/groupadd -p x -k user_mgmt
-w /usr/sbin/groupdel -p x -k user_mgmt
-w /usr/sbin/passwd -p x -k user_mgmt

# SSH config changes
-w /etc/ssh/sshd_config -p wa -k ssh_config

# Systemd service changes
-w /etc/systemd/system/ -p wa -k service_modification

# Cron changes
-w /etc/cron.d/ -p wa -k cron_modification
-w /var/spool/cron/ -p wa -k cron_modification

# Privileged command execution
-a always,exit -F path=/usr/bin/sudo -F perm=x -F auid>=1000 -F auid!=-1 -k priv_cmd
-a always,exit -F path=/usr/bin/su -F perm=x -F auid>=1000 -F auid!=-1 -k priv_cmd

# DO NOT add -e 2 (immutable) until rules are finalized
# -e 2 requires a reboot to change any rules
EOF
```

**Load the rules:**
```bash
sudo augenrules --load

# Verify rules are active
sudo auditctl -l
# Must list all the rules you just wrote
```

**Generate a test event and verify it appears in audit.log:**
```bash
sudo useradd auditverify
sudo userdel auditverify

sudo grep "user_modification\|user_mgmt" /var/log/audit/audit.log | tail -10
# Must show the useradd/userdel events
```

**Enable syslog forwarding so Promtail can pick up audit events:**
```bash
# Check current state
cat /etc/audit/plugins.d/syslog.conf

sudo sed -i 's/^active = no/active = yes/' /etc/audit/plugins.d/syslog.conf

sudo systemctl restart auditd

# Verify audit events now appear in syslog
sudo grep "type=USER_MGMT\|type=ADD_USER" /var/log/syslog | tail -5
```

**Generate another event and verify it reaches syslog:**
```bash
sudo useradd auditverify2
sudo userdel auditverify2
sudo grep "auditverify2" /var/log/syslog | tail -5
# Must show the event — syslog forwarding is working
```

**Add audit scrape job to Promtail on client01:**
```bash
sudo tee -a /etc/promtail/promtail.yaml > /dev/null << 'EOF'

  - job_name: audit
    static_configs:
      - targets: [localhost]
        labels:
          job: audit
          hostname: client01
          env: lab
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
EOF

sudo systemctl restart promtail
```

**Verify audit events appear in Grafana:**
```logql
{log_type="audit", hostname="client01"}
{log_type="audit"} |= "user_modification"
```

Must return results within 60 seconds after generating a test event (`sudo useradd test && sudo userdel test`).

---

### Step 11 — Ansible Automation for Auditd

Add an `auditd` role to the existing Ansible structure.

**`roles/auditd/tasks/main.yml`:**
```yaml
---
- name: Install auditd and plugins
  apt:
    name:
      - auditd
      - audispd-plugins
    state: present
    update_cache: yes

- name: Deploy audit rules
  template:
    src: audit.rules.j2
    dest: /etc/audit/rules.d/logging-platform.rules
    mode: '0640'
  notify: reload auditd rules

- name: Enable syslog forwarding for audit events
  lineinfile:
    path: /etc/audit/plugins.d/syslog.conf
    regexp: '^active'
    line: 'active = yes'
  notify: restart auditd

- name: Enable and start auditd
  systemd:
    name: auditd
    enabled: yes
    state: started
```

**`roles/auditd/handlers/main.yml`:**
```yaml
---
- name: reload auditd rules
  command: augenrules --load

- name: restart auditd
  systemd:
    name: auditd
    state: restarted
```

**`roles/auditd/templates/audit.rules.j2`:**
```bash
-D
-b 8192
-f 1

-w /etc/passwd -p wa -k user_modification
-w /etc/shadow -p wa -k user_modification
-w /etc/group -p wa -k group_modification
-w /etc/gshadow -p wa -k group_modification
-w /etc/sudoers -p wa -k sudoers_modification
-w /etc/sudoers.d/ -p wa -k sudoers_modification

-w /usr/sbin/useradd -p x -k user_mgmt
-w /usr/sbin/userdel -p x -k user_mgmt
-w /usr/sbin/usermod -p x -k user_mgmt
-w /usr/sbin/groupadd -p x -k user_mgmt
-w /usr/sbin/groupdel -p x -k user_mgmt
-w /usr/sbin/passwd -p x -k user_mgmt

-w /etc/ssh/sshd_config -p wa -k ssh_config
-w /etc/systemd/system/ -p wa -k service_modification
-w /etc/cron.d/ -p wa -k cron_modification
-w /var/spool/cron/ -p wa -k cron_modification

-a always,exit -F path=/usr/bin/sudo -F perm=x -F auid>=1000 -F auid!=-1 -k priv_cmd
-a always,exit -F path=/usr/bin/su -F perm=x -F auid>=1000 -F auid!=-1 -k priv_cmd
```

**Update `deploy-agents.yml` to include auditd:**
```yaml
---
- name: Deploy agents to all clients
  hosts: clients
  become: yes
  roles:
    - auditd
    - logging-agent
```

**Update `promtail.yaml.j2` to include the audit scrape job:**
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

**Run and test idempotency:**
```bash
ansible-playbook -i inventory/hosts.ini deploy-agents.yml --ask-vault-pass

# Second run must show changed=0
ansible-playbook -i inventory/hosts.ini deploy-agents.yml --ask-vault-pass
```

---

## Part 4 — Application Logs (Manual — Add to Ansible After Verification)

### Step 12 — PostgreSQL Logs (on the node running PostgreSQL)

**Enable logging in PostgreSQL:**
```bash
# Find the config file location
sudo -u postgres psql -c "SHOW config_file;"

sudo nano /etc/postgresql/*/main/postgresql.conf
```

Set these values:
```
log_destination = 'stderr'
logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d.log'
log_rotation_age = 1d
log_min_duration_statement = 1000
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_connections = on
log_disconnections = on
log_lock_waits = on
```

```bash
sudo systemctl reload postgresql

# Verify log directory and file exist
ls -la /var/log/postgresql/
sudo tail -10 /var/log/postgresql/postgresql-$(date +%Y-%m-%d).log
```

**Verify Promtail can read PostgreSQL logs:**
```bash
# Check permissions
ls -la /var/log/postgresql/

# Test read access with your user (Promtail runs as your service user)
sudo tail -5 /var/log/postgresql/postgresql-$(date +%Y-%m-%d).log

# If permission denied — fix permissions
sudo chmod 755 /var/log/postgresql
```

**Add to Promtail config:**
```yaml
  - job_name: postgresql
    static_configs:
      - targets: [localhost]
        labels:
          job: postgresql
          hostname: client01
          env: lab
          service: postgresql
          log_type: db
          __path__: /var/log/postgresql/postgresql-*.log
```

**Generate a test event and verify:**
```bash
# Auth failure
psql -U nonexistent -d postgres 2>/dev/null || true

# Slow query (triggers log_min_duration_statement = 1000ms)
sudo -u postgres psql -c "SELECT pg_sleep(1.5);"
```

In Grafana:
```logql
{service="postgresql"} |= "authentication failed"
{service="postgresql"} |= "duration"
```

---

### Step 13 — Nginx Logs (on the node running Nginx)

**Configure JSON log format:**
```bash
sudo nano /etc/nginx/nginx.conf
```

Add inside the `http {}` block:
```nginx
log_format json_combined escape=json '{'
  '"time":"$time_iso8601",'
  '"remote_addr":"$remote_addr",'
  '"status":$status,'
  '"request":"$request",'
  '"body_bytes_sent":$body_bytes_sent,'
  '"request_time":$request_time'
'}';

access_log /var/log/nginx/access.log json_combined flush=1s;
error_log /var/log/nginx/error.log warn;
```

```bash
sudo nginx -t
# Must return: syntax is ok

sudo nginx -s reload

# Verify JSON format
curl -s http://localhost/ > /dev/null
sudo tail -3 /var/log/nginx/access.log
# Must be a valid JSON line
```

**Add to Promtail config:**
```yaml
  - job_name: nginx-access
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx
          hostname: client02
          env: lab
          service: nginx
          log_type: web
          __path__: /var/log/nginx/access.log
    pipeline_stages:
      - json:
          expressions:
            status: status
      - labels:
          status:

  - job_name: nginx-error
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx-error
          hostname: client02
          env: lab
          service: nginx
          log_type: web
          __path__: /var/log/nginx/error.log
```

**Verify:**
```bash
curl -s http://localhost/doesnotexist > /dev/null   # 404
```

```logql
{service="nginx"} | json | status >= 400
```

---

## Part 5 — Dashboards

Build in Grafana UI. Test every query in Explore before adding it to a panel.

**Set up provisioning so dashboards survive reinstalls:**
```bash
sudo mkdir -p /var/lib/grafana/dashboards
sudo chown grafana:grafana /var/lib/grafana/dashboards

sudo tee /etc/grafana/provisioning/dashboards/default.yaml > /dev/null << 'EOF'
apiVersion: 1
providers:
  - name: default
    folder: Logging Platform
    type: file
    options:
      path: /var/lib/grafana/dashboards
      updateIntervalSeconds: 30
EOF

sudo systemctl restart grafana-server
```

After building each dashboard in the UI: **Share → Export → Save to file** → copy to `/var/lib/grafana/dashboards/`.

**Dashboard panel queries — verify each in Explore before using:**

```logql
# Auth Dashboard
{log_type="auth"} |= "Failed password"
{log_type="auth"} |= "COMMAND"
count_over_time({log_type="auth"} |= "Failed password" [24h])
sum(rate({log_type="auth"} |= "Failed password" [$__interval]))

# Audit Dashboard
{log_type="audit"} |= "user_modification"
{log_type="audit"} |= "sudoers_modification"
{log_type="audit", audit_key="user_mgmt"}

# PostgreSQL Dashboard
{service="postgresql"} |= "ERROR"
{service="postgresql"} |= "authentication failed"
{service="postgresql"} |= "duration"

# Nginx Dashboard
{service="nginx"} | json | status >= 400
{service="nginx"} | json | status >= 500
count_over_time({service="nginx"} [$__interval])
```

**Add these template variables to every dashboard:**

| Variable | Query |
|---|---|
| `hostname` | `label_values(hostname)` |
| `env` | `label_values(env)` |
| `service` | `label_values(service)` |

Use `$hostname` in all queries so dashboards are filterable per node.

---

## Final Validation

Generate every event type. Verify each one in Grafana before marking done.

| Event | Command | Grafana Query | Pass? |
|---|---|---|---|
| Failed SSH | `ssh baduser@client01` from workstation | `{log_type="auth"} \|= "Failed password"` | [ ] |
| Successful SSH | `ssh youruser@client01` from workstation | `{log_type="auth"} \|= "Accepted"` | [ ] |
| Sudo command | `sudo ls /root` on client01 | `{log_type="auth"} \|= "COMMAND"` | [ ] |
| User creation | `sudo useradd finaltest` on client01 | `{log_type="audit"} \|= "user_modification"` | [ ] |
| Sudoers change | `sudo touch /etc/sudoers.d/ztest && sudo rm /etc/sudoers.d/ztest` | `{log_type="audit"} \|= "sudoers_modification"` | [ ] |
| PostgreSQL auth fail | `psql -U nobody -d postgres` | `{service="postgresql"} \|= "authentication failed"` | [ ] |
| Slow query | `sudo -u postgres psql -c "SELECT pg_sleep(1.5);"` | `{service="postgresql"} \|= "duration"` | [ ] |
| Nginx 404 | `curl http://client02/doesnotexist` | `{service="nginx"} \| json \| status == 404` | [ ] |
| System log | `logger "test validation"` on any client | `{log_type="system"} \|= "test validation"` | [ ] |

Clean up:
```bash
sudo userdel finaltest
sudo rm -f /etc/sudoers.d/ztest
```

---

## Known Issues to Fix Later

| Gap | Risk | Fix When |
|---|---|---|
| No alerting | Failed SSH brute force generates no notification | Phase 2 — Grafana alerting |
| Plaintext log transport | Log content visible on network | Phase 2 — TLS between Promtail and Loki |
| No auth on Loki push endpoint | Any internal host can push fake logs | Phase 2 — nginx with basic auth in front of Loki |
| Grafana on HTTP | Sessions exposed | Phase 2 — nginx + TLS |
| No Loki data backup | Disk failure = data loss | Phase 2 — scheduled backup of `/var/lib/loki` |
| `-e 2` not set on auditd | Audit rules can be modified without a reboot | After rules are stable and tested |

---

## LogQL Quick Reference

```logql
{hostname="client01"}
{log_type="auth"} |= "Failed password"
{log_type="auth"} |= "Accepted publickey"
{log_type="auth"} |= "COMMAND"
{log_type="audit", audit_key="user_modification"}
{log_type="audit"} |= "sudoers_modification"
{service="postgresql"} |= "ERROR"
{service="postgresql"} |= "authentication failed"
{service="nginx"} | json | status >= 400
{service="nginx"} | json | status >= 500
count_over_time({log_type="auth"} |= "Failed password" [24h])
sum(rate({log_type="auth"} |= "Failed password" [$__interval]))
{hostname="$hostname", log_type="auth"} |= "Failed password"
```
