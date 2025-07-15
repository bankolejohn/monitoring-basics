## Monitor Linux Server using Prometheus Node Exporter

### **Introduction**

Effective monitoring is crucial for maintaining the health, performance, and reliability of any Linux server. It allows administrators to gain real-time insights into system metrics, detect anomalies, troubleshoot issues proactively, and optimize resource utilization. Prometheus Node Exporter is an official Prometheus exporter that collects a wide array of hardware and operating system metrics (like CPU usage, memory, disk I/O, network statistics, and more) from a Linux host.

This project will guide you through the process of installing and configuring Prometheus Node Exporter on a Linux server and integrating it with a Prometheus server to collect and visualize these vital metrics.

### **Objectives**

Upon successful completion of this project, you will be able to:

  * Install and configure Prometheus Node Exporter on a target Linux server.
  * Understand how to manage Node Exporter as a systemd service.
  * Configure your Prometheus server to discover and scrape metrics from the Node Exporter.
  * Access the Prometheus web interface to verify metric collection.
  * Explore, query, and analyze various system metrics collected by Node Exporter using PromQL.
  * (Optional) Understand the basic concept of setting up alerts for key metrics.

### **Prerequisites**

To successfully complete this project, you will need:

  * **Linux Server (Target Node):** A running Linux server (e.g., Ubuntu, CentOS, Debian) where Node Exporter will be installed. This server must have `sudo` privileges configured.
  * **Prometheus Instance (Monitoring Server):** A working Prometheus setup, either on a separate Linux server or on your local machine. If you don't have Prometheus installed, you will need to set it up first (this project *assumes* Prometheus is already running).
  * **Network Access:**
      * The Prometheus server must be able to connect to the Linux target server on **port 9100** (Node Exporter's default port). Ensure firewalls (e.g., `ufw`, `firewalld`) and cloud security groups allow this connection.
      * If Prometheus is running on a remote server, you'll need network access to **port 9090** (Prometheus UI default port) from your web browser.
  * **Tools:** Terminal access to both the Linux server (target) and Prometheus server, and a text editor for configuration files.

### **Estimated Time**

1-2 hours

### **Tasks Outline**

1.  **Install Prometheus Node Exporter** on the target Linux server.
2.  **Start and Enable Node Exporter** as a systemd service.
3.  **Configure Prometheus** to scrape metrics from Node Exporter.
4.  **Verify and Query Node Exporter Metrics** in the Prometheus web interface.
5.  **Explore and Analyze** the collected metrics using PromQL.

-----

### **Project Tasks: Detailed Steps**

Follow these detailed steps on your **Target Linux Server** unless explicitly stated otherwise.

#### **Task 1 - Install Prometheus Node Exporter**

This task involves downloading, extracting, and placing the Node Exporter binary in an executable path.

1.  **Download the latest Node Exporter binary:**
    On your **Target Linux Server**, use `curl` to download the compressed binary from the official Prometheus GitHub releases page.

    ```bash
    curl -LO https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-linux-amd64.tar.gz
    ```

2.  **Extract the downloaded tarball:**

    ```bash
    tar -xvf node_exporter-linux-amd64.tar.gz
    ```

    This will create a directory named `node_exporter-linux-amd64`.

3.  **Move the binary to a directory in your system's PATH:**
    It's standard practice to place executable binaries in `/usr/local/bin/`.

    ```bash
    sudo mv node_exporter-linux-amd64/node_exporter /usr/local/bin/
    ```

4.  **Clean up the extracted directory (optional):**

    ```bash
    rm -rf node_exporter-linux-amd64/ node_exporter-linux-amd64.tar.gz
    ```

#### **Task 2 - Start and Enable Node Exporter as a Service**

To ensure Node Exporter runs automatically and can be managed easily, we'll configure it as a systemd service.

1.  **Create a systemd service file for Node Exporter:**
    Use `nano` (or your preferred text editor) with `sudo` to create a new service file.

    ```bash
    sudo nano /etc/systemd/system/node_exporter.service
    ```

2.  **Add the following content to the file:**

    ```ini
    [Unit]
    Description=Prometheus Node Exporter
    Wants=network-online.target
    After=network-online.target

    [Service]
    User=nobody
    Group=nobody
    Type=simple
    ExecStart=/usr/local/bin/node_exporter
    Restart=always
    RestartSec=5s

    [Install]
    WantedBy=multi-user.target
    ```

      * `User=nobody` and `Group=nobody`: Running the exporter as a non-privileged user (like `nobody`) is a security best practice.
      * `ExecStart`: Specifies the path to the Node Exporter binary.
      * `Restart=always`: Ensures the service restarts if it crashes.

3.  **Save and close the file.** (Ctrl+X, Y, Enter for nano)

4.  **Reload systemd, start, and enable the Node Exporter service:**
    These commands tell systemd about the new service file, start the service, and configure it to start automatically at boot.

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl start node_exporter
    sudo systemctl enable node_exporter
    ```

5.  **Verify that Node Exporter is running:**
    Check the service status.

    ```bash
    sudo systemctl status node_exporter
    ```

    **Expected Output:** You should see `Active: active (running)` in green.

    ```
    ● node_exporter.service - Prometheus Node Exporter
         Loaded: loaded (/etc/systemd/system/node_exporter.service; enabled; vendor preset: enabled)
         Active: active (running) since Mon 2024-07-15 17:00:00 UTC; 1min ago
       Main PID: 1234 (node_exporter)
          Tasks: 7 (limit: 1109)
         Memory: 7.8M
            CPU: 0.089s
         CGroup: /system.slice/node_exporter.service
                 └─1234 /usr/local/bin/node_exporter
    ```

6.  **Confirm Node Exporter is accessible by visiting its metrics endpoint:**
    Open a web browser on your local machine and navigate to `http://<your-target-server-ip>:9100/metrics`.

      * Replace `<your-target-server-ip>` with the actual IP address of your **Target Linux Server**.
      * **Expected Output:** You should see a page full of raw Prometheus metrics data. This confirms Node Exporter is running and serving metrics.

    **IMPORTANT: Configure Firewall:**
    If you cannot access the metrics page, your firewall on the **Target Linux Server** is likely blocking port 9100.

      * **For Ubuntu/Debian with UFW:**
        ```bash
        sudo ufw allow 9100/tcp
        sudo ufw reload
        ```
      * **For CentOS/RHEL/Fedora with Firewalld:**
        ```bash
        sudo firewall-cmd --zone=public --add-port=9100/tcp --permanent
        sudo firewall-cmd --reload
        ```

#### **Task 3 - Configure Prometheus to Scrape Metrics from Node Exporter**

Now, you need to tell your Prometheus server where to find the Node Exporter metrics. Perform these steps on your **Prometheus Instance** (monitoring server).

1.  **Open the Prometheus configuration file (`prometheus.yml`):**
    The default location is usually `/etc/prometheus/prometheus.yml`.

    ```bash
    sudo nano /etc/prometheus/prometheus.yml
    ```

2.  **Add a new scrape job for Node Exporter:**
    Locate the `scrape_configs:` section in your `prometheus.yml` and add the following new job definition. Ensure correct YAML indentation.

    ```yaml
    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['localhost:9090']

      - job_name: 'node-exporter' # New job for Node Exporter
        static_configs:
          - targets: ['<your-target-server-ip>:9100'] # IP of your Linux server running Node Exporter
        # Example if you have multiple servers:
        # - targets:
        #     - 'server1-ip:9100'
        #     - 'server2-ip:9100'
        #     - 'server3-ip:9100'
    ```

      * **CRITICAL:** Replace `<your-target-server-ip>` with the actual IP address of your **Target Linux Server** where Node Exporter is running. Do *not* use `localhost` unless Node Exporter is running on the same machine as Prometheus.

3.  **Save the file and restart Prometheus to apply the changes:**

    ```bash
    sudo systemctl restart prometheus
    ```

      * **Troubleshooting:** If Prometheus fails to restart, check the logs (`sudo journalctl -u prometheus`) for YAML syntax errors in your `prometheus.yml`. You can also run `promtool check config /etc/prometheus/prometheus.yml` to validate the syntax.

#### **Task 4 - Verify and Query Node Exporter Metrics in Prometheus**

Confirm that Prometheus is successfully scraping metrics from your Node Exporter.

1.  **Access the Prometheus web interface:**
    Open a web browser and navigate to `http://<prometheus-server-ip>:9090`.

      * Replace `<prometheus-server-ip>` with the IP address of your Prometheus server (e.g., `localhost:9090` if running locally, or your cloud instance's public IP).

2.  **Check the "Targets" page:**
    In the Prometheus UI, go to **Status** -\> **Targets**. You should see the `node-exporter` job listed, and the state should be **UP**. This confirms Prometheus is successfully connecting to and scraping metrics from your Node Exporter.

3.  **Run a test query to verify Node Exporter metrics:**
    Go to the **Graph** tab in Prometheus. In the expression input box, type a common Node Exporter metric name, for example:

    ```
    node_cpu_seconds_total
    ```

      * Click the **Execute** button. You should see time series data related to CPU usage on your target server.

#### **Task 5 - Explore and Analyze Metrics**

Node Exporter provides a wealth of metrics. Let's explore some key ones.

1.  **Use the Prometheus query interface to explore key Node Exporter metrics:**
    Type these PromQL expressions into the graph tab and click "Execute":

      * **Available Memory (bytes):** `node_memory_MemAvailable_bytes`
      * **Available Disk Space (bytes):** ` node_filesystem_avail_bytes{fstype!="rootfs", mountpoint="/"}  ` (adjust mountpoint if needed)
      * **Network Bytes Received (total):** `node_network_receive_bytes_total`
      * **Network Bytes Transmitted (total):** `node_network_transmit_bytes_total`
      * **Load Average (1-minute):** `node_load1`
      * **Number of running processes:** `node_procs_running`

2.  **Create basic time-series graphs using Prometheus expressions (PromQL):**

      * **CPU Usage (rate over last 5 minutes):**

        ```promql
        sum by (mode) (rate(node_cpu_seconds_total{mode!="idle", mode!="iowait", mode!="steal"}[5m]))
        ```

        This query sums CPU usage across different modes (user, system, nice) over a 5-minute window, excluding idle, iowait, and steal.

      * **Free Disk Space (percentage):**

        ```promql
        (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
        ```

        This shows the percentage of available disk space for the root filesystem.

      * **Network Throughput (bytes/second):**

        ```promql
        rate(node_network_receive_bytes_total[1m]) + rate(node_network_transmit_bytes_total[1m])
        ```

        This shows total network I/O speed (received + transmitted) over the last 1 minute.

3.  **(Optional) Set up Alert Rules for critical metrics:**
    While beyond the scope of this basic setup, in a production environment, you would define alerting rules in Prometheus to notify you when metrics cross certain thresholds (e.g., CPU usage consistently above 90%, disk space below 10%). These rules are typically defined in separate `.yml` files and included in `prometheus.yml`.

### **Conclusion**

In this project, you successfully installed and configured Prometheus Node Exporter on a Linux server, integrated it with your Prometheus monitoring instance, and began exploring the rich set of system metrics it collects. You've established a fundamental real-time monitoring solution for your Linux infrastructure.

This setup provides a strong foundation for maintaining server health and performance. From here, you can extend your monitoring capabilities significantly by:

  * **Adding more target servers** to your Prometheus configuration.
  * **Integrating Grafana:** For powerful, customizable dashboards and visualization of your Prometheus metrics.
  * **Implementing advanced PromQL queries** for deeper insights and custom dashboards.
  * **Configuring robust alerting rules** with Alertmanager for automated notifications (email, Slack, PagerDuty).
  * **Exploring other Prometheus exporters** for specific applications (e.g., MySQL Exporter, Blackbox Exporter for HTTP/TCP checks).

-----

### **Important Notes and Troubleshooting**

  * **Firewall:** The most common issue is a firewall blocking port 9100 on the **Target Linux Server** or port 9090 on the **Prometheus Server**. Double-check your `ufw` or `firewalld` settings and any cloud provider security groups.
  * **IP Addresses:** Ensure you use the correct IP addresses:
      * `http://<your-target-server-ip>:9100/metrics` to check Node Exporter directly.
      * `targets: ['<your-target-server-ip>:9100']` in `prometheus.yml` refers to the **target server's IP**.
      * `http://<prometheus-server-ip>:9090` to access the Prometheus UI.
  * **Prometheus Restart:** After modifying `prometheus.yml`, you *must* restart the Prometheus service for changes to take effect. Always validate your YAML syntax before restarting (`promtool check config /etc/prometheus/prometheus.yml`).
  * **Systemd Service:** If Node Exporter isn't running, check its systemd status (`sudo systemctl status node_exporter`) and logs (`sudo journalctl -u node_exporter`).
  * **Permissions:** Ensure the `node_exporter` binary has execute permissions and that the `nobody` user can read necessary system files (which it typically can by default for common metrics).
  * **Time Synchronization:** Ensure both your Prometheus server and the target Linux server have their clocks synchronized (e.g., using NTP). Time discrepancies can lead to inaccurate metric collection and graphing issues.
  * **Security:** For production environments, consider securing Node Exporter by limiting access to port 9100 to only your Prometheus server's IP address. If exposing Node Exporter over the internet, enable TLS/HTTPS.
