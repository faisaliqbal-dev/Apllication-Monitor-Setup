# Prometheus + Grafana monitoring for Java app on EC2

> README: step-by-step setup to monitor a Java application running on an EC2 instance using Node Exporter, Blackbox Exporter, Prometheus and Grafana. This guide assumes Prometheus + Grafana run on a separate "monitoring" EC2 instance and the Java app runs on an "app" EC2 instance.

---

## Overview (short)

* **App EC2**: runs the Java application + **Node Exporter** (to expose system metrics on port `9100`).
* **Monitoring EC2**: runs **Prometheus** (scrapes metrics), **Blackbox Exporter** (external checks, default `9115`) and **Grafana** (visualization, `3000`).
* Prometheus scrapes Node Exporter on the App EC2 and Blackbox exporter probes the application endpoints.

ASCII flow:

```
[Java App EC2 (app)] -- node_exporter:9100 --> (metrics)
                 \                          
                  \-- http(s):8080 (app)

[Monitoring EC2] -- Prometheus:9090 - scrapes -> app:9100 and blackbox:9115 (probes app endpoint)
                 \-- Grafana:3000 -> reads from Prometheus
```

---

## Prerequisites

* Two EC2 instances (or more): `app` and `monitoring`.
* SSH access to both with sudo.
* Basic ports in AWS Security Groups opened appropriately (see Security section below).
* Java app running on `app` EC2 (example: `http://APP_PRIVATE_IP:8080`).

---

## Ports & Security (recommended)

* Prometheus (monitoring EC2): `9090` (open to your admin IP or internal network)
* Grafana: `3000` (open to trusted IPs only)
* Node Exporter (app EC2): `9100` **should be accessible only from the monitoring EC2** (restrict via SG to monitoring private IP)
* Blackbox Exporter (monitoring EC2): `9115` (localhost or 9115 on monitoring)

**Important:** Do NOT open node exporter port to the public internet. Use private IPs or limit SG access to monitoring server.

---
## 0) Install Java Application and Run that App on EC2
 git clone https://github.com/faisaliqbal-dev/BoardGame-App.git
 ls 
 apt install java 
 apt install maven
 cd BoardGame-App
 mvn package
 ls
 cd target
 java -jar .jar


## 1) Install Node Exporter on App EC2

Run on the **app** EC2 (example commands for Linux):

# download (adjust version if needed)
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
ls
tar -xvf node_exporter-1.9.1.linux-amd64.tar.gz
mv node_exporter-1.9.1.linux-amd64.tar.gz node_exporter
cd node_exporter
./node_exporter 
node exporter will be ruuning on port number 9100

# verify
curl http://ec2ip:9100/metrics


## 2) Install Prometheus on Monitoring EC2

apt update -y

wget https://github.com/prometheus/prometheus/releases/download/v3.5.0/prometheus-3.5.0.linux-amd64.tar.gz
tar -xvf prometheus-3.5.0.linux-amd64.tar.gz
ls
mv prometheus-3.5.0.linux-amd64.tar.gz prometheus

cd prometheus
./prometheus 
it will get running on port 9090

# prometheus.yml will go to /etc/prometheus/prometheus.yml (example below)
```

**Example `prometheus.yml`:**

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'node_exporter_app'
    static_configs:
      - targets: ['<App-EC2-Public-IP>:9100']

  - job_name: 'blackbox_http_app'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - http://<App-EC2-Public-IP>:8080   # java app URL
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115   # blackbox exporter port

```

Save the file to `/prometheus/prometheus.yml` and set ownership:

run prometheus yaml file
./prometheus.yml

Open `http://MONITORING_PRIVATE_IP:9090/targets` in browser and verify both `node_exporter_app` and `blackbox_http` show up and are `UP`.

---

## 3) Install Blackbox Exporter on Monitoring EC2

wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.27.0/blackbox_exporter-0.27.0.linux-amd64.tar.gz
ls
tar -xvf blackbox_exporter-0.27.0.linux-amd64.tar.gz
mv blackbox_exporter-0.27.0.linux-amd64.tar.gz blackbox_exporter
ls
cd blackbox_exporter
./blackbox_exporter


