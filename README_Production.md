# Complete End-to-End Monitoring Deployment: Exporters, Pushgateway, Prometheus & Grafana for NATed Servers

## This comprehensive guide covers everything you need for a production-ready deployment where all servers behind NAT push their metrics to a **centrally managed** Pushgateway. Prometheus scrapes these metrics, and Grafana visualizes the data with pre-configured dashboards.

### 1. Architecture Overview

**Scenario:**
- **77 servers** across the country behind NAT
- **Linux servers:** Run **node_exporter** on port **9100**
- **Windows servers:** Run **windows_exporter** on port **9182**
- **Central monitoring server** (public IP: *203.0.113.10*) running:
  - Pushgateway (port 9091)
  - Prometheus (port 9090)
  - Grafana (port 3000)

**Data Flow:**
```
[Remote Servers] → [Push Scripts] → [Pushgateway] → [Prometheus] → [Grafana Dashboards]
```

### C. Troubleshooting Your Cron Push Method

#### Common Issues and Solutions:

1. **Cron Jobs Not Running:**
```bash
# Check if cron service is running
sudo systemctl status cron

# Start cron if not running
sudo systemctl start cron
sudo systemctl enable cron

# Check cron logs
sudo tail -f /var/log/syslog | grep CRON
```

2. **Curl Command Issues:**
```bash
# Test each part separately
# 1. Test node_exporter access
curl -s http://localhost:9100/metrics

# 2. Test pushgateway connectivity
curl -I https://pushgateway.schertech.com

# 3. Test complete push manually
curl -s http://localhost:9100/metrics | curl --data-binary @- https://pushgateway.schertech.com/metrics/job/node_exporter/instance/test-server
```

3. **SSL/HTTPS Issues:**
```bash
# If you get SSL errors, add -k flag for testing (not recommended for production)
curl -s http://localhost:9100/metrics | curl -k --data-binary @- https://pushgateway.schertech.com/metrics/job/node_exporter/instance/Client-AT-V12-Live

# Better solution: ensure your system has updated CA certificates
sudo apt update && sudo apt install ca-certificates
```

4. **Permission Issues:**
```bash
# Ensure cron has necessary permissions
# Check if user can execute curl
which curl

# If needed, use full path in crontab
/usr/bin/curl -s http://localhost:9100/metrics | /usr/bin/curl --data-binary @- https://pushgateway.schertech.com/metrics/job/node_exporter/instance/Client-AT-V12-Live
```

#### Monitoring Your Push Success:
```bash
# Check if your server appears in Pushgateway
curl -s https://pushgateway.schertech.com/metrics | grep "Client-AT-V12-Live"

# Monitor push frequency (should see updates every 30 seconds)
watch -n 5 'curl -s https://pushgateway.schertech.com/metrics | grep -A5 -B5 "Client-AT-V12-Live"'
```

## 2. Prerequisites and System Preparation

### A. Central Monitoring Server Setup
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y wget curl tar systemd firewalld

# Create monitoring user
sudo useradd --no-create-home --shell /bin/false prometheus
sudo useradd --no-create-home --shell /bin/false pushgateway

# Create directories
sudo mkdir -p /etc/prometheus /var/lib/prometheus /var/lib/pushgateway
sudo chown prometheus:prometheus /etc/prometheus /var/lib/prometheus
sudo chown pushgateway:pushgateway /var/lib/pushgateway
```

### B. Firewall Configuration
```bash
# Enable and configure firewall
sudo systemctl enable firewalld
sudo systemctl start firewalld

# Open required ports
sudo firewall-cmd --permanent --add-port=9090/tcp  # Prometheus
sudo firewall-cmd --permanent --add-port=9091/tcp  # Pushgateway
sudo firewall-cmd --permanent --add-port=3000/tcp  # Grafana
sudo firewall-cmd --reload
```

---

## 3. Pushgateway Installation & Configuration

### A. Download and Install Pushgateway
```bash
# Download latest version
cd /tmp
wget https://github.com/prometheus/pushgateway/releases/download/v1.6.2/pushgateway-1.6.2.linux-amd64.tar.gz
tar -xzf pushgateway-1.6.2.linux-amd64.tar.gz

# Install binary
sudo cp pushgateway-1.6.2.linux-amd64/pushgateway /usr/local/bin/
sudo chown pushgateway:pushgateway /usr/local/bin/pushgateway
sudo chmod +x /usr/local/bin/pushgateway
```

### B. Create Pushgateway Service
Create `/etc/systemd/system/pushgateway.service`:
```ini
[Unit]
Description=Prometheus Pushgateway
Wants=network-online.target
After=network-online.target

