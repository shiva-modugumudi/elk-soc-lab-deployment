# ELK SOC Lab Deployment Guide

[Back to project overview](../README.md)

**Validation status:** Deployment has not been tested end to end. Confirm package availability and complete every checkpoint before marking the build successful.


A new standalone guide for building a small lab and confirming that Windows security telemetry reaches Elasticsearch and can be searched in Kibana.

Prepared 16 October 2026  |  Beginner walkthrough  |  New document

### What you will build

One Ubuntu server runs Elasticsearch, Kibana and Fleet Server. One Windows endpoint runs Sysmon and Elastic Agent. Fleet manages the agent; the Windows agent sends event data directly to Elasticsearch. Kibana searches and displays that data. Logstash is not required for this design.

### Scope and verification

This is a newly written lab procedure informed by the supplied guide and official Elastic and Microsoft documentation. The supplied document is unchanged. Commands have been reviewed against documentation; this procedure has not been executed on the reader’s virtual machines. Screenshots and successful deployment results must come from the actual build.

Baseline for this new build: Ubuntu Server 24.04 LTS x86_64, Windows 11 x64 and Elastic Stack 9.5.5. The source document used 9.5.2; this new baseline is a deliberate change. Use the same version for Elasticsearch, Kibana, Fleet Server and Windows Elastic Agent. Check availability before installation; do not silently mix versions.

### Read this before starting

Use an isolated private lab network with internet access for downloads. This quick start uses HTTP for the Kibana browser connection and a self-signed Fleet certificate. It is a learning setup. Before using organizational infrastructure, use approved certificates, browser HTTPS, restricted access and the organization’s change process.

### How to follow the guide

Complete the steps in order. Each page identifies where to work, the command or UI action, and a checkpoint. Replace example addresses with your own throughout. Stop at any failed checkpoint and use the troubleshooting page before continuing.

## 1 Prepare the two virtual machines

Where: your hypervisor, such as VMware Workstation, VMware Fusion or an equivalent x86_64 virtualization platform. These commands target x86_64 guests; an Apple Silicon Mac needs a compatible architecture and different downloads.

| Machine | Suggested lab resources | Example address |
| --- | --- | --- |
| elk-lab | Ubuntu Server 24.04 LTS; 4 vCPU; 12 GB RAM; 80 GB disk | 192.168.190.137 |
| win-lab | Windows 11 x64; 2–4 vCPU; 4–8 GB RAM; 64 GB disk | 192.168.190.138 |

These are suggested allocations, not vendor minimum requirements. Allow additional memory for the host OS; 24 GB host RAM is a practical starting point for both VMs. The original 8 GB Ubuntu allocation may be tight when all server components share a VM.

Create each VM, attach the official OS installation ISO and finish OS setup. Give both machines the same private NAT virtual network. NAT must allow guest-to-guest traffic and the host browser must be able to reach Ubuntu. Reserve their DHCP addresses so they remain stable. Do not use bridged exposure unless you understand the network access implications.

### Record your actual values

Ubuntu IP: __________________  Windows IP: __________________
Elastic version: __________________  Build date: __________________

### Ubuntu terminal

```text
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl wget gnupg ca-certificates nano openssl
hostname -I
ip route
timedatectl status
```

Checkpoint: Ubuntu has the recorded IP, a default route and correct time. Restart after OS updates if requested, then reconnect. On Windows, run ipconfig and confirm its address is on the same subnet. Take a clean snapshot of each VM.

### Required connections

| Source | Destination | Purpose |
| --- | --- | --- |
| Host browser | Ubuntu TCP 5601 | Kibana web UI |
| Windows agent | Ubuntu TCP 8220 | Fleet management over HTTPS |
| Windows agent | Ubuntu TCP 9200 | Event ingestion over HTTPS |
| Ubuntu and Windows | Internet TCP 443 | Official packages and updates |

Fleet Server also reaches Elasticsearch locally. Kibana needs epr.elastic.co over HTTPS for integrations. No internet inbound port forwarding is needed.

## 2 Install matching Elastic packages

Where: Ubuntu terminal. Run each command in order. The repository definition below must occupy one line.

```text
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
  sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/9.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/elastic-9.x.list
sudo apt update
apt-cache madison elasticsearch
apt-cache madison kibana
```

