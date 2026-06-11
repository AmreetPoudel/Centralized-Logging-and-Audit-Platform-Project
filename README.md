# Centralized Logging & Audit Platform — Production Build Plan

> **Core Philosophy:** Every step is done manually first and verified with an exact command. Only after the manual step passes do you write the Ansible task for it. Automating something you cannot do manually is automating a broken process.

> **This document is the implementation record.** When a step is done, mark it. When a step fails, write down why and what fixed it — that is more valuable than the original plan.

---

## Pre-Implementation Checklist

Resolve every item here before Day 1. Each one is a full-day blocker if discovered later.

- [ ] OS version decided and committed — Ubuntu 22.04 LTS on all nodes (do not mix 22.04 and 24.04)
- [ ] Static IPs assigned to all nodes — DHCP will silently break Ansible inventory
- [ ] DNS entries or `/etc/hosts` prepared for all nodes
- [ ] SSH key-based auth working from your workstation to all nodes — test with `ssh -i ~/.ssh/your_key user@node`
- [ ] Network topology confirmed — logging server reachable from all clients on port 3100
- [ ] Cloud/hypervisor-level firewall rules checked (AWS Security Groups, GCP firewall, proxmox — these block traffic before the OS sees it)
- [ ] Separate disk or volume allocated for Loki data (minimum 20GB) — do not put log data on the root filesystem
- [ ] NTP consistent across all nodes — run `timedatectl` on each and confirm synchronized
- [ ] Loki version decided: **2.9.8** (v2.x and v3.x have breaking config differences — this plan targets 2.9.x)
- [ ] Non-root `ansible` user with `NOPASSWD` sudo created on all nodes before Day 1

### Node Layout

| Hostname | IP | Role |
|---|---|---|
| `logging-server` | `192.168.1.10` | Loki + Grafana |
| `client01` | `192.168.1.11` | PostgreSQL source |
| `client02` | `192.168.1.12` | Nginx source |
| `client03` | `192.168.1.13` | Application source |

> Replace IPs with your actual addresses. Every command in this document uses these values — do a find-replace before you start.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  LOGGING SERVER (192.168.1.10)                               │
│                                                              │
│  Loki     :3100  — log ingest and query                      │
│  Grafana  :3000  — dashboards                                │
│  nginx    :443   — TLS reverse proxy (Phase 2)               │
└──────────────────────────────────────────────────────────────┘
           ▲               ▲               ▲
     Promtail          Promtail        Promtail
     client01          client02        client03
     port → 3100       port → 3100     port → 3100
```

**Why no message queue (Kafka, Redis)?** At 3 clients with moderate log volume, Promtail direct push to Loki is the right call. A queue adds operational complexity that only pays off at 20+ nodes or burst-heavy workloads. This decision is documented here — revisit at scale.

---

## Label Strategy — Decide Before Writing Any Config

Inconsistent labels break every LogQL query and cause Loki cardinality problems. These are the labels for this platform. Do not deviate.

| Label | Allowed Values | Source |
|---|---|---|
| `hostname` | `client01`, `client02`, `client03` | `/etc/hostname` via Ansible var |
| `env` | `lab`, `staging`, `prod` | Ansible inventory group var |
| `service` | `ssh`, `sudo`, `postgresql`, `nginx`, `audit`, `system` | Per scrape job — hardcoded |
| `log_type` | `auth`, `audit`, `app`, `web`, `db`, `system` | Per scrape job — hardcoded |

**What never becomes a label:** IP addresses, usernames, request IDs, URLs, session tokens. High-cardinality values go in the log line and are queried with `|=` or regex filters. Putting them in labels creates one stream per value and kills Loki performance.

**Rule:** Every Promtail `scrape_config` must emit the same label set. A missing label on one client means you cannot filter across all clients.

---

## Day 1 — Infrastructure & Connectivity

### Goal
All four nodes exist, are reachable by SSH, have stable hostnames, and pass an Ansible connectivity check.

---

### Step 1.1 — Set Hostnames

**Manual — on each node individually:**

```bash
# On the logging server
hostnamectl set-hostname logging-server
hostname   # Verify immediately — should return "logging-server"

# On client01
hostnamectl set-hostname client01

# On client02
hostnamectl set-hostname client02

# On client03
hostnamectl set-hostname client03
```

**Verification:**
```bash
hostname
cat /etc/hostname
```

Both must return the correct name. `hostnamectl` persists across reboots; `hostname` alone does not.

---

### Step 1.2 — Populate /etc/hosts on All Nodes

**Manual — on every node:**

```bash
cat >> /etc/hosts << 'EOF'
192.168.1.10  logging-server
192.168.1.11  client01
192.168.1.12  client02
192.168.1.13  client03
EOF
```

**Verification — from each node, ping every other node by hostname:**
```bash
ping -c 2 logging-server
ping -c 2 client01
ping -c 2 client02
ping -c 2 client03
```

All four must resolve and respond. If any fails, `/etc/hosts` is missing or has the wrong IP. Fix before continuing.

---

### Step 1.3 — Create the Ansible User

**Manual — on every node:**

```bash
# Create user
useradd -m -s /bin/bash ansible

# Add SSH public key (replace with your actual public key)
mkdir -p /home/ansible/.ssh
echo "ssh-ed25519 AAAA...your_public_key_here..." >> /home/ansible/.ssh/authorized_keys
chmod 700 /home/ansible/.ssh
chmod 600 /home/ansible/.ssh/authorized_keys
chown -R ansible:ansible /home/ansible/.ssh

# Grant NOPASSWD sudo
echo "ansible ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/ansible
chmod 440 /etc/sudoers.d/ansible

# Verify sudo works without a password
su - ansible -c "sudo whoami"
# Must return: root
```

**Verification — from your workstation:**
```bash
ssh -i ~/.ssh/your_key ansible@192.168.1.10 "sudo whoami"
ssh -i ~/.ssh/your_key ansible@192.168.1.11 "sudo whoami"
ssh -i ~/.ssh/your_key ansible@192.168.1.12 "sudo whoami"
ssh -i ~/.ssh/your_key ansible@192.168.1.13 "sudo whoami"
```

Every command must return `root` with no password prompt. Any prompt or error means the user setup is incomplete.

---

### Step 1.4 — Firewall Rules

**Manual — on logging server:**
```bash
# Check current status first
ufw status

# Allow required ports
ufw allow 22/tcp     # SSH
ufw allow 3100/tcp   # Loki — ingest from clients
ufw allow 3000/tcp   # Grafana — restrict to your IP in prod

ufw enable
ufw status verbose
```

**Manual — verify from each client that port 3100 is reachable:**
```bash
# Run this on client01, client02, client03
nc -zv 192.168.1.10 3100
# Expected: Connection to 192.168.1.10 3100 port [tcp/*] succeeded!
```

If `nc` is not installed: `apt-get install -y netcat-openbsd`

If the connection fails but `ufw` shows the rule: check your cloud/hypervisor firewall. OS-level `ufw` is irrelevant if AWS Security Groups or GCP firewall rules block the traffic upstream.

---

### Step 1.5 — Separate Disk for Loki

**Manual — on logging server:**
```bash
# List available block devices
lsblk