[Service]
User=pushgateway
Group=pushgateway
Type=simple
ExecStart=/usr/local/bin/pushgateway \
    --web.listen-address=":9091" \
    --web.telemetry-path="/metrics" \
    --persistence.file="/var/lib/pushgateway/pushgateway.db" \
    --persistence.interval=5m \
    --log.level=info
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

### C. Start Pushgateway Service
```bash
sudo systemctl daemon-reload
sudo systemctl enable pushgateway
sudo systemctl start pushgateway

# Verify status
sudo systemctl status pushgateway

# Test accessibility
curl http://localhost:9091/metrics
```

---

## 4. Prometheus Installation & Configuration

### A. Install Prometheus
```bash
# Download Prometheus
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz
tar -xzf prometheus-2.45.0.linux-amd64.tar.gz

# Install binaries
sudo cp prometheus-2.45.0.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.45.0.linux-amd64/promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool

# Copy console files
sudo cp -r prometheus-2.45.0.linux-amd64/consoles /etc/prometheus/
sudo cp -r prometheus-2.45.0.linux-amd64/console_libraries /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
```

### B. Create Prometheus Configuration
Create `/etc/prometheus/prometheus.yml`:
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    monitor: 'pushgateway-monitor'

rule_files:
  - "/etc/prometheus/rules/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: []

scrape_configs:
  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
    scrape_interval: 5s

  # Pushgateway metrics
  - job_name: 'pushgateway'
    static_configs:
      - targets: ['localhost:9091']
    scrape_interval: 15s
    honor_labels: true
    metrics_path: /metrics

  # Direct monitoring of local services
  - job_name: 'pushgateway-service'
    static_configs:
      - targets: ['localhost:9091']
    metrics_path: /metrics
    params:
      collect[]: ['pushgateway']
```

### C. Create Prometheus Service
Create `/etc/systemd/system/prometheus.service`:
```ini
[Unit]
Description=Prometheus Server
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries \
    --web.listen-address=0.0.0.0:9090 \
    --web.enable-lifecycle \
    --storage.tsdb.retention.time=30d
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

### D. Create Alert Rules Directory and Start Prometheus
```bash
sudo mkdir -p /etc/prometheus/rules
sudo chown -R prometheus:prometheus /etc/prometheus

# Start Prometheus
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus

# Verify
sudo systemctl status prometheus
curl http://localhost:9090/api/v1/status/config
```

---

## 5. Node Exporter Setup (Linux Servers)

### A. Installation Script for Linux Servers
Create `/usr/local/bin/install_node_exporter.sh`:
```bash
#!/bin/bash
set -e

NODE_EXPORTER_VERSION="1.6.1"
ARCH="linux-amd64"

# Create user
sudo useradd --no-create-home --shell /bin/false node_exporter || true

# Download and install
cd /tmp
wget "https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.${ARCH}.tar.gz"
tar -xzf "node_exporter-${NODE_EXPORTER_VERSION}.${ARCH}.tar.gz"

# Install binary
sudo cp "node_exporter-${NODE_EXPORTER_VERSION}.${ARCH}/node_exporter" /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
sudo chmod +x /usr/local/bin/node_exporter

# Create service file
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
    --web.listen-address=":9100" \
    --collector.systemd \
    --collector.processes \
    --collector.diskstats.ignored-devices="^(ram|loop|fd|(h|s|v|xv)d[a-z]|nvme\\\\d+n\\\\d+p)\\\\d+$"
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

# Start service
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter

echo "Node Exporter installed and started successfully"
```

### B. Simple Direct Cron Push (Your Method)
This is the straightforward approach you're using - directly pushing via cron without intermediate scripts.

#### Step 1: Verify Node Exporter is Running
```bash
# Test that node_exporter is accessible
curl -s http://localhost:9100/metrics | head -20

# Check if service is running
sudo systemctl status node_exporter
```

#### Step 2: Set Up Direct Cron Jobs
```bash
# Edit crontab for the current user (or use sudo crontab -e for root)
crontab -e

# Add these two lines to push metrics twice per minute with 30-second delay
# Replace 'Client-AT-V12-Live' with your actual server identifier
* * * * * curl -s http://localhost:9100/metrics | curl --data-binary @- https://pushgateway.schertech.com/metrics/job/node_exporter/instance/Client-AT-V12-Live
* * * * * sleep 30 && curl -s http://localhost:9100/metrics | curl --data-binary @- https://pushgateway.schertech.com/metrics/job/node_exporter/instance/Client-AT-V12-Live
```

