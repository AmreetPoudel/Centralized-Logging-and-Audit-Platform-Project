# Centralized Logging & Audit Platform — Final Build Plan

> **Philosophy:** Manual first. Verify it works. Then automate.
> Loki and Grafana are one-time manual installs on the logging server.
> Promtail and Auditd go on hundreds of nodes — manual on one node first, then Ansible.
> No passwordless SSH. No passwordless sudo. No extra users created just for tooling.
> Every step has a verification command. Do not proceed until it passes.

---

## What Gets Automated vs What Does Not

| Component | Method |
|---|---|
| Loki | Manual — one time on logging server |
| Grafana | Manual — one time on logging server |
| Promtail | Manual on first node to verify, then Ansible |
| Auditd | Manual on first node to verify, then Ansible |
| Firewall | Not touched — configure separately |

---

## Architecture

```
┌─────────────────────────────────────┐
│  LOGGING SERVER                     │
│  Loki    :3100                      │
│  Grafana :3000                      │
└─────────────────────────────────────┘
        ▲            ▲            ▲
   Promtail      Promtail     Promtail
   node01        node02       node03
```

---

## Label Strategy — Define Once, Never Deviate

| Label | Values | Set By |
|---|---|---|
| `hostname` | node hostname | Ansible `inventory_hostname` |
| `env` | `lab`, `staging`, `prod` | Ansible group var |
| `service` | `ssh`, `sudo`, `audit`, `system`, `nginx`, `postgresql` | Per scrape job |
| `log_type` | `auth`, `audit`, `system`, `web`, `db` | Per scrape job |

Never put IP addresses, usernames, request IDs, or URLs into labels. High-cardinality values go in the log line and are filtered with `|=`.

---

## Pre-Start Checklist

- [ ] Ubuntu 22.04 or 24.04 on all nodes — do not mix
- [ ] Static IPs on all nodes
- [ ] `/etc/hosts` or DNS resolving all hostnames
- [ ] NTP synchronized: `timedatectl` shows `NTP synchronized: yes` on every node
- [ ] Loki version: **2.9.8** — this plan is tested on this version
- [ ] Separate disk available for Loki data (minimum 20GB)
- [ ] SSH access with your normal user to all nodes
- [ ] Your user has sudo access (with password) on all nodes

---

## Part 1 — Logging Server (Manual Only)

### Step 1 — Prepare Loki Data Volume

```bash
lsblk
# Identify the separate disk — adjust /dev/sdb below to your actual device

mkfs.ext4 /dev/sdb
mkdir -p /var/lib/loki

blkid /dev/sdb
# Copy the UUID from output

echo "UUID=your-uuid-here  /var/lib/loki  ext4  defaults  0  2" >> /etc/fstab
mount -a

df -h /var/lib/loki
# Must show /dev/sdb mounted here — not the root filesystem
# If it shows root filesystem, stop and fix before continuing
```

---

### Step 2 — Install Loki

```bash
cd /tmp
LOKI_VERSION="2.9.8"

wget https://github.com/grafana/loki/releases/download/v${LOKI_VERSION}/loki-linux-amd64.zip
unzip loki-linux-amd64.zip

./loki-linux-amd64 --version
# Verify version matches before installing

sudo mv loki-linux-amd64 /usr/local/bin/loki
sudo chmod +x /usr/local/bin/loki
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

Verify config before creating the service:
```bash
loki -config.file=/etc/loki/loki.yaml -verify-config
# Warnings about deprecated flags are acceptable — errors are not
```

> `from: 2024-01-01` must always be a past date. A future date causes Loki to silently reject all writes.

---

### Step 4 — Loki Systemd Service

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

Verify:
```bash
sudo systemctl status loki
sudo journalctl -u loki -n 30 --no-pager
# No level=error lines

curl -s http://localhost:3100/ready
# Must return: ready
# First run may show "Ingester not ready: waiting for 15s" — wait and retry, this is normal