# Assuming /dev/sdb is the separate 20GB disk — adjust to your actual device
mkfs.ext4 /dev/sdb
mkdir -p /var/lib/loki

# Get the UUID
blkid /dev/sdb

# Add to fstab using UUID (more reliable than device name)
echo "UUID=your-uuid-here  /var/lib/loki  ext4  defaults  0  2" >> /etc/fstab

# Mount it
mount -a

# Verify
df -h /var/lib/loki
# Must show /dev/sdb mounted at /var/lib/loki, not the root filesystem
```

**Critical check:** `df -h /var/lib/loki` must show a separate filesystem. If it shows the root filesystem (`/dev/sda` or similar), Loki data will fill your root disk and crash the server.

---

### Step 1.6 — Ansible Inventory and Connectivity Test

**Manual — create the inventory file on your workstation:**
```bash
mkdir -p ansible/inventory
```

`ansible/inventory/hosts.ini`:
```ini
[logging]
logging-server ansible_host=192.168.1.10

[clients]
client01 ansible_host=192.168.1.11
client02 ansible_host=192.168.1.12
client03 ansible_host=192.168.1.13

[clients:vars]
ansible_user=ansible
ansible_become=true
loki_url=192.168.1.10:3100

[logging:vars]
ansible_user=ansible
ansible_become=true

[all:vars]
env=lab
ansible_ssh_private_key_file=~/.ssh/your_key
```

**Verification:**
```bash
ansible all -i ansible/inventory/hosts.ini -m ping
```

Expected output for every host:
```
logging-server | SUCCESS => {"ping": "pong"}
client01       | SUCCESS => {"ping": "pong"}
client02       | SUCCESS => {"ping": "pong"}
client03       | SUCCESS => {"ping": "pong"}
```

Any `UNREACHABLE` or `FAILED` must be resolved before Day 2. Do not proceed past a broken ping.

---

### Day 1 Validation Gate

Do not proceed to Day 2 until every item is checked:

- [ ] `ansible all -m ping` — all four hosts return `pong`
- [ ] `nc -zv logging-server 3100` succeeds from every client
- [ ] `hostname` returns correct name on each node
- [ ] `df -h /var/lib/loki` shows a separate mounted filesystem on logging server
- [ ] NTP check: `timedatectl` on all nodes shows `NTP synchronized: yes`

### Known Failure Modes

**Cloud firewall blocking port 3100:** `ufw` shows the rule open but `nc` times out from the client. This means the cloud-level firewall (AWS Security Group, GCP VPC firewall, etc.) is blocking it. Fix at the cloud console, not on the OS.

**`/etc/hosts` entries duplicated:** If you run the `cat >>` command more than once, you get duplicate entries. Check with `cat /etc/hosts | grep logging-server`. Fix with manual edit if duplicated.

**Ansible SSH key not matching:** If `ssh -i ~/.ssh/your_key ansible@node` prompts for a password, the public key was not written correctly to `authorized_keys`. Verify the key content matches exactly.

---

## Day 2 — Install and Configure Loki

### Goal
Loki running on the logging server, accessible at port 3100, writing data to the separate volume, with verified retention.

---

### Step 2.1 — Create Loki System User and Directories

**Manual — on logging server:**
```bash
# Create unprivileged system user
useradd --system --no-create-home --shell /bin/false loki

# Create required directories
mkdir -p /etc/loki
mkdir -p /var/lib/loki/{index,chunks,wal,compactor,boltdb-cache,rules}

# Set ownership
chown -R loki:loki /var/lib/loki /etc/loki

# Verify ownership
ls -la /var/lib/loki/
# All directories must be owned by loki:loki
```

---

### Step 2.2 — Download and Install Loki Binary

**Manual — on logging server:**
```bash
cd /tmp
LOKI_VERSION="2.9.8"

wget https://github.com/grafana/loki/releases/download/v${LOKI_VERSION}/loki-linux-amd64.zip
unzip loki-linux-amd64.zip

# Verify the binary before moving it
./loki-linux-amd64 --version
# Expected: loki, version 2.9.8

mv loki-linux-amd64 /usr/local/bin/loki
chmod +x /usr/local/bin/loki

# Final verification
loki --version
```

---

### Step 2.3 — Write Loki Configuration

**Manual — on logging server:**

```bash
cat > /etc/loki/loki.yaml << 'EOF'
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

chunk_store_config:
  max_look_back_period: 30d

table_manager:
  retention_deletes_enabled: true
  retention_period: 30d
EOF

chown loki:loki /etc/loki/loki.yaml
chmod 640 /etc/loki/loki.yaml
```

**Verify the config is valid before creating the service:**
```bash
# Loki will exit immediately with an error if config is invalid
sudo -u loki loki -config.file=/etc/loki/loki.yaml -verify-config
# Expected: config is valid
```

> **`from: 2024-01-01` must be a past date.** If it is in the future, Loki starts successfully but silently rejects all incoming log writes with no useful error message. Always use a date well in the past.

---

### Step 2.4 — Create and Start the Systemd Service

**Manual — on logging server:**
```bash
cat > /etc/systemd/system/loki.service << 'EOF'
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
EOF

systemctl daemon-reload
systemctl enable loki
systemctl start loki
```

**Verification:**
```bash
# Service must be active
systemctl status loki

# No errors in first 30 seconds of logs
journalctl -u loki -n 50 --no-pager

# HTTP endpoint must respond
curl -s http://localhost:3100/ready
# Expected: ready

# Labels endpoint must respond with valid JSON
curl -s http://localhost:3100/loki/api/v1/labels
# Expected: {"status":"success","data":[]}  (empty is correct — no logs yet)
```

**Verify Loki is running as the correct user:**
```bash
ps aux | grep loki
# Must show "loki" in the USER column, not root
```

---

### Day 2 Validation Gate

- [ ] `curl http://localhost:3100/ready` returns `ready`
- [ ] `systemctl status loki` shows `active (running)`
- [ ] `journalctl -u loki -n 50` shows no ERROR lines
- [ ] `ps aux | grep loki` — process running as `loki` user, not root
- [ ] `df -h /var/lib/loki` — data directory is still on the separate volume (not root)

### Known Failure Modes

**Loki starts but writes silently fail:** Usually a `from` date in the future in `schema_config`. Check the date. Must be before today.

**`permission denied` on startup:** The `loki` user cannot write to `/var/lib/loki`. Verify with `ls -la /var/lib/loki` — all directories must be owned by `loki:loki`. Fix: `chown -R loki:loki /var/lib/loki`.

**Port 3100 already in use:** Some systems have other services on this port. Check: `ss -tlnp | grep 3100`.

**Do not set `log_level: debug` in production.** At `debug`, a moderately active system will write gigabytes of Loki internal logs per day and fill your disk. `warn` is the correct production setting.

---

## Day 3 — Install Grafana and Connect to Loki