#### Step 3: Server Instance Naming Convention
For consistency across your 77 servers, use a clear naming pattern:
```bash
# Examples of instance names:
# Client-AT-V12-Live     (your current format)
# Client-NY-V08-Prod
# Client-TX-V15-Dev
# Client-CA-V03-Test

# The format appears to be: Client-[Location]-V[Number]-[Environment]
```

#### Step 4: Verify Cron Jobs Are Working
```bash
# Check cron service is running
sudo systemctl status cron

# View crontab to confirm entries
crontab -l

# Check system logs for cron execution
sudo tail -f /var/log/syslog | grep CRON

# Test manual execution
curl -s http://localhost:9100/metrics | curl --data-binary @- https://pushgateway.schertech.com/metrics/job/node_exporter/instance/$(hostname)
```

#### Step 5: Optional - Add Error Logging
If you want to add basic logging without complex scripts:
```bash
# Edit crontab again
crontab -e

# Replace your existing lines with these (adds basic logging):
* * * * * curl -s http://localhost:9100/metrics | curl --data-binary @- https://pushgateway.schertech.com/metrics/job/node_exporter/instance/Client-AT-V12-Live >> /var/log/push_metrics.log 2>&1
* * * * * sleep 30 && curl -s http://localhost:9100/metrics | curl --data-binary @- https://pushgateway.schertech.com/metrics/job/node_exporter/instance/Client-AT-V12-Live >> /var/log/push_metrics.log 2>&1
```

#### Step 6: Mass Deployment Script
For deploying to all 77 servers, create `deploy_cron_push.sh`:
```bash
#!/bin/bash
# Script to deploy cron-based push to multiple servers

# Configuration
PUSHGATEWAY_URL="https://pushgateway.schertech.com"
SERVER_IDENTIFIER="${1:-$(hostname)}"  # Use provided name or hostname

# Ensure node_exporter is running
if ! systemctl is-active --quiet node_exporter; then
    echo "Starting node_exporter..."
    sudo systemctl start node_exporter
    sudo systemctl enable node_exporter
fi

# Add cron jobs
echo "Setting up cron jobs for server: $SERVER_IDENTIFIER"

# Remove any existing push jobs
crontab -l 2>/dev/null | grep -v "pushgateway.schertech.com" | crontab -

# Add new jobs
(crontab -l 2>/dev/null; echo "* * * * * curl -s http://localhost:9100/metrics | curl --data-binary @- $PUSHGATEWAY_URL/metrics/job/node_exporter/instance/$SERVER_IDENTIFIER") | crontab -
(crontab -l 2>/dev/null; echo "* * * * * sleep 30 && curl -s http://localhost:9100/metrics | curl --data-binary @- $PUSHGATEWAY_URL/metrics/job/node_exporter/instance/$SERVER_IDENTIFIER") | crontab -

echo "Cron jobs installed successfully for $SERVER_IDENTIFIER"
echo "Current crontab:"
crontab -l
```

#### Usage:
```bash
# Make executable
chmod +x deploy_cron_push.sh

# Deploy with custom server name
./deploy_cron_push.sh Client-AT-V12-Live

# Or deploy with hostname
./deploy_cron_push.sh
```

---

## 6. Windows Exporter Setup (Windows Servers)

### A. Windows Exporter Installation PowerShell Script
Create `install_windows_exporter.ps1`:
```powershell
# Install Windows Exporter
param(
    [string]$Version = "0.21.1",
    [string]$InstallPath = "C:\Program Files\windows_exporter"
)

# Create installation directory
if (!(Test-Path $InstallPath)) {
    New-Item -ItemType Directory -Path $InstallPath -Force
}

# Download and install
$downloadUrl = "https://github.com/prometheus-community/windows_exporter/releases/download/v$Version/windows_exporter-$Version-amd64.exe"
$exePath = Join-Path $InstallPath "windows_exporter.exe"

Write-Host "Downloading Windows Exporter v$Version..."
Invoke-WebRequest -Uri $downloadUrl -OutFile $exePath

# Install as service
Write-Host "Installing as Windows service..."
& $exePath install --config.file="C:\Program Files\windows_exporter\config.yml"

# Create config file
$configContent = @"
collectors:
  enabled: cpu,cs,logical_disk,net,os,service,system,textfile,memory,iis,process
collector:
  service:
    services-where: "Name='windows_exporter'"
  textfile:
    directory: "C:\Program Files\windows_exporter\textfile_inputs"
"@

$configContent | Out-File -FilePath "C:\Program Files\windows_exporter\config.yml" -Encoding UTF8

# Create textfile directory
New-Item -ItemType Directory -Path "C:\Program Files\windows_exporter\textfile_inputs" -Force

# Start service
Start-Service -Name "windows_exporter"
Set-Service -Name "windows_exporter" -StartupType Automatic

Write-Host "Windows Exporter installed and started successfully on port 9182"
```

