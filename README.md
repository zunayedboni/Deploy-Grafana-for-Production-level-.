

# End-to-End Monitoring Deployment: Exporters, Pushgateway, Prometheus & Grafana for NATed Servers

This guide covers everything you need for a production-ready deployment where all servers behind NAT push their metrics to a **centrally managed** Pushgateway. Prometheus scrapes these metrics, and Grafana is used to visualize the data.

## 1. Overview

**Scenario:**
- **Servers:**
  - 77 servers across the country.
  - **Linux servers:** Run **node_exporter** on port **9100**.
  - **Windows servers:** Run **windows_exporter** on port **9182**.
- **Network:**
  - All servers are behind NAT, so they cannot be directly scraped by Prometheus.
- **Solution:**
  - Each server pushes its locally collected metrics to a central **Pushgateway** running on the same server as Prometheus (assumed public IP: *203.0.113.10*).
  - Prometheus scrapes the Pushgateway.
  - Grafana is installed on the same server (or elsewhere) to visualize the metrics.

*Note:* This guide incorporates best practices and insights from sources such as the Pluralsight Labs post on [Installing Prometheus Pushgateway](https://www.pluralsight.com/labs/aws/installing-prometheus-pushgateway).

---

## 2. Exporter Setup on the Servers

### A. Linux Servers: Installing & Configuring node_exporter

1. **Download and Extract:**
   ```bash
   wget https://github.com/prometheus/node_exporter/releases/download/v1.6.0/node_exporter-1.6.0.linux-amd64.tar.gz
   tar -zxvf node_exporter-1.6.0.linux-amd64.tar.gz
   unzip the file and move it to "/opt/node_exporter/"
   sudo mv node_exporter-1.6.0.linux-amd64 /opt/node_exporter/
   ```
   *Explanation:*  
   - `wget` downloads the tarball.  
   - `tar -zxvf` extracts the archive.


2. ** Install as a Service:**
   Create a systemd service file at `/etc/systemd/system/node_exporter.service`:
   ```ini
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
   Then reload and enable:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl start node_exporter
   sudo systemctl enable node_exporter
   ```

*Repeat on each Linux server (or automate with configuration management) so every Linux node runs node_exporter on port 9100.*

---

### B. Windows Servers: Installing & Configuring windows_exporter

1. **Download and Install:**
   - Download the latest MSI installer from the [windows_exporter GitHub releases](https://github.com/prometheus-community/windows_exporter/releases).
   - Run the installer on each Windows server; the default listening port is **9182**.

2. **Verify the Installation:**
   - Open a browser on the server and navigate to:
     ```
    curl -L localhost:9100/metrics
     ```
   - Confirm that metrics are displayed.

3. **Service Installation:**
   - The installer sets up windows_exporter as a Windows service. Verify in the Services console.

---

## 3. Setting Up the Pushgateway on the Prometheus Server

The Pushgateway will run on the same server as Prometheus (public IP: *203.0.113.10*).

### A. Download and Extract the Pushgateway

1. **Download:**
   ```bash
   wget https://github.com/prometheus/pushgateway/releases/download/v1.5.0/pushgateway-1.5.0.linux-amd64.tar.gz
   tar -zxvf pushgateway-1.5.0.linux-amd64.tar.gz
   cd pushgateway-1.5.0.linux-amd64
   ```

### B. Run the Pushgateway for Testing

Start it manually:
```bash
./pushgateway
```
- By default, it listens on port **9091**.
- Verify by running:
  ```bash
  curl http://localhost:9091/metrics
  ```

### C. Configure the Pushgateway

Run it with command-line flags:
```bash
./pushgateway --web.listen-address=":9091" \
  --persistence.file="/var/lib/pushgateway/metrics" \
  --log.level="info" \
  --web.external-url="https://pushgateway.site.com/"
```
*Explanation:*
- **--web.listen-address=":9091"**: Sets the listening port.
- **--persistence.file="/var/lib/pushgateway/metrics"**: Persists metrics across restarts.
- **--log.level="info"**: Sets the logging level.
- **--web.external-url="https://pushgateway.site.com/"**: Specifies the external URL if behind a reverse proxy.

### D. Run Pushgateway as a Service (systemd)

1. **Create a Service File:**
   ```bash
   sudo nano /etc/systemd/system/pushgateway.service
   ```
   Insert:
   ```ini
   [Unit]
   Description=Prometheus Pushgateway
   After=network.target

   [Service]
   User=prometheus
   Group=prometheus
   ExecStart=/usr/local/bin/pushgateway \
       --web.listen-address=":9091" \
       --persistence.file="/var/lib/pushgateway/metrics" \
       --log.level="info" \
       --web.external-url="https://pushgateway.site.com/"
   Restart=always

   [Install]
   WantedBy=multi-user.target
   ```
2. **Reload and Enable:**
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl start pushgateway
   sudo systemctl enable pushgateway
   ```
3. **Verify:**
   ```bash
   sudo systemctl status pushgateway
   ```

---

## 4. Integrating Prometheus with the Pushgateway

Edit your `prometheus.yml` (on the same server or where Prometheus is installed):

```yaml
scrape_configs:
  - job_name: 'pushgateway'
    metrics_path: /metrics
    static_configs:
      - targets: ['localhost:9091']
```
*Explanation:*
- **job_name:** Identifies the scrape job.
- **metrics_path:** The endpoint that exposes Pushgateway’s metrics.
- **targets:** Since Pushgateway is local, use `localhost:9091`.

Reload Prometheus after editing.

---

## 5. Client Push Scripts for Exporting Metrics

Since your servers are behind NAT, each server must push its metrics to the Pushgateway.

### A. Linux Servers: Push Script for node_exporter

1. **Create the Script:**
   Create `/usr/local/bin/push_metrics.sh`:
   ```bash
   sudo nano /usr/local/bin/push_metrics.sh
   ```
   Insert:
   ```bash
   #!/bin/bash
   # Fetch metrics from node_exporter and push to the Pushgateway

   # First push: Retrieve metrics and push immediately
   /usr/bin/curl -s http://localhost:9100/metrics | /usr/bin/curl --data-binary @- https://pushgateway.site.com/metrics/job/node_exporter/instance/Client-LP-Test

   # Wait for 30 seconds
   sleep 30

   # Second push: Repeat the process
   /usr/bin/curl -s http://localhost:9100/metrics | /usr/bin/curl --data-binary @- https://pushgateway.site.com/metrics/job/node_exporter/instance/Client-LP-Test
   ```
   *Explanation:*  
   - Uses `curl -s` to silently fetch metrics from node_exporter.
   - Pipes the output to a second `curl` command that pushes the data.
   - `sleep 30` provides a pause to update metrics.

2. **Make It Executable:**
   ```bash
   sudo chmod +x /usr/local/bin/push_metrics.sh
   ```

3. **Schedule via Cron:**
   Open the crontab editor:
   ```bash
   crontab -e
   ```
   Add:
   ```cron
   * * * * * /usr/local/bin/push_metrics.sh
   ```
   This schedules the script to run every minute.

### B. Windows Servers: Push Script for windows_exporter

1. **Create a Batch File:**
   Create `C:\push_metrics.bat` with the following content:
   ```batch
   @echo off
   REM Push metrics from windows_exporter to the Pushgateway

   REM First push: Retrieve metrics and push to Pushgateway
   curl -s http://localhost:9182/metrics | curl --data-binary @- https://pushgateway.site.com/metrics/job/windows_exporter/instance/Client-LP-Test

   REM Wait for 30 seconds
   timeout /T 30

   REM Second push: Repeat the process
   curl -s http://localhost:9182/metrics | curl --data-binary @- https://pushgateway.site.com/metrics/job/windows_exporter/instance/Client-LP-Test
   ```
   *Explanation:*
   - `@echo off` silences command output.
   - `curl -s` fetches metrics silently.
   - `timeout /T 30` pauses for 30 seconds.

2. **Schedule with Task Scheduler:**
   - **Open Task Scheduler:** Press **Win+R**, type `taskschd.msc`, and hit Enter.
   - **Create a New Task:**
     - **General Tab:**  
       - Name the task (e.g., "Push Metrics to Pushgateway").  
       - Select “Run whether user is logged on or not” and “Run with highest privileges.”
     - **Triggers Tab:**  
       - Create a trigger to start on a schedule—repeat every 1 minute indefinitely.
     - **Actions Tab:**  
       - Create an action to “Start a program” with the script path `C:\push_metrics.bat`.
   - **Save the Task.**

---

## 6. Installing and Configuring Grafana

Grafana is used to visualize the metrics stored in Prometheus. Below are step-by-step instructions for installing Grafana on your Prometheus server.

### A. Install Grafana on Linux

1. **Add the Grafana APT Repository (Debian/Ubuntu):**
   ```bash
   sudo apt-get install -y software-properties-common wget
   wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
   echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
   sudo apt-get update
   ```
   *Explanation:*
   - Installs necessary utilities.
   - Imports the Grafana GPG key and adds the repository.

2. **Install Grafana:**
   ```bash
   sudo apt-get install grafana
   ```

3. **Start and Enable Grafana:**
   ```bash
   sudo systemctl start grafana-server
   sudo systemctl enable grafana-server
   ```
   *Explanation:*
   - Starts the Grafana service and enables it to start on boot.

4. **Access Grafana:**
   - Open a web browser and navigate to:  
     ```
     http://<your-prometheus-server-IP>:3000
     ```
   - Default login is **admin/admin** (you’ll be prompted to change the password).

### B. Configuring Grafana to Use Prometheus as a Data Source

1. **Log into Grafana:**
   - Use your web browser to log in at `http://203.0.113.10:3000`.

2. **Add a Data Source:**
   - Click on the gear icon (Configuration) on the left sidebar and select **Data Sources**.
   - Click **Add data source**.
   - Select **Prometheus**.
   - In the URL field, enter:
     ```
     http://localhost:9090
     ```
     (Assuming Prometheus is running locally on port 9090.)
   - Click **Save & Test** to ensure Grafana can connect to Prometheus.

3. **Create Dashboards:**
   - Click on the **+** icon on the left sidebar and select **Dashboard**.
   - Click **Add new panel**.
   - Use the Query editor to create visualizations (for example, query the metrics pushed from the Pushgateway such as `node_cpu_seconds_total` or your custom test metrics).
   - Save the dashboard for continuous monitoring.

---

## 7. Testing and Final Verification

### A. Testing the Pushgateway

1. **Manual Test Push:**
   ```bash
   echo "test_metric 123" | curl --data-binary @- http://localhost:9091/metrics/job/test/instance/localhost
   ```
2. **Verify via Curl:**
   ```bash
   curl http://localhost:9091/metrics
   ```
   Confirm that `test_metric 123` appears in the output.

### B. Verify Prometheus and Grafana

- **Prometheus:**  
  Open the Prometheus UI and query for `test_metric` or your pushed metrics to confirm data is being scraped.
- **Grafana:**  
  Verify your dashboards display up-to-date metrics from Prometheus.

---

## 8. Summary

1. **Exporter Setup:**
   - **Linux:** Install and run node_exporter (port 9100) on all Linux servers; optionally set it up as a service.
   - **Windows:** Install windows_exporter (port 9182) on all Windows servers.
2. **Pushgateway Setup:**
   - Download, extract, and test the Pushgateway on your Prometheus server.
   - Configure it with flags and run it as a systemd service.
3. **Prometheus Integration:**
   - Update `prometheus.yml` to scrape from the Pushgateway running on `localhost:9091`.
4. **Client Push Scripts:**
   - **Linux:** Create and schedule a bash script to push node_exporter metrics via cron.
   - **Windows:** Create and schedule a batch file to push windows_exporter metrics via Task Scheduler.
5. **Grafana Installation & Configuration:**
   - Install Grafana using the official repository.
   - Start and enable Grafana.
   - Configure Grafana to use Prometheus as a data source.
   - Create dashboards to visualize your metrics.
6. **Testing:**
   - Manually push test metrics, verify via curl, and ensure Prometheus and Grafana show the correct data.
7. **Best Practices:**
   - Incorporate persistence in the Pushgateway.
   - Secure access (via reverse proxies or firewall rules) to the Pushgateway and Grafana.
   - Monitor logs and service statuses to ensure reliability.
# Deploy-Grafana-for-Production-level-.