### Goal
Grafana running at port 3000 with the Loki datasource connected, default admin password changed, and the datasource queryable.

---

### Step 3.1 — Install Grafana OSS

**Manual — on logging server:**
```bash
# Install dependencies
apt-get install -y apt-transport-https software-properties-common wget

# Add Grafana GPG key
mkdir -p /usr/share/keyrings
wget -q -O /usr/share/keyrings/grafana.key https://apt.grafana.com/gpg.key

# Add repository
echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com stable main" \
  > /etc/apt/sources.list.d/grafana.list

apt-get update
apt-get install -y grafana

# Verify installed version
grafana-server -v
```

---

### Step 3.2 — Configure Grafana Before First Start

**Manual — edit `/etc/grafana/grafana.ini`:**

Find and change these specific lines. Do not replace the whole file — Grafana's default ini is heavily commented and the comments are documentation.

```bash
# Change admin password (find the line, uncomment and change)
sed -i 's/^;admin_password = admin/admin_password = CHANGE_THIS_TO_SOMETHING_STRONG/' /etc/grafana/grafana.ini

# Disable user self-registration
sed -i 's/^;allow_sign_up = true/allow_sign_up = false/' /etc/grafana/grafana.ini

# Disable anonymous access
sed -i 's/^;enabled = false/enabled = false/' /etc/grafana/grafana.ini
```

**Safer approach — verify the changes actually took:**
```bash
grep "admin_password" /etc/grafana/grafana.ini
grep "allow_sign_up" /etc/grafana/grafana.ini
```

If the `sed` commands did not find the lines (Grafana ini format can vary by version), edit the file manually:
```bash
nano /etc/grafana/grafana.ini
```
Find the `[security]` section and set `admin_password` to something strong. Find `[users]` and set `allow_sign_up = false`.

---

### Step 3.3 — Provision the Loki Datasource

**Manual — create the provisioning file:**
```bash
mkdir -p /etc/grafana/provisioning/datasources

cat > /etc/grafana/provisioning/datasources/loki.yaml << 'EOF'
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

chown root:grafana /etc/grafana/provisioning/datasources/loki.yaml
chmod 640 /etc/grafana/provisioning/datasources/loki.yaml
```

Provisioned datasources cannot be deleted through the Grafana UI — this is intentional. To change the datasource, edit this file and restart Grafana.

---

### Step 3.4 — Start Grafana

**Manual:**
```bash
systemctl enable grafana-server
systemctl start grafana-server

# Wait 10 seconds for startup, then check
sleep 10
systemctl status grafana-server
journalctl -u grafana-server -n 30 --no-pager
```

**Verification:**
```bash
# Grafana should respond on port 3000
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/login
# Expected: 200
```

**Browser test:**
Open `http://192.168.1.10:3000` in a browser. Log in with `admin` and the password you set.

Navigate to: **Connections → Data sources → Loki → Test**

The test must return "Data source connected and labels found" or "Data source successfully connected." A connection error means either Loki is not running or the URL is wrong.

---

### Step 3.5 — Verify Loki is Queryable from Grafana

In Grafana: go to **Explore**, select the Loki datasource, and run this query:

```logql
{job="varlogs"}
```

It will return no results — that is correct. There are no logs yet. What you are verifying is that Grafana can reach Loki and return a response without an error. An error here means the datasource is misconfigured.

---

### Day 3 Validation Gate

- [ ] Grafana login page accessible at `http://192.168.1.10:3000`
- [ ] Default `admin` password changed and working
- [ ] Loki datasource shows green status in **Connections → Data sources**
- [ ] Grafana Explore runs a query without returning a connection error
- [ ] `journalctl -u grafana-server` shows no `permission denied` errors

### Known Failure Modes

**Datasource URL using external IP instead of localhost:** If Grafana and Loki are on the same machine, use `http://localhost:3100`. Using the external IP works but adds unnecessary routing and can break if firewall rules change.

**Provisioned datasource not appearing:** Grafana reads provisioning files only on startup. If you created the file after Grafana started, restart the service: `systemctl restart grafana-server`.

**Grafana cannot read the provisioning file:** Check file ownership. Grafana runs as the `grafana` user. The provisioning directory must be readable: `ls -la /etc/grafana/provisioning/datasources/`.

---

## Day 4 — Promtail Agent and System Log Collection

### Goal
System logs (`/var/log/syslog`) and auth logs (`/var/log/auth.log`) from all three clients flowing into Loki with correct labels, visible in Grafana within 30 seconds of generation.

---

### Step 4.1 — Install Promtail Manually on One Client First

**Manual — on client01 only:**
```bash
# Create promtail user with adm group membership
# adm group is required to read /var/log on Ubuntu
useradd --system --no-create-home --shell /bin/false -G adm promtail

# Verify the user is in the adm group
id promtail
# Must show: groups=...,4(adm),...

# Download and install
cd /tmp
PROMTAIL_VERSION="2.9.8"
wget https://github.com/grafana/loki/releases/download/v${PROMTAIL_VERSION}/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip

./promtail-linux-amd64 --version
# Must return version 2.9.8

mv promtail-linux-amd64 /usr/local/bin/promtail
chmod +x /usr/local/bin/promtail

# Create required directories
mkdir -p /etc/promtail /var/lib/promtail
chown promtail:promtail /etc/promtail /var/lib/promtail
```

**Verify the adm group can actually read the log files:**
```bash
sudo -u promtail cat /var/log/syslog | head -5
sudo -u promtail cat /var/log/auth.log | head -5
```

If either command returns `permission denied`, the `adm` group membership did not work. This is the most common silent failure in Promtail setups — it starts, reports healthy, and ships nothing.

---

### Step 4.2 — Write Promtail Configuration

**Manual — on client01:**
```bash
cat > /etc/promtail/promtail.yaml << 'EOF'
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
      - targets:
          - localhost
        labels:
          job: system
          hostname: client01
          env: lab
          service: system
          log_type: system
          __path__: /var/log/syslog

  - job_name: auth
    static_configs:
      - targets:
          - localhost
        labels:
          job: auth
          hostname: client01
          env: lab
          service: ssh
          log_type: auth
          __path__: /var/log/auth.log
EOF

chown promtail:promtail /etc/promtail/promtail.yaml
chmod 640 /etc/promtail/promtail.yaml
```

**Verify the config syntax before starting the service:**
```bash
sudo -u promtail promtail -config.file=/etc/promtail/promtail.yaml -dry-run
# Expected: exits with no errors
```

---

### Step 4.3 — Create and Start Promtail Service

**Manual — on client01:**
```bash
cat > /etc/systemd/system/promtail.service << 'EOF'
[Unit]
Description=Promtail Log Shipping Agent
After=network.target
Documentation=https://grafana.com/docs/loki/latest/clients/promtail/

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

systemctl daemon-reload
systemctl enable promtail
systemctl start promtail
```