### B. Windows Push Script
Create `push_windows_metrics.ps1`:
```powershell
param(
    [string]$PushgatewayUrl = "http://203.0.113.10:9091",
    [string]$ServerName = $env:COMPUTERNAME,
    [string]$JobName = "windows_exporter"
)

# Logging function
function Write-Log {
    param([string]$Message)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $logEntry = "$timestamp - $Message"
    Write-Output $logEntry
    Add-Content -Path "C:\push_metrics.log" -Value $logEntry
}

# Health check function
function Test-WindowsExporter {
    try {
        $response = Invoke-WebRequest -Uri "http://localhost:9182/metrics" -TimeoutSec 10
        return $response.StatusCode -eq 200
    }
    catch {
        Write-Log "ERROR: Windows exporter health check failed: $($_.Exception.Message)"
        return $false
    }
}

# Push metrics function
function Push-Metrics {
    $maxAttempts = 3
    $attempt = 1
    
    while ($attempt -le $maxAttempts) {
        Write-Log "Pushing metrics (attempt $attempt/$maxAttempts)"
        
        try {
            # Get metrics from windows_exporter
            $metrics = Invoke-WebRequest -Uri "http://localhost:9182/metrics" -TimeoutSec 30
            
            # Push to gateway
            $pushUrl = "$PushgatewayUrl/metrics/job/$JobName/instance/$ServerName"
            $response = Invoke-WebRequest -Uri $pushUrl -Method Post -Body $metrics.Content -ContentType "text/plain" -TimeoutSec 30
            
            if ($response.StatusCode -eq 200 -or $response.StatusCode -eq 202) {
                Write-Log "Successfully pushed metrics"
                return $true
            }
        }
        catch {
            Write-Log "Failed to push metrics (attempt $attempt): $($_.Exception.Message)"
        }
        
        $attempt++
        if ($attempt -le $maxAttempts) {
            Start-Sleep -Seconds 5
        }
    }
    
    Write-Log "ERROR: Failed to push metrics after $maxAttempts attempts"
    return $false
}

# Main execution
Write-Log "Starting metrics push for server: $ServerName"

if (Test-WindowsExporter) {
    if (Push-Metrics) {
        Write-Log "Metrics push completed successfully"
        exit 0
    } else {
        Write-Log "Metrics push failed"
        exit 1
    }
} else {
    Write-Log "Windows exporter health check failed"
    exit 1
}
```

### C. Windows Task Scheduler Setup
Create `create_scheduled_task.ps1`:
```powershell
# Create scheduled task for metrics pushing
$taskName = "Push Metrics to Pushgateway"
$scriptPath = "C:\push_windows_metrics.ps1"

# Create the scheduled task
$action = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-ExecutionPolicy Bypass -File `"$scriptPath`""
$trigger = New-ScheduledTaskTrigger -RepetitionInterval (New-TimeSpan -Minutes 1) -RepetitionDuration (New-TimeSpan -Days 365) -At (Get-Date) -Once
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -StartWhenAvailable -RunOnlyIfNetworkAvailable
$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount

Register-ScheduledTask -TaskName $taskName -Action $action -Trigger $trigger -Settings $settings -Principal $principal -Description "Push Windows Exporter metrics to Pushgateway every minute"

Write-Host "Scheduled task '$taskName' created successfully"
```

---

## 7. Grafana Installation & Configuration

### A. Install Grafana
```bash
# Add Grafana repository
sudo apt-get install -y software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

# Update and install
sudo apt-get update
sudo apt-get install -y grafana

# Configure Grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