Checkpoint: both package lists contain 9.5.5. If either is absent, stop and choose an available matching supported release using the official release documentation. Update every versioned command in this guide together.

```text
sudo apt install -y elasticsearch=9.5.5 kibana=9.5.5
sudo apt-mark hold elasticsearch kibana
dpkg-query -W elasticsearch kibana
```

Expected: both installed packages report 9.5.5. Holding packages keeps this lab reproducible during general OS updates; plan explicit upgrades later rather than leaving packages frozen indefinitely.

### Check the virtual memory setting

```text
sysctl vm.max_map_count
```

If the value is below 1048576, apply:

```text
echo "vm.max_map_count=1048576" | sudo tee /etc/sysctl.d/99-elk-lab.conf
sudo sysctl --system
```

### Prepare the single Elasticsearch node

```text
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Set the following keys once, editing existing entries where present. Keep the automatically generated security and certificate settings. Comment out cluster.initial_master_nodes if it exists because this build uses single-node discovery. Do not disable security or overwrite the whole file.

```text
cluster.name: elk-lab
node.name: elkserver
discovery.type: single-node
http.host: 0.0.0.0
```

In nano: Ctrl+O, Enter saves; Ctrl+X exits. YAML keys must not be duplicated. Keep the automatically generated xpack.security entries intact. If auto security setup was skipped and certificates are absent on first start, stop and follow the official Debian installation guidance rather than inventing certificate paths.

## 3 Start and validate Elasticsearch

Where: Ubuntu terminal. Starting the first time may take several minutes.

```text
sudo systemctl daemon-reload
sudo systemctl enable --now elasticsearch
sudo systemctl status elasticsearch --no-pager
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -i
```

Choose and privately store a strong elastic password. The password is for the initial lab administrator login, not for embedding in agent configuration or documentation.

```text
sudo ls -l /etc/elasticsearch/certs/http_ca.crt
sudo curl --cacert /etc/elasticsearch/certs/http_ca.crt \
  -u elastic https://localhost:9200
sudo curl --cacert /etc/elasticsearch/certs/http_ca.crt \
  -u elastic "https://localhost:9200/_cluster/health?pretty"
```

Enter the password when prompted. Expected: JSON showing node/cluster details and a health response. A single-node cluster may be yellow because replica shards cannot be placed on a second node. Red health is a failure to investigate.

### Permit only the lab connections

Check the Ubuntu firewall first. The example rules below assume the example addresses. Replace them with your actual Windows and host addresses. If working over SSH, permit your actual management IP on TCP 22 before enabling the firewall so you do not lock yourself out.

```text
sudo ufw status
sudo ufw allow from 192.168.190.138 to any port 9200 proto tcp
sudo ufw allow from 192.168.190.138 to any port 8220 proto tcp
```

For the browser, replace HOST_IP below with the host computer’s private IP on the lab virtual network; do not run the placeholder literally.

```text
sudo ufw allow from HOST_IP to any port 5601 proto tcp
sudo ufw enable
sudo ufw status numbered
```

If the Windows VM will also open Kibana, add a TCP 5601 rule for its actual address. Apply equivalent allow rules to any upstream lab firewall. Port 9300 is unnecessary between these two machines in this single-node lab.

### If the service fails

```text
sudo journalctl -u elasticsearch -n 100 --no-pager
sudo tail -n 100 /var/log/elasticsearch/elk-lab.log
free -h
df -h
```

Checkpoint: Elasticsearch is active, its HTTPS API responds using the CA certificate, and lab-specific firewall rules are in place.

## 4 Configure and enroll Kibana

Where: Ubuntu terminal, then your host browser. Use enrollment to configure Kibana authentication and certificate trust instead of manually inserting an incomplete service token.

```text
sudo /usr/share/kibana/bin/kibana-encryption-keys generate
sudo nano /etc/kibana/kibana.yml
```

Set server.host to the Ubuntu IP. Add the generated encryption settings once using the actual generated values, not the placeholder text below. Keep the keys private and retain them across restarts.

```text
server.host: "192.168.190.137"
xpack.encryptedSavedObjects.encryptionKey: "REPLACE_WITH_GENERATED_VALUE"
xpack.reporting.encryptionKey: "REPLACE_WITH_GENERATED_VALUE"
xpack.security.encryptionKey: "REPLACE_WITH_GENERATED_VALUE"
```

```text
sudo systemctl enable --now kibana
sudo systemctl status kibana --no-pager
sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```

Open http://192.168.190.137:5601 in the browser. Paste the freshly generated Kibana enrollment token and select Configure Elastic. If a verification code is requested, obtain it on Ubuntu:

```text
sudo /usr/share/kibana/bin/kibana-verification-code
```

Finish setup, then sign in with username elastic and the password set on the previous page. The enrollment token and the elastic password are different credentials. If enrollment expires, generate a new enrollment token.

### Checkpoint

Kibana loads after login and no longer shows its setup screen. If it remains unavailable, run:

```text
sudo journalctl -u kibana -n 100 --no-pager
sudo ss -lntp | grep -E ":(5601|9200)"
```

A Kibana service being active does not alone prove it is ready; wait for the browser to load successfully. Do not configure elasticsearch.username as elastic for Kibana’s internal connection. Enrollment handles that service authentication.

### Certificate fingerprint for agent data output

```text
sudo openssl x509 -in /etc/elasticsearch/certs/http_ca.crt \
  -noout -fingerprint -sha256