# verify
curl http://monitorec2ip:9115/metrics | head -n 20
```

Blackbox's default config includes modules like `http_2xx`. You can customize `/etc/blackbox.yml` if you need advanced checks.

---

## 4) Install Grafana on Monitoring EC2

sudo apt-get install -y adduser libfontconfig1 musl
wget https://dl.grafana.com/grafana-enterprise/release/12.1.1/grafana-enterprise_12.1.1_16903967602_linux_amd64.deb
sudo dpkg -i grafana-enterprise_12.1.1_16903967602_linux_amd64.deb

after that 3 commands will come and you have to run that commands for running graffana 

Open `http://MONITORING_IP:3000` and login (`admin`/`admin` default). Change the admin password when prompted.

---

## 5) Configure Grafana Data Source & Dashboards

1. In Grafana UI → Configuration → Data Sources → Add data source → choose **Prometheus**.

   * URL: `http://localhost:9090` (when Grafana is on the same monitoring EC2)
   * Save & Test → should succeed.

2. Import Dashboards:

   * **Node Exporter dashboard**: Grafana.com dashboard ID `1860` (you already used this). Use **Import** → enter `1860` → select Prometheus data source.
   * **Blackbox exporter dashboard**: ID `7587` (you already used this). Import similarly.

3. If metrics are visible in the dashboards, Grafana is reading Prometheus successfully.

---

## 6) Verify end-to-end

* Prometheus targets: `http://MONITORING_IP:9090/targets`
* Node exporter metrics: `http://APP_PRIVATE_IP:9100/metrics`
* Blackbox probe example (from monitoring host):

  * `curl "http://localhost:9115/probe?target=http://APP_PRIVATE_IP:8080&module=http_2xx"`
* Grafana: open dashboards and check panels.

---

## 7) Example alert rule (optional)

Create `/etc/prometheus/alert.rules.yml`:

```yaml
groups:
- name: instance-down
  rules:
  - alert: NodeExporterDown
    expr: up{job="node_exporter_app"} == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Node exporter DOWN (instance={{ $labels.instance }})"
      description: "No scrapes for node_exporter on {{ $labels.instance }} for > 2m"
```

Add `rule_files: - "/etc/prometheus/alert.rules.yml"` under `global` or top-level in `prometheus.yml`. To use alerts, also install Alertmanager and configure Prometheus `alerting` section.

---

## 8) Common troubleshooting

* **Targets show DOWN in Prometheus**: check SG rules (monitoring can reach app on 9100) and node\_exporter process on app host.
* **No metrics in Grafana**: verify Prometheus UI (query `up{job="node_exporter_app"}`) to ensure Prometheus scraped metrics.
* **Blackbox probe returns non-200**: run curl from monitoring machine to the target and ensure app health endpoint works.
* **prometheus.yml syntax errors**: run `/usr/local/bin/promtool check config /etc/prometheus/prometheus.yml` to validate.
* **systemd service failing**: `journalctl -u <service> -f` and `systemctl status <service>`.

---

## 9) Security & production notes

* Use private subnets and private IPs for monitoring traffic when possible.
* Restrict Node Exporter (9100) access to only your monitoring server via AWS Security Groups.
* Consider putting Grafana behind an nginx reverse proxy with TLS and authenticate using SSO or strong credentials.
* For long-term storage or large setups consider remote storage options for Prometheus.

---

## 10) Next steps / automation

* If you want, I can provide:

  * systemd unit files (ready-to-copy) for each component, or
  * a bash script that installs & configures Node Exporter/Prometheus/Blackbox/Grafana automatically, or
  * Terraform templates to create EC2 + Security Groups + provisioners.

---

## Footer

If anything doesn't work, paste the `prometheus.yml`, output of `curl http://MONITORING_IP:9090/targets`, and `journalctl -u prometheus -n 200` and I will help debug.
