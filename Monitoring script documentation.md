# Data Processing and Monitoring Script Documentation

## Table of Contents

1. Introduction  
2. System Overview  
3. Pre-requisites  
4. Script Setup and Configuration  
   - 4.1. Directory Structure  
   - 4.2. Configuration File  
   - 4.3. Python Data Processing Script  
   - 4.4. Systemd Service Setup  
5. Data Pipeline and Grafana Integration  
6. Testing and Validation  
7. Troubleshooting  
8. Best Practices and Recommendations  
9. Conclusion  

---

## 1. Introduction

This documentation outlines the setup and management of a **Data Processing Script** designed to collect, process, and forward key system performance metrics for real-time visualization in Grafana. The script gathers performance data, transforms and aggregates metrics, and sends them to a time-series database (e.g., InfluxDB or Prometheus), which Grafana then uses to generate dynamic dashboards for monitoring.

---

## 2. System Overview

The system monitoring solution is comprised of the following components:

- **Data Collection:**  
  The Python script monitors system metrics such as CPU load, memory usage, disk I/O, and network statistics in real time.

- **Data Processing:**  
  Collected metrics are parsed, aggregated, and transformed into a format suitable for storage in a time-series database.

- **Data Forwarding:**  
  Processed metrics are sent to a database (e.g., InfluxDB) via its REST API or a direct connector.

- **Visualization:**  
  Grafana queries the time-series database to provide real-time, interactive dashboards that display performance trends and alert conditions.

- **Automation and Reliability:**  
  The script runs as a daemon under `systemd` for continuous operation, ensuring metrics are processed and forwarded without manual intervention.

---

## 3. Pre-requisites

Before deploying the data processing script, ensure the following requirements are met:

- **Operating System:**  
  A Unix-like OS (e.g., Linux).

- **Python 3:**  
  Installed on the system.

- **Required Python Packages:**  
  Install necessary packages including `psutil` for system metrics, `requests` for HTTP API calls, and `watchdog` (if file-based triggers are required). For example:
  ```bash
  sudo apt-get update
  sudo apt-get install python3-pip -y
  pip3 install psutil requests watchdog
  ```

- **Time-Series Database:**  
  An instance of InfluxDB or Prometheus is required. Ensure it is installed and configured for receiving metrics.

- **Grafana:**  
  Grafana is installed and configured to query your time-series database.

- **Network Connectivity:**  
  Ensure proper network connectivity between the data processing server and the time-series database.

- **Permissions:**  
  Sudo or root privileges may be required to access certain system metrics or to configure systemd services.

---

## 4. Script Setup and Configuration

### 4.1. Directory Structure

Organize the script files and configuration as follows:

**Data Processing Script Directory:**

```bash
/etc/data_monitoring/
├── data_processing.py           # Main Python script
├── config_data_processing.json  # Configuration file
└── data_processing.log          # Log file for the script
```

### 4.2. Configuration File

**Path:** `/etc/data_monitoring/config_data_processing.json`

This configuration file contains parameters for metric collection intervals, database endpoints, and logging details. An example configuration:

```json
{
    "collection_interval": 10,
    "metrics": {
        "cpu": true,
        "memory": true,
        "disk": true,
        "network": true
    },
    "database": {
        "type": "influxdb",
        "url": "http://127.0.0.1:8086/write?db=system_metrics",
        "auth": {
            "username": "admin",
            "password": "secret"
        }
    },
    "logging": {
        "log_file": "/etc/data_monitoring/data_processing.log",
        "log_level": "INFO"
    }
}
```

**Notes:**

- **`collection_interval`:** Frequency (in seconds) to collect system metrics.
- **`metrics`:** Toggle collection for various performance metrics.
- **`database`:** Specify the target database type (e.g., InfluxDB) and connection details.
- **`logging`:** Configures the log file path and verbosity level.

### 4.3. Python Data Processing Script

**Path:** `/etc/data_monitoring/data_processing.py`

Below is a sample Python script that collects system metrics, processes them, and forwards the data to the specified time-series database:

```python
#!/usr/bin/env python3

import os
import time
import json
import psutil
import logging
import requests
from datetime import datetime

# ---------------------------
# Load Configuration
# ---------------------------
CONFIG_FILE = "/etc/data_monitoring/config_data_processing.json"

def load_config(config_path):
    if not os.path.exists(config_path):
        raise FileNotFoundError(f"Configuration file {config_path} not found.")
    with open(config_path, 'r') as f:
        return json.load(f)

config = load_config(CONFIG_FILE)
COLLECTION_INTERVAL = config.get("collection_interval", 10)
METRICS_CONFIG = config.get("metrics", {})
DATABASE_CONFIG = config.get("database", {})

# ---------------------------
# Setup Logging
# ---------------------------
LOGGING_CONFIG = config.get("logging", {})
LOG_FILE = LOGGING_CONFIG.get("log_file", "/etc/data_monitoring/data_processing.log")
LOG_LEVEL = LOGGING_CONFIG.get("log_level", "INFO").upper()

logging.basicConfig(
    filename=LOG_FILE,
    level=getattr(logging, LOG_LEVEL, logging.INFO),
    format='%(asctime)s - %(levelname)s - %(message)s'
)

# ---------------------------
# Helper Functions to Collect Metrics
# ---------------------------
def collect_cpu_metrics():
    cpu_percent = psutil.cpu_percent(interval=None)
    return f"cpu_load value={cpu_percent}"

def collect_memory_metrics():
    memory = psutil.virtual_memory()
    return f"memory_usage value={memory.percent}"

def collect_disk_metrics():
    disk = psutil.disk_usage('/')
    return f"disk_usage value={disk.percent}"

def collect_network_metrics():
    net_io = psutil.net_io_counters()
    return f"network_sent value={net_io.bytes_sent}, network_recv value={net_io.bytes_recv}"

# ---------------------------
# Function to Format Data for InfluxDB
# ---------------------------
def format_metrics():
    timestamp = int(datetime.utcnow().timestamp() * 1e9)  # InfluxDB expects nanoseconds
    lines = []
    if METRICS_CONFIG.get("cpu", False):
        lines.append(f"system_metrics,metric=cpu {collect_cpu_metrics()} {timestamp}")
    if METRICS_CONFIG.get("memory", False):
        lines.append(f"system_metrics,metric=memory {collect_memory_metrics()} {timestamp}")
    if METRICS_CONFIG.get("disk", False):
        lines.append(f"system_metrics,metric=disk {collect_disk_metrics()} {timestamp}")
    if METRICS_CONFIG.get("network", False):
        lines.append(f"system_metrics,metric=network {collect_network_metrics()} {timestamp}")
    return "\n".join(lines)

# ---------------------------
# Function to Forward Data to Database
# ---------------------------
def send_metrics(data):
    db_url = DATABASE_CONFIG.get("url")
    auth = DATABASE_CONFIG.get("auth", {})
    try:
        response = requests.post(db_url, data=data, auth=(auth.get("username"), auth.get("password")))
        if response.status_code == 204:
            logging.info("Metrics sent successfully.")
        else:
            logging.error(f"Failed to send metrics. Status: {response.status_code}, Response: {response.text}")
    except Exception as e:
        logging.error(f"Error sending metrics: {e}")

# ---------------------------
# Main Processing Loop
# ---------------------------
def main():
    logging.info("Starting Data Processing Script for System Monitoring.")
    while True:
        try:
            # Collect and format metrics
            data = format_metrics()
            logging.debug(f"Collected Metrics:\n{data}")
            # Forward data to time-series database
            send_metrics(data)
        except Exception as e:
            logging.error(f"Error during metric processing: {e}")
        time.sleep(COLLECTION_INTERVAL)

if __name__ == "__main__":
    main()
```

**Notes:**

- The script uses **psutil** to collect system metrics.
- Metrics are formatted using InfluxDB's line protocol, including a timestamp in nanoseconds.
- Data is sent to the database endpoint using HTTP POST with basic authentication.
- Logging captures both debug and error messages for troubleshooting.

### 4.4. Systemd Service Setup

**Path:** `/etc/systemd/system/data_processing.service`

Create a systemd service file to run the data processing script continuously:

```ini
[Unit]
Description=Data Processing and Monitoring Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /etc/data_monitoring/data_processing.py
Restart=on-failure
RestartSec=30
User=root
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

**Setup Steps:**

1. Create the service file:
   ```bash
   sudo nano /etc/systemd/system/data_processing.service
   ```
2. Paste the above content and save.
3. Reload systemd and enable the service:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable data_processing.service
   ```
4. Start the service:
   ```bash
   sudo systemctl start data_processing.service
   ```
5. Verify the service status:
   ```bash
   sudo systemctl status data_processing.service
   ```

---

## 5. Data Pipeline and Grafana Integration

- **Data Ingestion:**  
  The script pushes formatted metrics to your time-series database (e.g., InfluxDB) using its HTTP API.

- **Grafana Dashboard:**  
  In Grafana, configure a data source pointing to your database and create dashboards that display key metrics (CPU, memory, disk, network). Utilize Grafana panels to visualize trends over time, set alerts, and drill down into specific metric details.

- **Real-time Visualization:**  
  The collection interval (configured in `config_data_processing.json`) defines how frequently metrics are updated. Grafana’s dashboard refresh rate can be adjusted to reflect near-real-time data.

---

## 6. Testing and Validation

To ensure the system is working correctly, follow these tests:

### 6.1. Verify Data Collection

- Manually run the script:
  ```bash
  sudo /usr/bin/python3 /etc/data_monitoring/data_processing.py
  ```
- Check the log file (`/etc/data_monitoring/data_processing.log`) for successful metric collection and data formatting messages.

### 6.2. Validate Database Ingestion

- Query your time-series database (e.g., using InfluxDB CLI or web interface) to confirm that metrics are being stored.
- Example InfluxDB query:
  ```sql
  SELECT * FROM system_metrics LIMIT 10
  ```

### 6.3. Test Grafana Visualization

- Open Grafana and verify that the dashboard panels are displaying updated metrics.
- Adjust refresh intervals and test alerts if configured.

---

## 7. Troubleshooting

If issues are encountered, follow these steps:

### 7.1. Check Service Status

- Verify the service is running without errors:
  ```bash
  sudo systemctl status data_processing.service
  ```

### 7.2. Review Script Logs

- Monitor the log file for errors:
  ```bash
  sudo tail -f /etc/data_monitoring/data_processing.log
  ```

### 7.3. Validate Configuration Settings

- Ensure the configuration file `/etc/data_monitoring/config_data_processing.json` contains correct paths and database connection details.
- Check that the time-series database is reachable from the host running the script.

### 7.4. Test Manual Data Forwarding

- Use a tool like `curl` or Postman to simulate data ingestion into your database and compare the response with what the script logs indicate.

---

## 8. Best Practices and Recommendations

- **Regular Monitoring:**  
  Monitor the service and log files to detect and resolve issues promptly.

- **Security:**  
  Secure access to configuration files and ensure sensitive data (like database credentials) are protected.

- **Scalability:**  
  If handling high-frequency metrics, consider batch processing or message queuing to optimize data throughput.

- **Grafana Dashboards:**  
  Regularly update dashboards and alerts based on evolving monitoring needs.

- **Documentation:**  
  Keep documentation and configuration files under version control for audit and rollback purposes.

