# Grafana and Prometheus Setup
 #### Prerequisites:

Install Docker and Docker Compose on the server.
### Step 1: Create the Project Directory
Create a directory named **Grafana** inside **/opt**:
```bash
sudo mkdir -p /opt/Grafana
cd /opt/Grafana
```
### Step 2: Create the Docker Compose File

Create a file named **"docker-compose.yml"** inside the **/opt/Grafana** directory and paste the following content:

```bash
services:
  prometheus:
	image: prom/prometheus:latest
	restart: always
	logging:
  	driver: "json-file"
  	options:
     	max-size: "10m"
     	max-file: "3"
	container_name: prometheus
	volumes:
  	- prometheus_data:/prometheus
  	- ./prometheus.yml:/etc/prometheus/prometheus.yml
	ports:
  	- "9090:9090"
	command:
  	- "--config.file=/etc/prometheus/prometheus.yml"
 
  grafana:
	image: grafana/grafana:latest
	restart: always
	logging:
  	driver: "json-file"
  	options:
     	max-size: "10m"
     	max-file: "3"
	container_name: grafana
	environment:
  	- GF_SECURITY_ADMIN_PASSWORD=admin
	volumes:
  	- grafana_data:/var/lib/grafana
	ports:
  	- "3000:3000"
 
  # pushgateway:
	# image: prom/pushgateway:latest
	# restart: always
	# logging:
  # 	driver: "json-file"
  # 	options:
  #    	max-size: "10m"
  #    	max-file: "3"
	# container_name: pushgateway
	# ports:
  # 	- "9091:9091"
 
 
volumes:
  prometheus_data:
	driver: local
  grafana_data:
	driver: local
```
### Step 3: Create the Prometheus Configuration File

Create another file named **prometheus.yml** inside the same **/opt/Grafana** directory and paste the following content:

```bash
global:
  scrape_interval: 30s
 
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']       # example:192.168.0.24:9090
                                          # example:192.168.10:9090
 
#   - job_name: 'pushgateway'
#     static_configs:
#       - targets: ['pushgateway:9091']             
```
### Step 4: Start the Containers
From the **/opt/Grafana** directory, run:
```bash
sudo docker compose up -d
```
### Verify Running Containers
```bash
docker ps
```
You should see the following containers running:
**prometheus**
**grafana**

If **Pushgateway** is enabled in the future, you should also see:
**pushgateway**

### step 1: Now install node_exporter from where you wanna collect log : 
Create a folder inside **/opt** named **node_exporter** . 

```bash
sudo mkdir node_exporter
```
now install node exporter for server.

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.0/node_exporter-1.6.0.linux-amd64.tar.gz
sudo tar -xvzf node_exporter-1.6.0.linux-amd64.tar.gz       # unzip the file 
sudo mv node_exporter-1.6.0.linux-amd64/node_exporter /opt/node_exporter/
```
### step 2 : 
**Install as a Service:** Create a systemd service file at `/etc/systemd/system/node_exporter.service`:
```bash
[Unit]
Description=Node Expoter
After=network.target

[Service]
User=root
Group=root
ExecStart=/opt/node_exporter/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
```
### Then reload and enable:
```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
```
Repeat on each Linux server (or automate with configuration management) so every Linux node runs **node_exporter** on port **9100**.

## Grafana Queries (Prometheus / Node Exporter Dashboard)
This document provides ready-to-use PromQL queries for building a **Grafana** monitoring dashboard using **Prometheus** + **Node Exporter**.

Prerequisites

Before using these queries, ensure:

**Prometheus is running**
**Node Exporter is configured**
**Grafana datasource is connected to Prometheus**
Test in **Grafana** Explore:
```bash
up
```
### 1. Disk Utilization Gauge (%)
Query
```bash
100 - (
  node_filesystem_avail_bytes{mountpoint="/",fstype!~"tmpfs|overlay"}
  * 100
  /
  node_filesystem_size_bytes{mountpoint="/",fstype!~"tmpfs|overlay"}
)
```
Unit:
```bash
Percent (0-100)
```
Description:
Shows how much of the root filesystem (/) is used.

### 2. Disk Used (GiB)
Query
```bash
(
  node_filesystem_size_bytes{mountpoint="/",fstype!~"tmpfs|overlay"}
  -
  node_filesystem_avail_bytes{mountpoint="/",fstype!~"tmpfs|overlay"}
) / 1024^3
```
Unit:
```bash
gibibytes (GiB)
```
Description:
Total used disk space.

### 3. Disk Free (GiB)
Query:
```bash
node_filesystem_avail_bytes{mountpoint="/",fstype!~"tmpfs|overlay"}
/ 1024^3
```
Unit:
```bash
GiB
```
Description
Available disk space.

### 4. Disk Total (GiB)
Query
```bash
node_filesystem_size_bytes{mountpoint="/",fstype!~"tmpfs|overlay"}
/ 1024^3
```
Unit:
```bash
GiB
```
Description:
Total filesystem size.

### 5. Uptime
Query:
```bash
node_time_seconds - node_boot_time_seconds
```
Unit:
```bash
seconds
```
Grafana Unit

Choose:
duration (s)

Description:
Server uptime.

### 6. Memory Utilization (%)
Query
```bash
100 * (
  1 -
  (
    node_memory_MemAvailable_bytes
    /
    node_memory_MemTotal_bytes
  )
)
```
Unit:
```bash
Percent (0-100)
```
Panel Type: 

### Time Series

Description:
Shows RAM usage over time.

### 7. Memory Usage Gauge (%)
Query
```bash
100 * (
  1 -
  (
    node_memory_MemAvailable_bytes
    /
    node_memory_MemTotal_bytes
  )
)
```
Unit:
```bash
Percent (0-100)
```
Panel Type : 
### Gauge

### 8. CPU Utilization Gauge (%)
Query
```bash
100 * (
  1 -
  avg(
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  )
)
```
Unit:
```bash
Percent (0-100)
```
Description:

Current CPU usage.

### 9. CPU Utilization Graph (%)
Query
```bash
100 * (
  1 -
  avg(
    rate(node_cpu_seconds_total{mode="idle"}[1m])
  )
)
```
Unit:
```bash
Percent (0-100)
```
Panel Type:
### Time Series

**Zunayed Islam Rabbi** <br> 
**System Engineer** <br>
**Schertech** 