curl -s http://localhost:3100/loki/api/v1/labels
# Must return: {"status":"success"}
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
```

---

### Step 6 — Configure Grafana

```bash
sudo nano /etc/grafana/grafana.ini
```

Find and set — do not replace the whole file:

Under `[security]`:
```ini
admin_password = your_strong_password_here
```

Under `[users]`:
```ini
allow_sign_up = false
```

Under `[auth.anonymous]`:
```ini
enabled = false
```

Provision the Loki datasource:
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

Verify:
```bash
sudo systemctl status grafana-server
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/login
# Must return: 200
```

Open `http://LOGGING_SERVER_IP:3000` — log in with `admin` and your password.

Go to **Connections → Data sources → Loki → Test** — must show green.

Go to **Explore → select Loki → run** `{job="test"}` — no results is correct, no connection error is what you are verifying.

---

### Logging Server Validation Gate

- [ ] `curl http://localhost:3100/ready` returns `ready`
- [ ] `systemctl status loki` — active, running
- [ ] `journalctl -u loki -n 30` — no `level=error` lines
- [ ] `df -h /var/lib/loki` — separate volume, not root filesystem
- [ ] Grafana accessible, admin password changed, Loki datasource green

---

## Part 2 — Promtail: System and Auth Logs (Manual on First Node)

### Step 7 — Install Promtail

SSH into the first node and run as your normal user with sudo.

**Verify log files exist and are readable:**
```bash
ls -la /var/log/syslog /var/log/auth.log
sudo tail -5 /var/log/syslog
sudo tail -5 /var/log/auth.log
# Both must return content
```

**Create promtail system user:**
```bash
sudo useradd --system --no-create-home --shell /bin/false -G adm promtail

id promtail
# Must show adm in groups

sudo -u promtail cat /var/log/syslog | head -3
sudo -u promtail cat /var/log/auth.log | head -3
# Both must return content — if permission denied, stop and fix group membership
```

> **Note on Ubuntu 24.04:** Promtail 2.9.8 ignores `User=` in the systemd unit and runs as root regardless. The `promtail` user is still created for future compatibility. The service is hardened with systemd security directives to limit blast radius.

**Install binary:**
```bash
cd /tmp
PROMTAIL_VERSION="2.9.8"

wget https://github.com/grafana/loki/releases/download/v${PROMTAIL_VERSION}/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip

./promtail-linux-amd64 --version

sudo mv promtail-linux-amd64 /usr/local/bin/promtail
sudo chown root:root /usr/local/bin/promtail
sudo chmod +x /usr/local/bin/promtail
```

**Create directories:**
```bash
sudo mkdir -p /etc/promtail /var/lib/promtail
sudo chown -R promtail:promtail /var/lib/promtail
```

---

### Step 8 — Configure Promtail for System and Auth Logs Only

At this point auditd is not installed yet. The config only covers system and auth logs.

```bash
sudo tee /etc/promtail/promtail.yaml > /dev/null << 'EOF'
server:
  http_listen_port: 9080
  grpc_listen_port: 0
  log_level: warn

positions:
  filename: /var/lib/promtail/positions.yaml

clients:
  - url: http://LOKI_SERVER_IP:3100/loki/api/v1/push
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
          hostname: HOSTNAME
          env: lab
          service: system
          log_type: system
          __path__: /var/log/syslog

  - job_name: auth
    static_configs:
      - targets: [localhost]
        labels:
          job: auth
          hostname: HOSTNAME
          env: lab
          service: ssh
          log_type: auth
          __path__: /var/log/auth.log
EOF
```

Replace `LOKI_SERVER_IP` and `HOSTNAME` with actual values.

**Verify config and connectivity:**
```bash
promtail -config.file=/etc/promtail/promtail.yaml -dry-run
# Must exit with no errors

curl -s http://LOKI_SERVER_IP:3100/ready
# Must return: ready — if not, fix connectivity before starting the service
```

**Create systemd service:**
```bash
sudo tee /etc/systemd/system/promtail.service > /dev/null << 'EOF'
[Unit]
Description=Promtail Log Shipping Agent
After=network.target

[Service]
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/promtail.yaml
Restart=on-failure
RestartSec=5s
LimitNOFILE=65536
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/var/lib/promtail
ReadOnlyPaths=/var/log /etc/promtail

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable promtail
sudo systemctl start promtail
```

