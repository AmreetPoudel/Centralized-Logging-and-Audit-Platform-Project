# Centralized Logging & Audit Platform Project

## Goal

Build a centralized logging platform capable of:

* Collecting logs from multiple Linux servers
* Collecting audit events using auditd
* Collecting application logs from custom locations
* Centralized search and filtering
* Dashboard visualization
* Automated deployment using Ansible
* Basic security monitoring and alerting

---

# Success Criteria

By the end of the project I should be able to:

* Search logs from all servers in one place
* Identify failed SSH logins
* Identify sudo activity
* Identify auditd events
* Search PostgreSQL logs
* Search Nginx logs
* Filter logs by hostname
* Filter logs by service
* Filter logs by environment
* Automatically onboard new servers using Ansible

---

# Architecture

```text
+-------------+
| Server 01   |
| PostgreSQL  |
+-------------+
        |
+-------------+
| Server 02   |
| Nginx       |
+-------------+
        |
+-------------+
| Server 03   |
| Application |
+-------------+
        |
        v

+-------------------+
| Central Logging   |
| Loki              |
| Grafana           |
+-------------------+
```

---

# Technology Stack

## Client Side

* auditd
* rsyslog or Promtail
* chrony (NTP)

## Central Server

* Grafana
* Loki
* Linux Server

## Automation

* Ansible

---

# Week 1

## Day 1

### Prepare Lab Environment

Tasks:

* Create logging server
* Create test client 1
* Create test client 2
* Configure network connectivity

Deliverables:

* All servers reachable
* Hostnames configured

---

## Day 2

### Install Loki

Tasks:

* Install Loki
* Configure storage
* Verify service operation

Deliverables:

* Loki running
* Web interface accessible

---

## Day 3

### Install Grafana

Tasks:

* Install Grafana
* Connect Grafana to Loki

Deliverables:

* Grafana dashboard available
* Loki datasource working

---

## Day 4

### Configure Log Shipping

Tasks:

* Install Promtail or rsyslog agent
* Send system logs to Loki

Deliverables:

* Logs visible in Grafana

Verify:

* hostname
* timestamps
* source server

---

## Day 5

### Authentication Logging

Tasks:

Collect:

* auth.log
* secure
* ssh events
* sudo events

Create labels:

* hostname
* environment
* service
* log_type

Deliverables:

* Search failed logins
* Search sudo activity

---

# Week 2

## Day 6

### Auditd Integration

Tasks:

Install auditd

Monitor:

* passwd changes
* user creation
* service modifications
* sensitive file changes

Deliverables:

* Audit events visible in Grafana

---

## Day 7

### Application Logging

Tasks:

Collect:

* PostgreSQL logs
* Nginx logs
* Custom application logs

Deliverables:

* Separate filtering per application

---

## Day 8

### Dashboard Development

Create dashboards:

#### SSH Dashboard

* Login attempts
* Failed logins
* Top source IPs

#### Audit Dashboard

* User activity
* File modifications

#### Database Dashboard

* Errors
* Authentication failures

#### Web Dashboard

* Access logs
* Error logs

---

## Day 9

### Ansible Automation

Create roles:

```text
ansible/
├── auditd
├── logging-agent
├── chrony
├── common
└── grafana-agent
```

Tasks:

* Deploy logging automatically
* Deploy auditd automatically
* Configure labels automatically

Deliverables:

* One command deployment

---

## Day 10

### Validation & Security Testing

Generate events:

* Failed SSH login
* Successful SSH login
* Sudo execution
* PostgreSQL login failure
* Nginx errors
* File modification events

Verify:

* Events appear centrally
* Correct labels attached
* Search works properly

---

# Future Improvements

## Phase 2

* Alerting
* Email notifications
* Slack notifications

Examples:

* auditd stopped
* logging agent stopped
* repeated SSH failures
* root login detected

---

## Phase 3

* OpenSearch integration
* Long-term retention
* Immutable log archive
* Multi-environment logging

---

# Resume Entry

Built a centralized logging and audit platform using Grafana, Loki, auditd, rsyslog, and Ansible. Implemented centralized collection of Linux system logs, authentication events, database logs, web server logs, and security audit events across multiple hosts. Automated deployment and configuration using Ansible, enabling standardized observability, incident investigation, and security monitoring.
