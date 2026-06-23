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
<img width="975" height="543" alt="image" src="https://github.com/user-attachments/assets/6bf529f7-5386-4351-89a2-da741072ec41" />


### 2. Package Installation

Installed the `.deb` package using the `dpkg` package manager.

```bash
sudo dpkg -i splunk-8.4.0-7984d6904909-linux-amd64.deb

```

<img width="975" height="543" alt="image" src="https://github.com/user-attachments/assets/2f2a9a49-6dce-44b3-a03d-0d06ed02250a" />


### 3. Initial Configuration and Service Start

Navigated to the Splunk `bin` directory to start the service, accept the license agreement, and configure the initial administrative credentials.

```bash
cd /opt/splunk/bin
sudo ./splunk start --accept-license

```

<img width="975" height="543" alt="image" src="https://github.com/user-attachments/assets/5cc5f85d-e091-4466-b21a-8025d4321953" />


Once the service is successfully running, the Splunk Web interface becomes accessible at `[http://127.0.0.1:8000]`.

---

## Phase 2: SIEM Configuration

To prepare the SIEM to aggregate data from external endpoints, specific receiving ports and data indexes must be configured.

### 1. Enabling the Receiving Port

1. Navigated to **Settings** > **Forwarding and receiving**.
2. Selected **Configure receiving** > **Add new**.
3. Configured Splunk to listen on port `9997` (the default port for Splunk-to-Splunk communication).

<img width="975" height="543" alt="image" src="https://github.com/user-attachments/assets/9f1ab06b-8b0a-4a3a-8ebc-4da72e10231b" />

<img width="975" height="544" alt="image" src="https://github.com/user-attachments/assets/9485aeee-bc36-41e7-aeb4-7af5aa792703" />

<img width="975" height="542" alt="image" src="https://github.com/user-attachments/assets/da57750c-e510-4959-aca8-54b346c1c936" />



### 2. Creating a Dedicated Index

To maintain data hygiene and optimize search performance, a specific index was created for the incoming Windows telemetry.

1. Navigated to **Settings** > **Indexes** > **New Index**.
2. Created an index named `win_log`.
3. Left all other settings (Home path, Cold path, Thawed path) at their default configurations.

<img width="975" height="543" alt="image" src="https://github.com/user-attachments/assets/5846c301-4e20-4630-8b41-ad69c329f37e" />



---

## Phase 3: Universal Forwarder Setup (Windows Endpoint)

The Splunk Universal Forwarder acts as the lightweight agent responsible for securely transmitting local endpoint logs to the Splunk Indexer.

### 1. Downloading the Forwarder

The `.msi` installer was fetched directly via PowerShell using `wget`.

<img width="942" height="544" alt="image" src="https://github.com/user-attachments/assets/0bbd0009-56d5-4939-9c67-1cbccc9971a8" />


### 2. Installation Steps

The Universal Forwarder was installed using the Windows GUI wizard with the following configurations:


* inputs (left blank for manual configuration).
* 
  <img width="249" height="195" alt="image" src="https://github.com/user-attachments/assets/f270d81c-cd35-4aea-9023-6d2cb5fd3761" />

* Created admin credentials (`socadmin`).
* 
  <img width="247" height="193" alt="image" src="https://github.com/user-attachments/assets/470b0ea8-361b-4971-a389-53b33e66c7b9" />

* Bypassed the Deployment Server step (left blank for manual configuration).
* 
  <img width="248" height="194" alt="image" src="https://github.com/user-attachments/assets/b5821504-fed9-4afb-83a9-484d91d7dc62" />


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

<img width="975" height="549" alt="image" src="https://github.com/user-attachments/assets/19ec2941-10f0-4a30-8510-ef764ca952d4" />


### Restarting the Forwarder Service

To apply the new configurations, the Universal Forwarder service was restarted via PowerShell:

```powershell
cd 'C:\Program Files\SplunkUniversalForwarder\bin'
.\splunk restart

```

<img width="975" height="545" alt="image" src="https://github.com/user-attachments/assets/2744d180-acc3-4ab9-8d5e-05a66483c7cb" />


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

  <img width="975" height="544" alt="image" src="https://github.com/user-attachments/assets/97058757-32e7-4397-86a4-bdaa01787298" />


The presence of these logs confirms the successful architecture and deployment of the log aggregation pipeline, establishing a robust foundation for building SOC correlation rules and threat hunting dashboards.