**Verify:**
```bash
sudo systemctl status promtail
sudo journalctl -u promtail -n 30 --no-pager
# No connection refused, no permission denied

sudo cat /var/lib/promtail/positions.yaml
# Must show entries for syslog and auth.log with non-zero offsets

curl -s http://localhost:9080/metrics | grep promtail_sent_bytes_total
# Must return the metric line
```

---

### Step 9 — Verify System and Auth Logs in Grafana

Generate events and confirm each appears in Grafana before moving on.

```bash
sudo ls /root
ssh nonexistent@localhost 2>/dev/null || true
grep "COMMAND\|Failed password" /var/log/auth.log | tail -5
```

Grafana Explore:
```logql
{log_type="system", hostname="your-hostname"}
{log_type="auth"} |= "COMMAND"
{log_type="auth"} |= "Failed password"
```

All must return results within 60 seconds. Only then move to auditd.

---

## Part 3 — Auditd (Manual on First Node)

### Step 10 — Install Auditd

```bash
sudo apt-get install -y auditd audispd-plugins

sudo systemctl enable auditd
sudo systemctl start auditd
sudo systemctl status auditd

ls -la /var/log/audit/audit.log
sudo tail -5 /var/log/audit/audit.log
# Must show recent events
```

---

### Step 11 — Write Audit Rules

```bash
sudo tee /etc/audit/rules.d/logging-platform.rules > /dev/null << 'EOF'
## Clear existing rules
-D

## Buffer size — prevents dropped events under load
-b 8192

## Backlog wait time
--backlog_wait_time 60000

## Failure mode: 1 = silent, 2 = panic. Use 1 in production.
-f 1

## User and group file changes
-w /etc/passwd -p wa -k user_modification
-w /etc/shadow -p wa -k user_modification
-w /etc/group -p wa -k group_modification
-w /etc/gshadow -p wa -k group_modification
-w /etc/sudoers -p wa -k sudoers_modification
-w /etc/sudoers.d/ -p wa -k sudoers_modification

## User management commands
-w /usr/sbin/useradd -p x -k user_mgmt
-w /usr/sbin/userdel -p x -k user_mgmt
-w /usr/sbin/usermod -p x -k user_mgmt
-w /usr/sbin/groupadd -p x -k user_mgmt
-w /usr/sbin/groupdel -p x -k user_mgmt
-w /usr/sbin/passwd -p x -k user_mgmt

## SSH config changes
-w /etc/ssh/sshd_config -p wa -k ssh_config

## Systemd service changes
-w /etc/systemd/system/ -p wa -k service_modification

## Cron changes
-w /etc/cron.d/ -p wa -k cron_modification
-w /var/spool/cron/ -p wa -k cron_modification

## Privileged command execution
-a always,exit -F path=/usr/bin/sudo -F perm=x -F auid>=1000 -F auid!=-1 -k priv_cmd
-a always,exit -F path=/usr/bin/su -F perm=x -F auid>=1000 -F auid!=-1 -k priv_cmd

## DO NOT enable immutable mode until rules are finalised
## -e 2
EOF
```

Load and verify:
```bash
sudo augenrules --load
sudo auditctl -l
# Must list every rule above
```

---

### Step 12 — Test Every Rule Category

Run each command and verify the event appears in `/var/log/audit/audit.log` first.

```bash
# user_modification
sudo useradd auditverify
sudo userdel auditverify
sudo grep "user_modification" /var/log/audit/audit.log | tail -3

# user_mgmt
sudo useradd auditverify2
sudo userdel auditverify2
sudo grep "user_mgmt" /var/log/audit/audit.log | tail -3

# sudoers_modification
sudo touch /etc/sudoers.d/auditverify
sudo rm /etc/sudoers.d/auditverify
sudo grep "sudoers_modification" /var/log/audit/audit.log | tail -3

# ssh_config
sudo touch /etc/ssh/sshd_config
sudo grep "ssh_config" /var/log/audit/audit.log | tail -3

# priv_cmd
sudo ls /root
sudo grep "priv_cmd" /var/log/audit/audit.log | tail -3
```