```

Copy the SHA-256 fingerprint for the next page, removing the label and colon separators. It should contain 64 hexadecimal characters. A CA fingerprint is public trust metadata; passwords and enrollment/service tokens are secrets.

## 5 Install Fleet Server on Ubuntu

Where: Kibana Fleet screens and Ubuntu terminal. Fleet Server runs inside Elastic Agent on Ubuntu; do not install a second Ubuntu agent alongside it.

### Set the agent output first

In Kibana open Management → Fleet → Settings → Outputs. Edit the default Elasticsearch output. Set Hosts to https://192.168.190.137:9200, replacing the IP. Set Elasticsearch CA trusted fingerprint to the SHA-256 value from page 4. Save. Do not leave localhost as the output: localhost on the Windows endpoint means Windows itself.

The Elasticsearch certificate must include the chosen address in its subject alternative names. If agents report an IP or hostname certificate mismatch, use a reachable name/address that is in the certificate, or reissue the certificate with the correct names. Do not disable Elasticsearch certificate validation to work around this.

### Create the Fleet Server policy

Open Fleet → Add Fleet Server. Select Quick Start. Name the policy ELK-Fleet-Server. Set the Fleet Server host URL to https://192.168.190.137:8220. Continue until Kibana generates an installation command and a Fleet service token. UI labels may vary slightly by version.

```text
mkdir -p ~/fleet-install
cd ~/fleet-install
curl -fLO https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-9.5.5-linux-x86_64.tar.gz
tar xzf elastic-agent-9.5.5-linux-x86_64.tar.gz
cd elastic-agent-9.5.5-linux-x86_64
```

Run the generated sudo ./elastic-agent install command from this directory. Preserve the real token and policy ID generated by your own Kibana. Confirm it includes --install-servers for Elastic Agent 9.x Fleet Server. The following is a structure example: replace both placeholders before running it.

```text
sudo ./elastic-agent install --install-servers \
  --fleet-server-es=https://192.168.190.137:9200 \
  --fleet-server-service-token=REPLACE_WITH_FLEET_SERVICE_TOKEN \
  --fleet-server-policy=REPLACE_WITH_FLEET_POLICY_ID \
  --fleet-server-es-ca=/etc/elasticsearch/certs/http_ca.crt \
  --fleet-server-port=8220