### B. Grafana Configuration File
Create/edit `/etc/grafana/grafana.ini`:
```ini
[server]
http_addr = 0.0.0.0
http_port = 3000
domain = 203.0.113.10

[security]
admin_user = admin
admin_password = your_secure_password_here

[users]
allow_sign_up = false
default_theme = dark

[auth.anonymous]
enabled = false

[dashboards]
default_home_dashboard_path = /var/lib/grafana/dashboards/overview.json

[provisioning]
datasources = /etc/grafana/provisioning/datasources
dashboards = /etc/grafana/provisioning/dashboards
```

### C. Provision Prometheus Data Source
Create `/etc/grafana/provisioning/datasources/prometheus.yml`:
```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    orgId: 1
    url: http://localhost:9090
    basicAuth: false
    isDefault: true
    version: 1
    editable: true
    jsonData:
      httpMethod: POST
      queryTimeout: "60s"
      timeInterval: "15s"
```

### D. Dashboard Provisioning
Create `/etc/grafana/provisioning/dashboards/dashboard.yml`:
```yaml
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards
```

### E. Create Comprehensive Dashboards

#### 1. System Overview Dashboard
Create `/var/lib/grafana/dashboards/system-overview.json`:
```json
{
  "dashboard": {
    "id": null,
    "title": "System Overview - All Servers",
    "tags": ["prometheus", "pushgateway"],
    "timezone": "browser",
    "refresh": "30s",
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "panels": [
      {
        "id": 1,
        "title": "Server Status",
        "type": "stat",
        "targets": [
          {
            "expr": "count(up{job=~\"node_exporter|windows_exporter\"} == 1)",
            "legendFormat": "Online Servers"
          },
          {
            "expr": "count(up{job=~\"node_exporter|windows_exporter\"} == 0)",
            "legendFormat": "Offline Servers"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
        "fieldConfig": {
          "defaults": {
            "color": {"mode": "palette-classic"},
            "custom": {
              "displayMode": "basic",
              "orientation": "horizontal"
            }
          }
        }
      },
      {
        "id": 2,
        "title": "CPU Usage by Server",
        "type": "bargauge",
        "targets": [
          {
            "expr": "100 - (avg by (instance) (irate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
            "legendFormat": "{{instance}}"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0}
      },
      {
        "id": 3,
        "title": "Memory Usage by Server",
        "type": "bargauge",
        "targets": [
          {
            "expr": "(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100",
            "legendFormat": "{{instance}}"
          }
        ],
        "gridPos": {"h": 8, "w": 24, "x": 0, "y": 8}
      },
      {
        "id": 4,
        "title": "Disk Usage by Server",
        "type": "table",
        "targets": [
          {
            "expr": "100 - ((node_filesystem_avail_bytes{mountpoint=\"/\"} / node_filesystem_size_bytes{mountpoint=\"/\"}) * 100)",
            "legendFormat": "{{instance}}",
            "format": "table"
          }
        ],
        "gridPos": {"h": 8, "w": 24, "x": 0, "y": 16}
      }
    ]
  }
}
```

#### 2. Linux Servers Dashboard
Create `/var/lib/grafana/dashboards/linux-servers.json`:
```json
{
  "dashboard": {
    "id": null,
    "title": "Linux Servers - Detailed Monitoring",
    "tags": ["linux", "node_exporter"],
    "timezone": "browser",
    "refresh": "30s",
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "templating": {
      "list": [
        {
          "name": "instance",
          "type": "query",
          "query": "label_values(node_exporter_build_info, instance)",
          "refresh": 1,
          "includeAll": true,
          "allValue": ".*"
        }
      ]
    },
    "panels": [
      {
        "id": 1,
        "title": "CPU Usage",
        "type": "timeseries",
        "targets": [
          {
            "expr": "100 - (avg by (instance) (irate(node_cpu_seconds_total{instance=~\"$instance\",mode=\"idle\"}[5m])) * 100)",
            "legendFormat": "CPU Usage - {{instance}}"
          }
        ],
        "gridPos": {"h": 9, "w": 12, "x": 0, "y": 0}
      },
      {
        "id": 2,
        "title": "Memory Usage",
        "type": "timeseries",
        "targets": [
          {
            "expr": "100 * (1 - ((node_memory_MemFree_bytes{instance=~\"$instance\"} + node_memory_Cached_bytes{instance=~\"$instance\"} + node_memory_Buffers_bytes{instance=~\"$instance\"}) / node_memory_MemTotal_bytes{instance=~\"$instance\"}))",
            "legendFormat": "Memory Usage - {{instance}}"
          }
        ],
        "gridPos": {"h": 9, "w": 12, "x": 12, "y": 0}
      },
      {
        "id": 3,
        "title": "Disk I/O",
        "type": "timeseries",
        "targets": [
          {
            "expr": "irate(node_disk_read_bytes_total{instance=~\"$instance\"}[5m])",
            "legendFormat": "Read - {{instance}} - {{device}}"
          },
          {
            "expr": "irate(node_disk_written_bytes_total{instance=~\"$instance\"}[5m])",
            "legendFormat": "Write - {{instance}} - {{device}}"
          }
        ],
        "gridPos": {"h": 9, "w": 12, "x": 0, "y": 9}
      },
      {
        "id": 4,
        "title": "Network I/O",
        "type": "timeseries",
        "targets": [
          {
            "expr": "irate(node_network_receive_bytes_total{instance=~\"$instance\",device!=\"lo\"}[5m])",
            "legendFormat": "Receive - {{instance}} - {{device}}"
          },
          {
            "expr": "irate(node_network_transmit_bytes_total{instance=~\"$instance\",device!=\"lo\"}[5m])",
            "legendFormat": "Transmit - {{instance}} - {{device}}"
          }
        ],
        "gridPos": {"h": 9, "w": 12, "x": 12, "y": 9}
      }
    ]
  }
}
```