Each `grep` must return events. If any returns nothing, the rule is not working — check `auditctl -l` to confirm it loaded.

---

### Auditd Validation Gate

- [ ] `auditctl -l` shows all rules loaded
- [ ] `sudo useradd` generates event with key `user_mgmt` in audit.log
- [ ] `sudo touch /etc/sudoers.d/test` generates `sudoers_modification` event
- [ ] `sudo ls /root` generates `priv_cmd` event
- [ ] All events visible in `/var/log/audit/audit.log` before proceeding

---

## Part 4 — Add Audit Logs to Promtail

Auditd is working and verified. Now add the audit scrape job to the existing Promtail config.

### Step 13 — Add Audit Scrape Job

```bash
sudo tee -a /etc/promtail/promtail.yaml > /dev/null << 'EOF'

  - job_name: audit
    static_configs:
      - targets: [localhost]
        labels:
          job: audit
          hostname: HOSTNAME
          env: lab
          service: audit
          log_type: audit
          __path__: /var/log/audit/audit.log
    pipeline_stages:
      - regex:
          expression: '.*key="(?P<audit_key>[^"]+)"'
      - labels:
          audit_key:
EOF
```

Replace `HOSTNAME` with the actual hostname.

```bash
sudo systemctl restart promtail
sudo systemctl status promtail
sudo journalctl -u promtail -n 20 --no-pager
```

---

### Step 14 — Verify Audit Events in Grafana

Generate events and confirm they appear:

```bash
sudo useradd finalverify
sudo userdel finalverify
sudo touch /etc/sudoers.d/ztest && sudo rm /etc/sudoers.d/ztest
```

Grafana Explore:
```logql
{log_type="audit"}
{log_type="audit", audit_key="user_mgmt"}
{log_type="audit", audit_key="sudoers_modification"}
{log_type="audit", audit_key="priv_cmd"}
```

All must return results within 60 seconds. Only proceed to Ansible after all pass.

---

## Part 5 — Ansible Automation

Everything above is verified manually. Now encode it into Ansible. No roles — flat playbooks with separate Jinja2 templates per service.

```
ansible/
├── inventory.ini
├── group_vars/
│   └── all/
│       ├── vars.yml
│       └── vault.yml          ← encrypted
├── templates/
│   ├── promtail.yaml.j2
│   ├── promtail.service.j2
│   └── audit.rules.j2
├── install_promtail.yml
├── install_auditd.yml
└── site.yml
```

---

### inventory.ini

```ini
[nodes]
node01 ansible_host=192.168.1.11
node02 ansible_host=192.168.1.12
node03 ansible_host=192.168.1.13

[nodes:vars]
loki_url=LOKI_SERVER_IP:3100
env=lab
```

---

### group_vars/all/vars.yml

```yaml
promtail_version: "2.9.8"
ansible_user: your_username
ansible_become: true
ansible_ssh_pass: "{{ vault_ssh_pass }}"
ansible_become_pass: "{{ vault_become_pass }}"
```

---

### group_vars/all/vault.yml

Create then encrypt:
```yaml
vault_ssh_pass: "ssh_password_here"
vault_become_pass: "sudo_password_here"
```

```bash
ansible-vault encrypt group_vars/all/vault.yml
```

If passwords differ per host, use `host_vars/` instead:
```bash
mkdir -p host_vars
# Create host_vars/node01.yml with that node's passwords
ansible-vault encrypt host_vars/node01.yml
```

---

### templates/promtail.yaml.j2

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
      - regex:
          expression: '.*key="(?P<audit_key>[^"]+)"'
      - labels:
          audit_key:
```

The audit scrape job is included here because by the time Ansible runs, auditd is already installed and verified — `site.yml` runs `install_auditd.yml` before `install_promtail.yml`.

---

### templates/promtail.service.j2

```ini
[Unit]
Description=Promtail Log Shipping Agent
After=network.target

[Service]
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/promtail.yaml
Restart=on-failure
RestartSec=5s
LimitNOFILE=65536
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/var/lib/promtail
ReadOnlyPaths=/var/log /etc/promtail

