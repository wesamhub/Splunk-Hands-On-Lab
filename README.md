Here is a comprehensive, professional GitHub write-up based on your screenshots. It is structured to highlight your expertise in defensive security operations and SIEM architecture.

---

# Splunk SIEM Architecture & Endpoint Telemetry Forwarding Lab

## Overview

This repository documents the deployment and configuration of a localized Security Information and Event Management (SIEM) environment. The project demonstrates the foundational architecture of log aggregation, specifically focusing on collecting and parsing Windows endpoint telemetry (including Sysmon) into a centralized Splunk Enterprise instance.

This lab simulates a standard Blue Team operations environment, ensuring high-fidelity data collection for threat detection and incident response.

## Architecture

* **SIEM Server (Indexer/Search Head):** Splunk Enterprise deployed on a Linux host (Kali).
* **Target Endpoint:** Windows 10 client generating system and security events.
* **Log Forwarder:** Splunk Universal Forwarder (UF) installed on the Windows endpoint.
* **Data Sources:** Windows Event Logs (Application, Security, System) and Sysmon Operational logs.

---

## Phase 1: Splunk Enterprise Installation (Linux)

### 1. Downloading the Package

The Splunk Enterprise Debian package was retrieved via the command line using `wget`.

```bash
wget -O splunk-8.4.0-7984d6904909-linux-amd64.deb "https://download.splunk.com/products/splunk/releases/8.4.0/linux/splunk-8.4.0-7984d6904909-linux-amd64.deb"

```

### 2. Package Installation

Installed the `.deb` package using the `dpkg` package manager.

```bash
sudo dpkg -i splunk-8.4.0-7984d6904909-linux-amd64.deb

```

### 3. Initial Configuration and Service Start

Navigated to the Splunk `bin` directory to start the service, accept the license agreement, and configure the initial administrative credentials (User: `socadmin`).

```bash
cd /opt/splunk/bin
sudo ./splunk start --accept-license

```

Once the service is successfully running, the Splunk Web interface becomes accessible at `[http://127.0.0.1:8000](http://127.0.0.1:8000)`.

---

## Phase 2: SIEM Configuration

To prepare the SIEM to aggregate data from external endpoints, specific receiving ports and data indexes must be configured.

### 1. Enabling the Receiving Port

1. Navigated to **Settings** > **Forwarding and receiving**.
2. Selected **Configure receiving** > **Add new**.
3. Configured Splunk to listen on port `9997` (the default port for Splunk-to-Splunk communication).

### 2. Creating a Dedicated Index

To maintain data hygiene and optimize search performance, a specific index was created for the incoming Windows telemetry.

1. Navigated to **Settings** > **Indexes** > **New Index**.
2. Created an index named `win_log`.
3. Left all other settings (Home path, Cold path, Thawed path) at their default configurations.

---

## Phase 3: Universal Forwarder Setup (Windows Endpoint)

The Splunk Universal Forwarder (UF) acts as the lightweight agent responsible for securely transmitting local endpoint logs to the Splunk Indexer.

### 1. Downloading the Forwarder

The `.msi` installer was fetched directly via PowerShell using `Invoke-WebRequest`. *(Note: Any remnant services from previous installations were cleared out using `sc.exe delete SplunkForwarder` prior to the fresh install).*

### 2. GUI Installation Steps

The Universal Forwarder was installed using the Windows GUI wizard with the following configurations:

* Accepted the End User License Agreement.
* Selected **Local System** for the installation user.
* Enabled inputs for **Application, Security, and System** Event Logs.
* Created local admin credentials (`socadmin`).
* Bypassed the Deployment Server step (left blank for manual configuration).

---

## Phase 4: Forwarding Configuration (`inputs.conf` & `outputs.conf`)

Instead of relying solely on the GUI, the configuration files were manually adjusted to ensure precise routing to the `win_log` index, particularly for Sysmon telemetry.

Located in `C:\Program Files\SplunkUniversalForwarder\etc\system\local\`:

### `outputs.conf`

Configured the destination Indexer IP and port.

```ini
[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
server = 192.168.31.130:9997

```

### `inputs.conf`

Explicitly defined the logs to collect and mapped them to the newly created `win_log` index. Enabled `renderXml = true` for Sysmon logs to ensure proper parsing in Splunk.

```ini
[WinEventLog://Application]
disabled = 0
index = win_log

[WinEventLog://Security]
disabled = 0
index = win_log

[WinEventLog://System]
disabled = 0
index = win_log

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
index = win_log
renderXml = true

```

### Restarting the Forwarder Service

To apply the new configurations, the Universal Forwarder service was restarted via PowerShell:

```powershell
cd 'C:\Program Files\SplunkUniversalForwarder\bin'
.\splunk restart

```

---

## Phase 5: Telemetry Verification

With the forwarder active, verification was performed on the Splunk Enterprise search head to ensure logs were successfully ingested and categorized.

**SPL Query executed:**

```spl
index="win_log" | stats count by host, sourcetype

```

**Results Verified:**
The search successfully returned active event counts from the host `DESKTOP-HUZN3I1`, categorized by the following sourcetypes:

* `XmlWinEventLog:Application`
* `XmlWinEventLog:Security`
* `XmlWinEventLog:System`
* `XmlWinEventLog:Microsoft-Windows-Sysmon/Operational`

The presence of these logs confirms the successful architecture and deployment of the log aggregation pipeline, establishing a robust foundation for building SOC correlation rules and threat hunting dashboards.