**Verification — step by step:**
```bash
# Service running
systemctl status promtail

# No errors — specifically look for "connection refused" or "permission denied"
journalctl -u promtail -n 50 --no-pager

# Promtail's own HTTP endpoint (confirms it is alive)
curl -s http://localhost:9080/metrics | grep promtail_sent_bytes_total
# Must return a line with the metric — if it returns nothing, Promtail is not running

# Positions file was created (confirms it is reading files)
cat /var/lib/promtail/positions.yaml
# Must show entries for /var/log/syslog and /var/log/auth.log
```

---

### Step 4.4 — Verify Logs Appear in Grafana

In Grafana → Explore:
```logql
{hostname="client01"}
```

You should see log lines within 30 seconds. If nothing appears after 60 seconds:

1. Check Promtail logs: `journalctl -u promtail -f` — look for push errors
2. Check Loki logs: `journalctl -u loki -f` — look for ingestion errors
3. Check Promtail positions file — if it shows a large offset number, Promtail is reading from the end of old files. Delete the positions file and restart to reset: `rm /var/lib/promtail/positions.yaml && systemctl restart promtail`
4. Verify connectivity again: from client01, `curl -s http://192.168.1.10:3100/ready` must return `ready`

---

### Step 4.5 — Repeat for client02 and client03

Once client01 is working perfectly, repeat Steps 4.1–4.4 on client02 and client03. The only difference in each config is the `hostname` label.

**On client02** — change the hostname label in the config:
```bash
# After installing, change hostname in config
sed -i 's/hostname: client01/hostname: client02/' /etc/promtail/promtail.yaml
```

**On client03:**
```bash
sed -i 's/hostname: client01/hostname: client03/' /etc/promtail/promtail.yaml
```

**Verification — all three clients visible in Grafana:**
```logql
{env="lab"}
```

The label browser (left side in Explore) must show `client01`, `client02`, and `client03` under `hostname`.

---

### Step 4.6 — Write the Ansible Role

Now that you have done it manually and know every command works, write the Ansible role. The role does exactly what you did above — nothing more.

`ansible/roles/logging-agent/tasks/main.yml`:
```yaml
---
- name: Create promtail group
  group:
    name: promtail
    system: yes

- name: Create promtail user
  user:
    name: promtail
    system: yes
    shell: /bin/false
    create_home: no
    groups: promtail,adm
    append: yes

- name: Create promtail directories
  file:
    path: "{{ item }}"
    state: directory
    owner: promtail
    group: promtail
    mode: '0750'
  loop:
    - /etc/promtail
    - /var/lib/promtail

- name: Download promtail archive
  get_url:
    url: "https://github.com/grafana/loki/releases/download/v{{ promtail_version }}/promtail-linux-amd64.zip"
    dest: /tmp/promtail-{{ promtail_version }}.zip
    mode: '0644'

- name: Extract promtail binary
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
    src: promtail.yaml.j2
    dest: /etc/promtail/promtail.yaml
    owner: promtail
    group: promtail
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

`ansible/roles/logging-agent/handlers/main.yml`:
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

`ansible/roles/logging-agent/templates/promtail.yaml.j2`:
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

`ansible/roles/logging-agent/templates/promtail.service.j2`:
```ini
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
```

**Test the Ansible role (idempotency test):**
```bash
# First run — should show changes
ansible-playbook -i ansible/inventory/hosts.ini ansible/deploy-clients.yml

# Second run — must show zero changes
ansible-playbook -i ansible/inventory/hosts.ini ansible/deploy-clients.yml
# All tasks should show "ok", none should show "changed"
```

---

### Day 4 Validation Gate

- [ ] `systemctl status promtail` shows `active (running)` on all three clients
- [ ] `journalctl -u promtail -n 20` shows no errors on any client
- [ ] Grafana Explore: `{hostname="client01"}` returns results
- [ ] Grafana Explore: `{hostname="client02"}` returns results
- [ ] Grafana Explore: `{hostname="client03"}` returns results
- [ ] All timestamps in Grafana match current time (NTP working)
- [ ] Ansible playbook runs twice with zero changes on second run

### Known Failure Modes

**Promtail starts and shows healthy but ships nothing:** The `promtail` user is not in the `adm` group. Verify with `id promtail`. If `adm` is missing: `usermod -aG adm promtail && systemctl restart promtail`.

**Logs appear once then stop:** Promtail positions file corruption. Delete `/var/lib/promtail/positions.yaml` and restart Promtail. It will re-send historical logs — this causes duplicates in Loki but is harmless.

**Loki rejects logs with "out of order" error:** System clock on the client is wrong. Run `timedatectl` — if NTP is not synchronized, fix it first. Logs with timestamps in the future will be rejected when they arrive "late."

**`__path__` does not become a queryable label.** It is consumed internally by Promtail. Do not attempt `{__path__="/var/log/syslog"}` in Grafana — it will return nothing.

---

## Day 5 — SSH Authentication and Sudo Event Collection

### Goal
Failed SSH logins, successful logins, and sudo commands are reliably searchable in Grafana within 60 seconds of occurring.

---

### Step 5.1 — Verify auth.log Exists and Is Being Written

**Manual — on each client:**
```bash
ls -la /var/log/auth.log
# Must exist and have a recent modification time

# Generate a test auth event and watch for it
sudo ls /root
tail -5 /var/log/auth.log
# Must show the sudo event you just generated
```

**If `/var/log/auth.log` does not exist:**
```bash
# Check if rsyslog is installed and running
systemctl status rsyslog

# If not installed
apt-get install -y rsyslog
systemctl enable --now rsyslog

# Verify auth logging config
grep -r "auth" /etc/rsyslog.conf /etc/rsyslog.d/ 2>/dev/null
# Must show: auth,authpriv.*   /var/log/auth.log
```

On Ubuntu 22.04+ with only `systemd-journald` and no rsyslog, `auth.log` may not exist. Install rsyslog or configure Promtail to read from journald directly (different config, not covered in this plan).

---

### Step 5.2 — Generate Test Events and Manually Verify in the Log File

**Before configuring anything, verify the events you want to capture are actually appearing in auth.log.**

```bash
# On client01 — generate events
sudo ls /root        # Sudo event
logger "test auth event"  # Manual syslog entry

# From your workstation — generate a failed SSH (replace with your client IP)
ssh nonexistent_user@192.168.1.11
# Enter wrong password or just let it fail

# Back on client01 — verify events in the file
grep "sudo" /var/log/auth.log | tail -5
grep "Failed\|Invalid\|Accepted" /var/log/auth.log | tail -5
```

You must see these events in the file before expecting Promtail to ship them. If the events are not in the file, no amount of Promtail configuration will capture them.

---

### Step 5.3 — Verify Events Appear in Grafana (Auth Log Already Collected)

The `auth.log` scrape job was already added in Day 4. Run these queries in Grafana Explore to verify:

```logql
# Must return results if anyone has done sudo on client01
{log_type="auth", hostname="client01"} |= "sudo"

# Must return results if any failed SSH attempt was made
{log_type="auth"} |= "Failed password"