[Install]
WantedBy=multi-user.target
```

---

### templates/audit.rules.j2

```bash
## Clear existing rules
-D

## Buffer size
-b 8192

## Backlog wait time
--backlog_wait_time 60000

## Failure mode: 1 = silent
-f 1

## User and group file changes
-w /etc/passwd -p wa -k user_modification
-w /etc/shadow -p wa -k user_modification
-w /etc/group -p wa -k group_modification
-w /etc/gshadow -p wa -k group_modification
-w /etc/sudoers -p wa -k sudoers_modification
-w /etc/sudoers.d/ -p wa -k sudoers_modification

## User management commands
-w /usr/sbin/useradd -p x -k user_mgmt
-w /usr/sbin/userdel -p x -k user_mgmt
-w /usr/sbin/usermod -p x -k user_mgmt
-w /usr/sbin/groupadd -p x -k user_mgmt
-w /usr/sbin/groupdel -p x -k user_mgmt
-w /usr/sbin/passwd -p x -k user_mgmt

## SSH config changes
-w /etc/ssh/sshd_config -p wa -k ssh_config

## Systemd service changes
-w /etc/systemd/system/ -p wa -k service_modification

## Cron changes
-w /etc/cron.d/ -p wa -k cron_modification
-w /var/spool/cron/ -p wa -k cron_modification

## Privileged command execution
-a always,exit -F path=/usr/bin/sudo -F perm=x -F auid>=1000 -F auid!=-1 -k priv_cmd
-a always,exit -F path=/usr/bin/su -F perm=x -F auid>=1000 -F auid!=-1 -k priv_cmd
```

---

### install_auditd.yml

```yaml
---
- name: Install and configure Auditd
  hosts: nodes
  become: yes

  tasks:
    - name: Install auditd and plugins
      apt:
        name:
          - auditd
          - audispd-plugins
        state: present
        update_cache: yes

    - name: Deploy audit rules
      template:
        src: templates/audit.rules.j2
        dest: /etc/audit/rules.d/logging-platform.rules
        mode: '0640'
      notify: reload audit rules

    - name: Enable and start auditd
      systemd:
        name: auditd
        enabled: yes
        state: started

    - name: Verify audit rules loaded
      command: auditctl -l
      register: audit_rules
      changed_when: false

    - name: Fail if no rules loaded
      fail:
        msg: "Audit rules not loaded on {{ inventory_hostname }}"
      when: audit_rules.stdout_lines | length < 5

    - name: Print audit rules count per host
      debug:
        msg: "{{ inventory_hostname }} — {{ audit_rules.stdout_lines | length }} audit rules loaded"

  handlers:
    - name: reload audit rules
      command: augenrules --load
```

---

### install_promtail.yml

```yaml
---
- name: Install and configure Promtail
  hosts: nodes
  become: yes

  tasks:
    - name: Check Loki is reachable before proceeding
      uri:
        url: "http://{{ loki_url }}/ready"
        return_content: yes
      register: loki_check
      failed_when: "'ready' not in loki_check.content"
      delegate_to: localhost
      become: no

    - name: Create promtail system user in adm group
      user:
        name: promtail
        system: yes
        shell: /bin/false
        create_home: no
        groups: adm
        append: yes

    - name: Create promtail directories
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - /etc/promtail
        - /var/lib/promtail

    - name: Set ownership on positions directory
      file:
        path: /var/lib/promtail
        owner: promtail
        group: promtail
        recurse: yes

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
        owner: root
        group: root
        mode: '0755'
        remote_src: yes
      notify: restart promtail

    - name: Deploy promtail config
      template:
        src: templates/promtail.yaml.j2
        dest: /etc/promtail/promtail.yaml
        mode: '0640'
      notify: restart promtail

    - name: Deploy promtail systemd unit
      template:
        src: templates/promtail.service.j2
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

    - name: Wait for promtail to start
      pause:
        seconds: 10

    - name: Verify promtail is running
      systemd:
        name: promtail
      register: promtail_status

    - name: Fail if promtail is not running
      fail:
        msg: "Promtail is not running on {{ inventory_hostname }} — check journalctl -u promtail"
      when: promtail_status.status.ActiveState != "active"

    - name: Print status per host
      debug:
        msg: "{{ inventory_hostname }} — promtail {{ promtail_version }} — {{ promtail_status.status.ActiveState }}"

  handlers:
    - name: reload systemd
      systemd:
        daemon_reload: yes

    - name: restart promtail
      systemd:
        name: promtail
        state: restarted