#### 3. Windows Servers Dashboard
Create `/var/lib/grafana/dashboards/windows-servers.json`:
```json
{
  "dashboard": {
    "id": null,
    "title": "Windows Servers - Detailed Monitoring",
    "tags": ["windows", "windows_exporter"],
    "timezone": "browser",
    "refresh": "30s",
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "templating": {
      "list": [
        {
          "name": "instance",
          "type": "query",
          "query": "label_values(windows_exporter_build_info, instance)",
          "refresh": 1,
          "includeAll": true,
          "allValue": ".*"
        }
      ]
    },
    "panels": [
      {
        "id": 1,
        "title": "CPU Usage",
        "type": "timeseries",
        "targets": [
          {
            "expr": "100 - (avg by (instance) (windows_cpu_time_total{instance=~\"$instance\",mode=\"idle\"}) * 100)",
            "legendFormat": "CPU Usage - {{instance}}"
          }
        ],
        "gridPos": {"h": 9, "w": 12, "x": 0, "y": 0}
      },
      {
        "id": 2,
        "title": "Memory Usage",
        "type": "timeseries",
        "targets": [
          {
            "expr": "100 * (1 - (windows_os_physical_memory_free_bytes{instance=~\"$instance\"} / windows_cs_physical_memory_bytes{instance=~\"$instance\"}))",
            "legendFormat": "Memory Usage - {{instance}}"
          }
        ],
        "gridPos": {"h": 9, "w": 12, "x": 12, "y": 0}
      },
      {
        "id": 3,
        "title": "Disk Usage",
        "type": "timeseries",
        "targets": [
          {
            "expr": "100 * (1 - (windows_logical_disk_free_bytes{instance=~\"$instance\"} / windows_logical_disk_size_bytes{instance=~\"$instance\"}))",
            "legendFormat": "Disk Usage - {{instance}} - {{volume}}"
          }
        ],
        "gridPos": {"h": 9, "w": 24, "x": 0, "y": 9}
      },
      {
        "id": 4,
        "title": "Services Status",
        "type": "table",
        "targets": [
          {
            "expr": "windows_service_status{instance=~\"$instance\"}",
            "legendFormat": "{{instance}} - {{name}}",
            "format": "table"
          }
        ],
        "gridPos": {"h": 9, "w": 24, "x": 0, "y": 18}
      }
    ]
  }
}
```

### F. Set Dashboard Permissions
```bash
# Set correct permissions for dashboard files
sudo chown -R grafana:grafana /var/lib/grafana/dashboards/
sudo chmod 644 /var/lib/grafana/dashboards/*.json

# Restart Grafana to load new configurations
sudo systemctl restart grafana-server
```

---

## 8. Monitoring and Alerting Setup

### A. Create Alert Rules
Create `/etc/prometheus/rules/alerting.yml`:
```yaml
groups:
  - name: server.alerts
    rules:
      - alert: ServerDown
        expr: up{job=~"node_exporter|windows_exporter"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Server {{ $labels.instance }} is down"
          description: "{{ $labels.instance }} has been down for more than 1 minute"

      - alert: HighCPUUsage
        expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is above 80% for more than 5 minutes"

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is above 85% for more



# Script by Zunayed islam Rabby.
# Designation : System Engineer .