# Must return results for the test sudo you just ran
{log_type="auth"} |= "COMMAND"
```

If these return no results, the issue is in the auth log collection from Day 4, not Day 5 configuration. Debug Day 4 before proceeding.

---

### Step 5.4 — Verify SSH Logging Configuration

**Manual — on each client:**
```bash
grep "SyslogFacility" /etc/ssh/sshd_config
# Must return: SyslogFacility AUTH
# If it returns DAEMON or is absent, SSH logs go to a different file

grep "LogLevel" /etc/ssh/sshd_config
# Must return INFO or VERBOSE
# QUIET suppresses most useful auth log entries
```

If `SyslogFacility` is not `AUTH`:
```bash
# Edit sshd_config
sed -i 's/^#SyslogFacility.*/SyslogFacility AUTH/' /etc/ssh/sshd_config
sed -i 's/^SyslogFacility.*/SyslogFacility AUTH/' /etc/ssh/sshd_config
systemctl reload sshd

# Generate a test event and verify it appears in auth.log
ssh wronguser@localhost
tail -10 /var/log/auth.log
```

---

### Step 5.5 — Useful Verification Queries

After generating test events, run these in Grafana Explore to confirm capture:

```logql
# Failed SSH logins
{log_type="auth"} |= "Failed password"

# Successful SSH logins
{log_type="auth"} |= "Accepted publickey"

# All sudo commands
{log_type="auth"} |= "COMMAND"

# Failed sudo (wrong password)
{log_type="auth"} |= "authentication failure" |= "sudo"

# Auth events from a specific host in last hour
{log_type="auth", hostname="client01"}
```

**Generate events and confirm they appear within 60 seconds:**
```bash
# Terminal 1 — watch Grafana with live tail (in Explore, enable "Live" mode)

# Terminal 2 — generate events
ssh wronguser@192.168.1.11    # Failed login
sudo useradd testverify2025   # User creation (also generates audit event later)
sudo userdel testverify2025
```

---

### Day 5 Validation Gate

- [ ] Failed SSH login appears in Grafana (`{log_type="auth"} |= "Failed password"`) within 60 seconds
- [ ] Sudo command appears in Grafana (`{log_type="auth"} |= "COMMAND"`) within 60 seconds
- [ ] Correct `hostname` label on each event
- [ ] `/var/log/auth.log` is written by rsyslog (not missing)
- [ ] `SyslogFacility AUTH` confirmed in `/etc/ssh/sshd_config` on all clients

---

## Day 6 — Auditd Integration

### Goal
Kernel-level audit events — user creation, passwd changes, sudoers modifications, sensitive file access — visible in Grafana with correct `audit_key` labels.

---

### Step 6.1 — Install Auditd Manually on One Client First

**Manual — on client01:**
```bash
apt-get install -y auditd audispd-plugins

systemctl enable auditd
systemctl start auditd
systemctl status auditd

# Verify the audit log is being written
ls -la /var/log/audit/audit.log
tail -5 /var/log/audit/audit.log
```

---

### Step 6.2 — Write and Load Audit Rules Manually

**Manual — on client01:**
```bash
cat > /etc/audit/rules.d/logging-platform.rules << 'EOF'
# Delete existing rules first
-D

# Buffer size — prevents dropped events under load
-b 8192

# Failure mode: 1 = silent (correct for production)
-f 1

# User and group file monitoring
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

# Systemd service file changes
-w /etc/systemd/system/ -p wa -k service_modification

# SSH configuration changes
-w /etc/ssh/sshd_config -p wa -k ssh_config

# Cron changes
-w /etc/cron.d/ -p wa -k cron_modification
-w /var/spool/cron/ -p wa -k cron_modification

# Privileged command execution
-a always,exit -F path=/usr/bin/sudo -F perm=x -F auid>=1000 -F auid!=-1 -k priv_cmd
-a always,exit -F path=/usr/bin/su -F perm=x -F auid>=1000 -F auid!=-1 -k priv_cmd

# NOTE: Do NOT add -e 2 (immutable) until rules are finalized and tested
EOF

# Load the rules
augenrules --load

# Verify rules are active
auditctl -l
# Must show all the rules you wrote above
```

**Generate a test event and verify it appears in audit.log:**
```bash
sudo useradd audit_test_user
sudo userdel audit_test_user

# Look for the events in audit.log
grep "user_modification\|user_mgmt" /var/log/audit/audit.log | tail -10
```

You must see the audit records before proceeding. The events must appear in `/var/log/audit/audit.log` first — then Promtail can ship them.

---

### Step 6.3 — Configure Auditd to Write to Syslog

This approach routes audit events through syslog, allowing Promtail to read them from `/var/log/syslog` without needing special access to `/var/log/audit/audit.log`.

```bash
# Check the syslog plugin config
cat /etc/audit/plugins.d/syslog.conf

# Enable syslog output
sed -i 's/^active = no/active = yes/' /etc/audit/plugins.d/syslog.conf

# Restart auditd to apply
systemctl restart auditd

# Verify audit events are now appearing in syslog
grep "audit" /var/log/syslog | tail -10
```

**Generate a test event to confirm syslog forwarding:**
```bash
sudo touch /etc/sudoers.d/audit_test_file
sudo rm /etc/sudoers.d/audit_test_file
grep "sudoers_modification" /var/log/syslog | tail -5
# Must show the event with the audit key
```

---

### Step 6.4 — Add Dedicated Audit Log Scrape to Promtail

The syslog approach (Step 6.3) means audit events are already flowing through the system scrape job from Day 4. However, a dedicated scrape job with the `audit_key` label extracted provides much better filtering in Grafana.

**Manual — update Promtail config on client01 (add these scrape_configs):**
```yaml
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
```

**For Promtail to read `/var/log/audit/audit.log`, it needs group access:**
```bash
# Check current permissions
ls -la /var/log/audit/audit.log
# Default: -rw------- root:root — promtail cannot read this

# Option A: Add promtail to audit group (create the group if needed)
groupadd -f audit
chgrp audit /var/log/audit/audit.log
chmod 640 /var/log/audit/audit.log
usermod -aG audit promtail

# Option B (simpler): Use syslog forwarding from Step 6.3 and query syslog
# If using Option B, skip this scrape job and filter audit events in the system job:
# {log_type="system"} |= "type=USER_MGMT" or similar
```

**Verify the permission fix:**
```bash
sudo -u promtail cat /var/log/audit/audit.log | head -5
# Must return content, not "permission denied"
```

**Restart Promtail and verify in Grafana:**
```bash
systemctl restart promtail