```

---

### site.yml

Auditd runs first — by the time Promtail is deployed, `/var/log/audit/audit.log` already exists on every node.

```yaml
---
- import_playbook: install_auditd.yml
- import_playbook: install_promtail.yml
```

---

### Running the Playbooks

**Deploy everything:**
```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
```

**Deploy only Promtail:**
```bash
ansible-playbook -i inventory.ini install_promtail.yml --ask-vault-pass
```

**Deploy only Auditd:**
```bash
ansible-playbook -i inventory.ini install_auditd.yml --ask-vault-pass
```

**Deploy to a single new node:**
```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass -l node04
```

**Idempotency test — run twice, second must show changed=0:**
```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
# Second run: ok=N  changed=0  failed=0 on all hosts
```

---

## Final Validation Matrix

Generate each event and verify it appears in Grafana before marking done.

| Event | Command | Grafana Query | Pass |
|---|---|---|---|
| Failed SSH | `ssh baduser@node01` from workstation | `{log_type="auth"} \|= "Failed password"` | [ ] |
| Successful SSH | `ssh youruser@node01` from workstation | `{log_type="auth"} \|= "Accepted"` | [ ] |
| Sudo command | `sudo ls /root` on node01 | `{log_type="auth"} \|= "COMMAND"` | [ ] |
| User creation | `sudo useradd finaltest` on node01 | `{log_type="audit", audit_key="user_mgmt"}` | [ ] |
| User deletion | `sudo userdel finaltest` on node01 | `{log_type="audit", audit_key="user_mgmt"}` | [ ] |
| Sudoers change | `sudo touch /etc/sudoers.d/ztest && sudo rm /etc/sudoers.d/ztest` | `{log_type="audit", audit_key="sudoers_modification"}` | [ ] |
| SSH config touch | `sudo touch /etc/ssh/sshd_config` | `{log_type="audit", audit_key="ssh_config"}` | [ ] |
| Priv command | `sudo ls /root` | `{log_type="audit", audit_key="priv_cmd"}` | [ ] |
| System log | `logger "test validation"` on any node | `{log_type="system"} \|= "test validation"` | [ ] |

Clean up after validation:
```bash
sudo userdel finaltest 2>/dev/null || true
sudo rm -f /etc/sudoers.d/ztest
```

---

## Known Gaps — Fix in Phase 2

| Gap | Risk | Fix |
|---|---|---|
| Promtail runs as root on Ubuntu 24.04 | Binary ignores `User=` directive — mitigated with systemd hardening | Investigate newer Promtail version |
| No alerting | SSH brute force generates no notification | Grafana alerting rules on `count_over_time` thresholds |
| Plaintext log transport | Log content visible on the network | TLS between Promtail and Loki |
| No auth on Loki push endpoint | Any internal host can push logs | nginx with basic auth in front of port 3100 |
| Grafana on HTTP | Sessions exposed | nginx + TLS termination |
| No Loki data backup | Disk failure = full data loss | Scheduled backup of `/var/lib/loki` |
| Auditd `-e 2` not set | Rules can be modified without reboot | Enable after rules are stable and tested |

---

## LogQL Quick Reference

```logql
{hostname="node01"}
{log_type="auth"} |= "Failed password"
{log_type="auth"} |= "Accepted publickey"
{log_type="auth"} |= "COMMAND"
{log_type="audit", audit_key="user_modification"}
{log_type="audit", audit_key="user_mgmt"}
{log_type="audit", audit_key="sudoers_modification"}
{log_type="audit", audit_key="priv_cmd"}
{log_type="audit", audit_key="ssh_config"}
count_over_time({log_type="auth"} |= "Failed password" [24h])
sum(rate({log_type="auth"} |= "Failed password" [$__interval]))
{hostname="$hostname", log_type="$log_type"}
```

