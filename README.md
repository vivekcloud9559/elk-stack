
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

## ELK Stack Deployment using Ansible

This repository contains Ansible playbooks to automate the high-availability deployment of the ELK (Elasticsearch, Logstash, Kibana) stack across multiple nodes.

### Playbooks Overview

- `01-elastic.yaml` – Installs and configures Elasticsearch in a multi-node, high-availability setup.
- `02-kibana-and-logstash.yaml` – Installs and configures Kibana and Logstash on the same ELK nodes.
- `03-filebeat.yaml` – Installs Filebeat on designated log collection nodes and ships logs to Logstash.

### Important Notes

- The `elastic` superuser password is **automatically reset** in the `01-elastic.yaml` playbook.
  - This happens in the task:  
    **Display elastic user password (primary only)**  
    which is only executed on the primary node.
  - The password is shown in the terminal using a debug message:
    ```yaml
    - name: Display elastic user password (primary only)
      when: node_role == 'primary'
      debug:
        msg: "Elastic user password is: {{ reset_pass_output.stdout }}"
    ```
  - ⚠️ **Make sure to capture and save this password**, as it is required for authentication in Kibana and API access and check the output of playbook for the password of elasticsearch.

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

### ✅ Step 4: Install Addional config for 3 node 

```bash
ansible-playbook -i inventory/myinventory 01-02-elastic-final-config.yaml
```

---

### ✅ Step 5: Install Filebeat

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