```

This uses the local Elasticsearch CA file and auto-generates a self-signed Fleet certificate. Use the generated command as the authority for your real policy/token values. Accept installation when prompted.

```text
sudo elastic-agent status
sudo ss -lntp | grep :8220
```

Checkpoint: Fleet Server is Healthy in Kibana and Ubuntu listens on 8220. Do not continue while it is unhealthy.

## 6 Prepare Sysmon and PowerShell logs

Where: Windows VM. Open Windows PowerShell 5.1 using Run as administrator. These registry paths and event channels target Windows PowerShell 5.1, not PowerShell 7.

### Install Sysmon from Microsoft

Download Sysmon from https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon and extract its ZIP into C:\Tools\Sysmon. Check the executable’s Digital Signatures tab for Microsoft before running it.

```text
Set-Location C:\Tools\Sysmon
.\Sysmon64.exe -accepteula -i
Get-Service Sysmon64
Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 3
```

Checkpoint: the Sysmon64 service is Running and local Sysmon events exist. This basic installation is sufficient for process creation validation. Other event types such as network connections need an appropriate Sysmon XML configuration; do not claim all event types are enabled by default.

### Enable script block logging and module logging

```text
$base = 'HKLM:\Software\Policies\Microsoft\Windows\PowerShell'
New-Item -Path "$base\ScriptBlockLogging" -Force | Out-Null
New-ItemProperty -Path "$base\ScriptBlockLogging" `
  -Name EnableScriptBlockLogging -PropertyType DWord -Value 1 -Force
New-Item -Path "$base\ModuleLogging" -Force | Out-Null
New-ItemProperty -Path "$base\ModuleLogging" `
  -Name EnableModuleLogging -PropertyType DWord -Value 1 -Force
New-Item -Path "$base\ModuleLogging\ModuleNames" -Force | Out-Null
New-ItemProperty -Path "$base\ModuleLogging\ModuleNames" `
  -Name '*' -PropertyType String -Value '*' -Force
```

The backtick at a PowerShell line end continues the command; do not add spaces after it. Close and reopen Windows PowerShell as administrator, then generate a harmless test:

```text
Write-Output 'ELK-LAB-VALIDATION'
Get-WinEvent -FilterHashtable @{
  LogName='Microsoft-Windows-PowerShell/Operational'; Id=4104
} -MaxEvents 3
```

Checkpoint: a recent event 4104 exists locally. Module logging uses event 4103. Script block logs can contain sensitive script content, so use harmless test data and protect the resulting logs.

## 7 Enroll Windows and collect event channels

Where: Kibana, then Windows PowerShell as administrator.

### Create a separate Windows policy

Open Fleet → Agent policies → Create agent policy. Name it Windows-Lab-Policy. Add the Windows integration to this policy, enabling PowerShell, PowerShell Operational and Sysmon Operational streams. Confirm the channel names match the three channels below. Add the System integration with Windows Application, System and Security log collection enabled if you also want basic Windows event logs.

```text
Windows PowerShell
Microsoft-Windows-PowerShell/Operational
Microsoft-Windows-Sysmon/Operational
```

Save the integration and verify it belongs to Windows-Lab-Policy. Never attach this Windows collection configuration only to the Ubuntu Fleet Server policy.

### Check Windows can reach both required services

```text
Test-NetConnection 192.168.190.137 -Port 8220
Test-NetConnection 192.168.190.137 -Port 9200
```

Expected: TcpTestSucceeded is True for both. The Windows agent sends event data to 9200 directly; a successful 8220 connection alone is insufficient.

### Download and install the Windows agent

In Fleet select Add agent, choose Windows-Lab-Policy, then Windows. Copy the generated download and install instructions. Confirm the package version is 9.5.5. You can download and extract it as follows:

```text
New-Item -ItemType Directory C:\Tools -Force | Out-Null
Set-Location C:\Tools
$uri = 'https://artifacts.elastic.co/downloads/beats/' + `
  'elastic-agent/elastic-agent-9.5.5-windows-x86_64.zip'
Invoke-WebRequest $uri -OutFile elastic-agent.zip
Expand-Archive .\elastic-agent.zip -DestinationPath . -Force
Set-Location .\elastic-agent-9.5.5-windows-x86_64
```

Use the actual enrollment token generated for Windows-Lab-Policy in this command. Quick Start Fleet uses a self-signed certificate; --insecure below bypasses verification of that Fleet certificate only. Elasticsearch output must still use the CA fingerprint configured on page 5.

```text
.\elastic-agent.exe install `
  --url=https://192.168.190.137:8220 `
  --enrollment-token=REPLACE_WITH_WINDOWS_POLICY_TOKEN `
  --insecure
Get-Service -DisplayName "Elastic Agent"
& "C:\Program Files\Elastic\Agent\elastic-agent.exe" status
```

Checkpoint: Windows agent is Healthy in Fleet and assigned to Windows-Lab-Policy. For organizational use, replace Quick Start trust with an approved Fleet CA and remove --insecure.

