
# 📘 ELK Stack High Availability Deployment Guide (Ansible-Based)

This guide outlines the deployment of a full ELK stack in high availability mode using Ansible automation across multiple nodes.

---

## 🧱 Components

- **Elasticsearch** (HA Cluster with 3 nodes)
- **Kibana** (Runs on all ELK nodes)
- **Logstash** (Runs on all ELK nodes)
- **Filebeat** (Runs on all log-sending nodes)

---

## 🗂️ Directory Structure

```
elk-deployment/
├── 01-elasticsearch.yaml
├── 02-kibana-and-logstash.yaml
├── 03-filebeat.yaml
└── inventory/
    └── myinventory
```

---

## 📁 Inventory File: `inventory/myinventory`

```ini

[filebeat_nodes]
# ✅ Add/remove nodes based on environment
app-node1 ansible_host=172.27.19.201
app-node2 ansible_host=172.27.19.202

[filebeat_nodes:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_rsa
logstash_host=172.27.19.111
filebeat_inputs=["/var/log/*.log", "/var/log/**/*.log", "/var/log/syslog", "/var/log/auth.log"]

[elk]
# 🔧 Change these hostnames and IPs to match the actual ELK server nodes
vivek-elk01-19 ansible_host=172.27.19.111 node_role=primary hostname=vivek-elk01-19
vivek-elk02-19 ansible_host=172.27.19.112 node_role=secondary hostname=vivek-elk02-19
vivek-elk03-19 ansible_host=172.27.19.113 node_role=secondary hostname=vivek-elk03-19

[elk:vars]
# ✅ Change for each deployment
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_rsa
cluster_name=elasticsearch-demo
elk_hosts_entries=
  172.27.19.111  vivek-elk01-19
  172.27.19.112  vivek-elk02-19
  172.27.19.113  vivek-elk03-19


```

---

## ✅ Parameters to Change Per Deployment

| Parameter                       | What to Change For                  |
|--------------------------------|-------------------------------------|
| `ansible_host`                 | Actual IP of the node               |
| `hostname`                     | Unique hostname per node            |
| `cluster_name`                 | Unique per environment              |
| `elk_hosts_entries`            | Must match ELK node IPs + names     |
| `ansible_user`                 | SSH user for environment            |
| `ansible_ssh_private_key_file` | Path to correct SSH key             |
| `logstash_host`                | IP of Logstash listener node        |
| `filebeat_inputs`             | Log paths to ship via Filebeat      |

---

## 🧩 Step-by-Step Deployment

### ✅ Step 1: Install Elasticsearch Cluster

```bash
ansible-playbook -i inventory/myinventory 01-elasticsearch.yaml
```

What this does:
- Sets hostnames, disables swap, and adds `/etc/hosts` entries
- Installs Elasticsearch
- Configures Elasticsearch on all nodes with cluster settings
- Uses `groups['elk'][0]` as the dynamic primary node
- Automatically resets password and distributes token to secondaries

---

### ✅ Step 2: Install Kibana & Logstash

```bash
ansible-playbook -i inventory/myinventory 02-kibana-and-logstash.yaml
```

What this does:
- Installs Kibana and Logstash
- Reads password from the primary node
- Creates Kibana service token using Elasticsearch API
- Configures:
  - `kibana.yml` with all cluster nodes and token
  - `logstash.conf` to receive Beats input and push to Elasticsearch

---

### ✅ Step 3: Install Filebeat

```bash
ansible-playbook -i inventory/myinventory 03-filebeat.yaml
```

What this does:
- Installs Filebeat on `[filebeat_nodes]`
- Configures input using `filestream` to read from paths defined in `filebeat_inputs`
- Sends logs to Logstash on port 5044
- Starts and enables Filebeat service

---

## 🔐 Accessing Kibana

- URL: `http://<elk-node-ip>:5601`
- Username: `elastic`
- Password: Output from playbook OR in `/tmp/elastic_user_password.txt` on primary node

---

## 🧪 Verification Commands

```bash
# Check Elasticsearch cluster health
curl -k -u elastic:<password> https://localhost:9200/_cluster/health?pretty

# Check Logstash is listening
netstat -plnt | grep 5044

# View Filebeat logs
journalctl -u filebeat -f
```

This eliminates the need to hardcode hostnames like `vivek-elk01-19`.

---

## 🛠️ Optional Add-ons

Let me know if you want to:
- Add Kibana dashboards setup
- Secure the stack with custom CA TLS certs
- Configure Slack or Email alerting
- Convert this into reusable roles and CI/CD pipelines

---