# Wait 30 seconds, then in Grafana Explore:
# {log_type="audit", hostname="client01"}
```

---

### Day 6 Validation Gate

- [ ] `auditctl -l` shows all rules loaded on all clients
- [ ] `sudo useradd audit_test_user` generates event visible in Grafana within 60 seconds
- [ ] `sudo touch /etc/sudoers.d/test && sudo rm /etc/sudoers.d/test` generates event with `audit_key="sudoers_modification"`
- [ ] `audit_key` label is extracted and filterable in Grafana label browser
- [ ] `grep "user_modification" /var/log/audit/audit.log` returns events from test

### Known Failure Modes

**`-e 2` (immutable rules) set accidentally:** If this flag is active, no rules can be changed until reboot. Do not use it during setup. Check with `auditctl -s | grep enabled` — value `2` means immutable.

**Promtail silently ships no audit logs:** Permission problem on `/var/log/audit/audit.log`. Always verify with `sudo -u promtail cat /var/log/audit/audit.log` before trusting that Promtail can read it.

**Auditd log rotation filling disk:** Auditd has independent rotation config at `/etc/audit/auditd.conf`. Ensure `max_log_file_action = ROTATE` and `num_logs = 5` are set.

---

## Day 7 — Application Logs (PostgreSQL and Nginx)

### Goal
PostgreSQL query logs and Nginx access/error logs visible in Grafana with correct service labels and filterable by status code (Nginx) and error type (PostgreSQL).

---

### Step 7.1 — PostgreSQL: Enable Logging Manually on client01

**Manual — on client01 (PostgreSQL server):**
```bash
# Find the config file
sudo -u postgres psql -c "SHOW config_file;"