## 8 Prove that the lab works

Where: Windows PowerShell, then Kibana Discover. A healthy agent is necessary, but searchable event data is the final evidence.

### Generate harmless activity

```text
Start-Process notepad.exe
powershell.exe -NoProfile -Command "Write-Output 'ELK-LAB-VALIDATION'"
```

Note the current time and Windows hostname. In Kibana open Discover. Select a logs data view that covers logs-*; create that data view with @timestamp if it does not already exist. Set Last 15 minutes, then refresh after allowing a few minutes for ingestion.

### Search using KQL

```text
host.name : "YOUR_WINDOWS_HOSTNAME"

winlog.channel : "Microsoft-Windows-Sysmon/Operational"
  and event.code : "1"

winlog.channel : "Microsoft-Windows-PowerShell/Operational"
  and event.code : "4104"
```

Run each query separately. Replace YOUR_WINDOWS_HOSTNAME with the value from Windows hostname. Add @timestamp, host.name, event.code, winlog.channel, process.name and message as visible columns where available.

### Expected evidence

| Check | Passing evidence |
| --- | --- |
| Process telemetry | Recent Sysmon event 1 for notepad.exe on the Windows host |
| PowerShell telemetry | Recent event 4104 containing the harmless validation command |
| Policy and agent | Windows endpoint Healthy under Windows-Lab-Policy |
| Persistence | After VM reboot, services return and new test events arrive |

Optional: verify module logging event 4103 separately after running a module command.

### Build record to complete after deployment

Versions and IPs: __________________
Validation date and time: __________________
Sysmon 1 / PowerShell 4104 / reboot checks passed: __________________
Remaining issues: __________________

Capture your own screenshots of VM resources, package versions, Elasticsearch health, both Fleet agents and the two Discover results. Redact passwords and tokens. Do not label this guide tested until those checks have actually passed. Take a working snapshot after validation.

## 9 Troubleshoot one layer at a time

### Ubuntu commands for service and resource failures

```text
sudo systemctl status elasticsearch kibana elastic-agent --no-pager
sudo journalctl -u elastic-agent -n 100 --no-pager
sudo elastic-agent status
free -h
df -h
sudo ss -lntp | grep -E ":(5601|8220|9200)"
```

If Elasticsearch is killed by memory pressure, increase Ubuntu RAM or reduce other workload. If disk is nearly full, free space and investigate ingestion/retention before restarting repeatedly. Do not delete Elasticsearch data directories as a routine fix.

### Kibana cannot open

Check the Ubuntu address, TCP 5601 allow rule and Kibana logs. An active service may still be initializing. If enrollment fails, generate a fresh Kibana enrollment token and confirm Elasticsearch responds with the CA certificate.

### Windows cannot enroll

Test 8220 reachability. Confirm Fleet Server is Healthy and the URL begins with https:// and ends in :8220. Use a current token for Windows-Lab-Policy. A Fleet service token is not a Windows agent enrollment token. For this isolated Quick Start only, use --insecure when the failure is an untrusted Fleet self-signed certificate.

### Agent is healthy but no logs arrive

Test 9200 reachability. Check Fleet output uses Ubuntu’s address rather than localhost and has the correct Elasticsearch CA fingerprint. Review agent component status for output errors. Inspect the same events locally in Event Viewer; an agent cannot ship an event that was never generated. Check Windows integration assignment, enabled streams, Discover data view, host filter and time range.

### Certificate errors on the Elasticsearch output

Unknown authority: compare the configured fingerprint with the active Elasticsearch HTTP CA. Name mismatch: compare the output IP/hostname with the server certificate subject alternative names. Fix trust or issue the correct certificate; Fleet --insecure does not fix Elasticsearch output trust.

### Installed agent already exists

Do not repeatedly reinstall or manually delete its directory. Check its current policy and status first. If a clean reinstall is genuinely required, use Elastic’s documented uninstall process, unenroll the stale Fleet record, then install once. Keep troubleshooting logs free of exposed secrets.

### Single node cluster is yellow

Yellow can be expected because replicas need another node. Confirm primary shards are assigned and test ingestion. A red cluster, unavailable primary shard or new indexing error requires investigation.