# Edit postgresql.conf
nano /etc/postgresql/*/main/postgresql.conf
```

Find and change these settings (uncomment if commented):
```
log_destination = 'stderr'
logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d.log'
log_rotation_age = 1d
log_rotation_size = 100MB
log_min_duration_statement = 1000
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_connections = on
log_disconnections = on
log_checkpoints = on
log_lock_waits = on
```

```bash
# Reload PostgreSQL (no restart needed for logging changes)
systemctl reload postgresql

# Verify log directory exists and is writable
ls -la /var/log/postgresql/

# Generate a test query and verify it appears in the log
sudo -u postgres psql -c "SELECT pg_sleep(2);"   # Triggers slow query log
ls -la /var/log/postgresql/
tail -20 /var/log/postgresql/postgresql-$(date +%Y-%m-%d).log
# Must show the query with a duration >= 2000ms
```

**Verify Promtail can read the PostgreSQL log directory:**
```bash
ls -la /var/log/postgresql/
# Typically: drwxr-xr-x postgres:postgres
# promtail (in adm group) may not be able to read this

# Fix permissions
chmod 755 /var/log/postgresql
sudo -u promtail cat /var/log/postgresql/postgresql-$(date +%Y-%m-%d).log | head -5
# Must return content
```

---

### Step 7.2 — Add PostgreSQL Scrape Job to Promtail on client01

**Manual — update `/etc/promtail/promtail.yaml` on client01:**
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

```bash
systemctl restart promtail

# Wait 30 seconds, then verify in Grafana:
# {service="postgresql", hostname="client01"}
```

**Generate events and verify in Grafana:**
```bash
# Authentication failure
psql -U nonexistentuser -d postgres 2>/dev/null || true

# Slow query
sudo -u postgres psql -c "SELECT pg_sleep(1.5);"

# In Grafana:
# {service="postgresql"} |= "authentication failed"
# {service="postgresql"} |= "duration"
```

---

### Step 7.3 — Nginx: Configure JSON Log Format on client02

**Manual — on client02 (Nginx server):**
```bash
# Verify Nginx is installed and running
nginx -v
systemctl status nginx

# Edit nginx.conf — add JSON log format
nano /etc/nginx/nginx.conf
```

Add inside the `http {}` block:
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

access_log /var/log/nginx/access.log json_combined flush=1s;
error_log /var/log/nginx/error.log warn;
```

```bash
# Test config before reloading
nginx -t
# Must return: syntax is ok / test is successful

nginx -s reload

# Generate a test request and verify JSON format in log
curl -s http://localhost/ > /dev/null
tail -3 /var/log/nginx/access.log
# Must be a valid JSON line, not the default combined format
```

**Verify Promtail can read Nginx logs:**
```bash
sudo -u promtail cat /var/log/nginx/access.log | head -3
# Must return content — nginx logs are typically readable by www-data group
# If permission denied: chmod 755 /var/log/nginx
```

---

### Step 7.4 — Add Nginx Scrape Jobs to Promtail on client02

**Manual — update `/etc/promtail/promtail.yaml` on client02:**
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
            remote_addr: remote_addr
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

```bash
systemctl restart promtail

# Generate requests to create log entries
curl -s http://localhost/ > /dev/null
curl -s http://localhost/doesnotexist > /dev/null    # 404
curl -s http://localhost/another404 > /dev/null

# Verify in Grafana (wait 30 seconds)
# {service="nginx", hostname="client02"}
# {service="nginx"} | json | status >= 400
```

---

### Day 7 Validation Gate

- [ ] `{service="postgresql"}` returns log lines in Grafana
- [ ] `{service="postgresql"} |= "authentication failed"` returns results after test
- [ ] `{service="nginx"}` returns log lines in Grafana
- [ ] `{service="nginx"} | json | status >= 400` returns the 404 you generated
- [ ] Slow query (>1s) visible with `{service="postgresql"} |= "duration"`
- [ ] PostgreSQL and Nginx log files are readable by the `promtail` user (verified with `sudo -u promtail cat`)

---

## Day 8 — Dashboards

### Goal
Four operational dashboards provisioned as files (not built via UI), loading without errors, with hostname dropdown filtering working.

---

### Step 8.1 — Set Up Dashboard Provisioning

**Manual — on logging server:**
```bash
# Create provisioning config
cat > /etc/grafana/provisioning/dashboards/default.yaml << 'EOF'
apiVersion: 1
providers:
  - name: default
    folder: Logging Platform
    type: file
    options:
      path: /var/lib/grafana/dashboards
      updateIntervalSeconds: 30
EOF

# Create dashboards directory
mkdir -p /var/lib/grafana/dashboards
chown grafana:grafana /var/lib/grafana/dashboards
```

---

### Step 8.2 — Verify Queries Before Building Dashboards

Every panel query must be tested in Grafana Explore first. Do not build a panel around a query you have not verified returns data.

Run these in Explore and confirm they return results:

```logql
# Dashboard 1 — Auth
{log_type="auth"} |= "Failed password"
{log_type="auth"} |= "COMMAND"
count_over_time({log_type="auth"} |= "Failed password" [24h])

# Dashboard 2 — Audit
{log_type="audit"} |= "user_modification"
{log_type="audit"} |= "sudoers_modification"
count_over_time({log_type="audit"} [24h])

# Dashboard 3 — PostgreSQL
{service="postgresql"} |= "ERROR"
{service="postgresql"} |= "authentication failed"
{service="postgresql"} |= "duration"

# Dashboard 4 — Nginx
{service="nginx"} | json | status >= 400
count_over_time({service="nginx", log_type="web"} [1h])
```

Generate test events for any query returning empty results before building panels.

---

### Step 8.3 — Dashboard Panel Reference

Build each dashboard in the Grafana UI first, then export as JSON to `/var/lib/grafana/dashboards/`. This gives you the provisioned file for future deployments.

**Dashboard 1 — SSH & Authentication:**

| Panel | Type | Query |
|---|---|---|
| Failed logins (24h) | Stat | `count_over_time({log_type="auth"} \|= "Failed password" [24h])` |
| Failed logins over time | Time series | `sum(count_over_time({log_type="auth"} \|= "Failed password" [$__interval]))` |
| Failed login events | Logs | `{log_type="auth"} \|= "Failed password"` |
| Sudo commands | Logs | `{log_type="auth"} \|= "COMMAND"` |
| Auth events by host | Table | `sum by (hostname) (count_over_time({log_type="auth"} [24h]))` |

**Dashboard 2 — Audit Events:**

| Panel | Type | Query |
|---|---|---|
| Audit events (24h) | Stat | `count_over_time({log_type="audit"} [24h])` |
| User modification events | Logs | `{log_type="audit"} \|= "user_modification"` |
| Sudoers changes | Logs | `{log_type="audit"} \|= "sudoers_modification"` |
| All audit events | Logs | `{log_type="audit"}` |

**Dashboard 3 — PostgreSQL:**

| Panel | Type | Query |
|---|---|---|
| DB errors (24h) | Stat | `count_over_time({service="postgresql"} \|= "ERROR" [24h])` |
| Authentication failures | Logs | `{service="postgresql"} \|= "authentication failed"` |
| Slow queries (>1s) | Logs | `{service="postgresql"} \|= "duration"` |
| All DB errors | Logs | `{service="postgresql"} \|= "ERROR"` |

**Dashboard 4 — Nginx Web:**

| Panel | Type | Query |
|---|---|---|
| Request rate | Time series | `count_over_time({service="nginx", log_type="web"} [$__interval])` |
| 4xx errors (1h) | Stat | `count_over_time({service="nginx"} \| json \| status >= 400 < 500 [1h])` |
| 5xx errors (1h) | Stat | `count_over_time({service="nginx"} \| json \| status >= 500 [1h])` |
| Error log | Logs | `{service="nginx", job="nginx-error"}` |
| 4xx/5xx requests | Logs | `{service="nginx"} \| json \| status >= 400` |

**Add these template variables to every dashboard:**

| Variable | Type | Query |
|---|---|---|
| `hostname` | Query | `label_values(hostname)` |
| `env` | Query | `label_values(env)` |
| `service` | Query | `label_values(service)` |

Use `$hostname` in all queries to make dashboards filterable by node.

---

### Step 8.4 — Export Dashboards as Provisioning Files

After building each dashboard in the UI:

1. Open the dashboard
2. Click the share icon → Export → Save to file
3. Copy the downloaded JSON to `/var/lib/grafana/dashboards/dashboard-name.json`
4. Restart Grafana: `systemctl restart grafana-server`
5. The dashboard now appears under the "Logging Platform" folder and will survive Grafana reinstalls

---

### Day 8 Validation Gate

- [ ] All 4 dashboards load without errors or empty panels
- [ ] Hostname dropdown filters data across all dashboards
- [ ] Stat panels show non-zero counts
- [ ] Logs panels show raw log lines with correct timestamps
- [ ] Dashboard JSON files exist in `/var/lib/grafana/dashboards/`

---

## Day 9 — Ansible Automation

### Goal
A single `ansible-playbook site.yml` command deploys everything on a fresh node. Idempotent — runs twice, second run shows zero changes.

---

### Step 9.1 — Final Ansible Role Structure

All roles encode what you have already done manually. No role should contain a command you have not verified by hand.

```
ansible/
├── inventory/
│   └── hosts.ini
├── group_vars/
│   ├── all.yml
│   └── clients.yml
├── roles/
│   ├── common/
│   │   └── tasks/main.yml
│   ├── chrony/
│   │   ├── tasks/main.yml
│   │   └── templates/chrony.conf.j2
│   ├── auditd/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   └── templates/audit.rules.j2
│   └── logging-agent/
│       ├── tasks/main.yml
│       ├── handlers/main.yml
│       └── templates/
│           ├── promtail.yaml.j2
│           └── promtail.service.j2
├── deploy-logging-server.yml
├── deploy-clients.yml
└── site.yml
```

`group_vars/all.yml`:
```yaml
promtail_version: "2.9.8"
loki_version: "2.9.8"
loki_url: "192.168.1.10:3100"
env: "lab"
```

`site.yml`:
```yaml
---
- import_playbook: deploy-logging-server.yml
- import_playbook: deploy-clients.yml
```

`deploy-clients.yml`:
```yaml
---
- name: Deploy logging agents to all clients
  hosts: clients
  become: yes
  roles:
    - common
    - chrony
    - auditd
    - logging-agent
```

---

### Step 9.2 — Idempotency Test

This is the only test that matters for Ansible:

```bash
# Run once — expect changes
ansible-playbook -i ansible/inventory/hosts.ini ansible/site.yml

# Run again immediately — must show zero changes
ansible-playbook -i ansible/inventory/hosts.ini ansible/site.yml
# All tasks: ok=N  changed=0  failed=0
```

Any `changed` on the second run means the role is not idempotent. Find the task that always reports changed and fix it. Common causes:

- Using `shell:` instead of a proper Ansible module
- `copy:` instead of `template:` with a file that has dynamic content
- `lineinfile:` with a pattern that matches multiple lines

---

### Step 9.3 — Onboarding a New Client

```bash
# 1. Add to inventory
echo "client04 ansible_host=192.168.1.14" >> ansible/inventory/hosts.ini

# 2. Ensure the ansible user exists on the new node (manual step — do this first)
# ssh to the new node and create the user as in Step 1.3

# 3. Deploy everything to the new node only
ansible-playbook -i ansible/inventory/hosts.ini ansible/deploy-clients.yml -l client04

# 4. Verify logs appear in Grafana
# {hostname="client04"} should return results within 5 minutes
```

---

### Day 9 Validation Gate

- [ ] `ansible-playbook site.yml` completes with zero failures
- [ ] Second run of `site.yml` shows `changed=0` for all hosts
- [ ] Adding a new client to inventory and running `deploy-clients.yml -l new-host` deploys fully within 10 minutes
- [ ] New client logs appear in Grafana within 5 minutes of deployment

---

## Day 10 — Validation and Hardening

### Goal
Generate every event type and confirm it appears in Grafana. Close the minimum security gaps required before calling this production-adjacent.

---

### Step 10.1 — Full Event Generation Matrix

Run every command and verify the result in Grafana before checking it off.

| Event | Command to Run | Expected Grafana Query | Max Latency |
|---|---|---|---|
| Failed SSH login | `ssh baduser@client01` from workstation | `{log_type="auth"} \|= "Failed password"` | 60s |
| Successful SSH login | `ssh ansible@client01` from workstation | `{log_type="auth"} \|= "Accepted"` | 60s |
| Sudo execution | `sudo ls /root` on client01 | `{log_type="auth"} \|= "COMMAND"` | 60s |
| User creation | `sudo useradd finaltest` on client01 | `{log_type="audit"} \|= "user_modification"` | 60s |
| Password change | `sudo passwd finaltest` on client01 | `{log_type="audit"} \|= "user_modification"` | 60s |
| Sudoers change | `sudo touch /etc/sudoers.d/test && sudo rm /etc/sudoers.d/test` | `{log_type="audit"} \|= "sudoers_modification"` | 60s |
| PostgreSQL auth fail | `psql -U nobody -d postgres` on client01 | `{service="postgresql"} \|= "authentication failed"` | 60s |
| PostgreSQL slow query | `sudo -u postgres psql -c "SELECT pg_sleep(1.5);"` | `{service="postgresql"} \|= "duration"` | 60s |
| Nginx 404 | `curl http://client02/doesnotexist` | `{service="nginx"} \| json \| status == 404` | 60s |
| Nginx 200 | `curl http://client02/` | `{service="nginx"} \| json \| status == 200` | 60s |
| System log | `logger "test message day10"` on client03 | `{log_type="system"} \|= "test message day10"` | 60s |

Clean up test artifacts after validation:
```bash
sudo userdel finaltest
sudo rm -f /etc/sudoers.d/test
```

---

### Step 10.2 — Hardening Checklist

**Loki:**
- [ ] Process running as `loki` user, not root: `ps aux | grep loki`
- [ ] Log data on separate volume, not root disk: `df -h /var/lib/loki`
- [ ] Retention configured and confirmed (`retention_period: 30d` in `loki.yaml`)
- [ ] Port 3100 not exposed to the internet (verify at cloud firewall level, not just ufw)
- [ ] Disk usage monitored — add cron job: `echo "0 * * * * root df /var/lib/loki | awk 'NR==2 {if(\$5+0>80) print \"Loki disk above 80%:\", \$0}'" >> /etc/crontab`

**Grafana:**
- [ ] Default `admin` password changed (verify you cannot log in with `admin`/`admin`)
- [ ] `allow_sign_up = false` confirmed: `grep allow_sign_up /etc/grafana/grafana.ini`
- [ ] Dashboard JSON files saved in `/var/lib/grafana/dashboards/` (survive reinstalls)
- [ ] No admin credentials shared — create viewer accounts for read-only users

**Promtail:**
- [ ] Running as `promtail` user on all clients: `ps aux | grep promtail`
- [ ] Config file not world-readable: `ls -la /etc/promtail/promtail.yaml` — mode must be `640`
- [ ] Positions file on persistent volume, not tmpfs: `df /var/lib/promtail`

**Auditd:**
- [ ] Rules loaded on all clients: `auditctl -l` on each client
- [ ] Log rotation configured: `grep "max_log_file_action\|num_logs" /etc/audit/auditd.conf`
- [ ] `-e 2` (immutable) NOT set — confirm: `auditctl -s | grep "enabled 2"` must return nothing

**Ansible:**
- [ ] No plaintext passwords in `group_vars/` — check with `grep -r "password" ansible/group_vars/`
- [ ] Ansible Vault used for any secrets (API keys, credentials)

---

### Step 10.3 — Performance Baselines

Record these values now. These are your reference points for detecting problems later.

```bash
# Loki disk usage
du -sh /var/lib/loki/

# Loki active streams (higher than 1000 is a cardinality warning sign)
curl -s http://localhost:3100/loki/api/v1/labels | python3 -m json.tool

# Promtail memory footprint on each client
ps -o pid,rss,comm -p $(pgrep promtail)
# RSS in KB — expect 30–80MB. If >200MB, check for label cardinality issues

# Auditd buffer — if num_lost > 0, increase buffer size in audit.rules (-b value)
auditctl -s | grep "backlog\|lost"

# Grafana database size (SQLite)
ls -lh /var/lib/grafana/grafana.db
```

Write these values in a comment below and re-check weekly:

```
Baseline recorded: [DATE]
Loki disk usage: ___
Promtail RSS (client01): ___
Promtail RSS (client02): ___
Promtail RSS (client03): ___
```

---

### Step 10.4 — Documented Gaps for Phase 2

These gaps exist after this 10-day build. They are known, documented, and accepted for a lab/initial deployment. They must be addressed before handling sensitive or regulated data.

| Gap | Risk | Phase 2 Fix |
|---|---|---|
| No alerting | SSH brute force generates no notification | Grafana alerting rules on `count_over_time` thresholds |
| Plaintext log transport | Log contents exposed on the network | TLS between Promtail and Loki (Loki TLS config + Promtail `tls_config`) |
| Unauthenticated Loki push endpoint | Any internal host can push fake logs | nginx reverse proxy with basic auth in front of Loki :3100 |
| Grafana on HTTP | Session tokens exposed on network | nginx with Let's Encrypt or internal CA cert |
| SQLite Grafana backend | Not suitable for HA, occasional corruption under load | PostgreSQL backend for Grafana |
| No log integrity | Logs can be deleted or modified after ingestion | Object storage backend with versioning (S3/MinIO) |
| No backup | Loki data and Grafana dashboards lost if disk fails | Daily backup of `/var/lib/loki` and `/var/lib/grafana` |
| No centralized auditd buffer | High log volume can drop audit events | auditd `-b` buffer tuning and monitoring `num_lost` |

---

## Critical LogQL Reference

```logql
# All logs from a specific host
{hostname="client01"}

# Failed SSH logins
{log_type="auth"} |= "Failed password"

# Successful SSH logins
{log_type="auth"} |= "Accepted publickey"

# All sudo commands
{log_type="auth"} |= "COMMAND"

# Audit events by key
{log_type="audit"} |= "user_modification"
{log_type="audit", audit_key="sudoers_modification"}

# PostgreSQL errors
{service="postgresql"} |= "ERROR"

# PostgreSQL auth failures
{service="postgresql"} |= "authentication failed"

# Nginx 4xx and 5xx
{service="nginx"} | json | status >= 400
{service="nginx"} | json | status >= 500

# All errors across all services
{env="lab"} |= "error"

# Count for stat panels
count_over_time({log_type="auth"} |= "Failed password" [24h])

# Rate for time series panels
sum(rate({log_type="auth"} |= "Failed password" [$__interval]))

# Filter by dashboard variable
{hostname="$hostname", log_type="auth"} |= "Failed password"
```

---

## Ansible Inventory Quick Reference

When adding a new server, these are the only files that need to change:

1. `ansible/inventory/hosts.ini` — add the new host under `[clients]`
2. Ensure the `ansible` user with NOPASSWD sudo exists on the new node (manual prerequisite)
3. Run: `ansible-playbook deploy-clients.yml -l new-hostname`

That is it. All configuration is handled by the roles.

---

## Resume Summary

Designed and deployed a centralized logging and security audit platform across a multi-server Linux environment using Grafana, Loki, Promtail, and auditd. Implemented structured log collection for SSH authentication events, sudo activity, kernel audit events, PostgreSQL query logs, and Nginx access logs across three client nodes. Configured Promtail pipeline stages for label extraction and field parsing. Built operational dashboards in Grafana for authentication monitoring, audit event review, database error tracking, and web traffic analysis. Automated full-stack deployment and new-server onboarding using Ansible roles with idempotent configuration management. Validated every component through manual command verification before automation. Established retention policies, security hardening baseline, and documented production gap remediation roadmap.
