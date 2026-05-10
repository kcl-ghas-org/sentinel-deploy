# Windows Event Collection (WEC/WEF) to Microsoft Sentinel — Complete Guide

## Overview

This guide covers end-to-end configuration of Windows Event Forwarding (WEF) using a Windows Event Collector (WEC) server to aggregate Windows Event Logs from source servers and forward them into Microsoft Sentinel via Azure Arc and the Azure Monitor Agent (AMA).

### Data Flow

```
Source Servers (GPO or manual config)
    → WinRM (TCP 5985 / 5986 for HTTPS) →
        WEC Collector Server (ForwardedEvents log)
            → AMA reads ForwardedEvents →
                DCR routes to Sentinel workspace
                    → WindowsEvent table
```

No agents are installed on source servers. GPO handles all configuration. Only the WEC collector gets Azure Arc + AMA.

---

## Part 1 — WEC Collector Server Setup

### 1.1 Enable the Windows Event Collector Service

```powershell
wecutil qc /q
```

This starts the Windows Event Collector service (wecsvc) and configures it to start automatically.

### 1.2 Increase the ForwardedEvents Log Size

The default ForwardedEvents log is capped at approximately 20 MB. At scale, this causes immediate event loss. Increase it based on your environment:

```powershell
# Set to 4 GB
wevtutil sl ForwardedEvents /ms:4294967296

# Set to 400 GB (for large environments)
wevtutil sl ForwardedEvents /ms:429496729600

# Set to 450 GB
wevtutil sl ForwardedEvents /ms:483183820800
```

Verify the setting:

```powershell
wevtutil gl ForwardedEvents | findstr "maxSize"
```

> **Note:** 419430400 KB = 400 GB

### 1.3 Move the ForwardedEvents Log to a Dedicated Drive (Recommended)

Keeping the ForwardedEvents log off the OS drive prevents high event volume from filling C: and allows you to use faster dedicated storage for better write I/O.

```powershell
# Create the destination directory
New-Item -Path "D:\EventLogs" -ItemType Directory -Force

# Stop services
Stop-Service Wecsvc
Stop-Service EventLog

# Move the log file location
wevtutil sl ForwardedEvents /lfn:"D:\EventLogs\ForwardedEvents.evtx"

# Start services
Start-Service EventLog
Start-Service Wecsvc
```

If the wevtutil command returns "parameter is incorrect," set it via registry instead:

```powershell
Stop-Service Wecsvc
Stop-Service EventLog

Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WINEVT\Channels\ForwardedEvents" -Name "LogFileName" -Value "D:\EventLogs\ForwardedEvents.evtx"

Start-Service EventLog
Start-Service Wecsvc
```

Verify the change:

```powershell
wevtutil gl ForwardedEvents | findstr "logFileName"
```

### 1.4 Create a Subscription

You can create a subscription via the GUI or via XML. For initial setup, the GUI is quickest.

**Via GUI:**

1. Open **Event Viewer → Subscriptions → Create Subscription**
2. **Name:** descriptive name (e.g., `Security-Events`)
3. **Subscription type:** Select **Source computer initiated** (push model — required for scale)
4. **Computer Groups:** Click **Select Computer Groups → Add Domain Computers** (or specific OUs/security groups)
5. **Select Events → Edit filter:** Choose the logs and Event IDs to forward:
   - Security log: Event IDs 4624, 4625, 4648, 4688, 4720, 4732, 7045
   - Application and System logs as needed
   - Use a broad filter for initial testing (all event levels, all event IDs)
6. **Advanced:** Set delivery optimization:
   - **Normal** — general purpose, batches events
   - **Minimize Latency** — near real-time forwarding (higher resource usage)
   - **Minimize Bandwidth** — conserves network, higher latency

### 1.5 Permissions for Security Log Forwarding

On the WEC collector, add **NETWORK SERVICE** to the **Event Log Readers** group to allow the collector service to read forwarded Security events:

```powershell
net localgroup "Event Log Readers" "NETWORK SERVICE" /add
```

Restart the collector service after adding:

```powershell
Restart-Service Wecsvc
```

### 1.6 Install Azure Arc + AMA on the Collector

Onboard the WEC collector to Azure Arc using a Service Principal:

```powershell
azcmagent connect `
  --service-principal-id "<APP_ID>" `
  --service-principal-secret "<SECRET_VALUE>" `
  --resource-group "<RG_NAME>" `
  --tenant-id "<TENANT_ID>" `
  --location "<AZURE_REGION>" `
  --subscription-id "<SUBSCRIPTION_ID>"
```

**Service Principal requirements:**
- Role: **Azure Connected Machine Onboarding** scoped to the target resource group
- Use the **Secret Value** (not the Secret ID) from the App Registration → Certificates & secrets blade
- The Value is the first column after Description — the longer random string with the copy icon

### 1.7 Create the Sentinel DCR

In Sentinel, go to **Data connectors → Windows Forwarded Events via AMA → Create data collection rule**.

> **Critical:** Use the **Windows Forwarded Events via AMA** connector — NOT "Windows Security Events via AMA." They read from different logs:
> - **Windows Security Events via AMA** → reads the local Security log
> - **Windows Forwarded Events via AMA** → reads the ForwardedEvents log

On the Resources tab, select the WEC collector machine. AMA will auto-install as an extension when the DCR is associated.

---

## Part 2 — Source Server Configuration via GPO

This is what makes WEF scale — you don't touch individual servers. Everything is configured through Group Policy.

### 2.1 Create the GPO

Open **Group Policy Management** on a domain controller and create a new GPO (e.g., `WEF - Forward Events`).

### 2.2 Configure the Subscription Manager

Navigate to: **Computer Configuration → Administrative Templates → Windows Components → Event Forwarding → Configure Target Subscription Manager**

Enable it, click **Show**, and add a value:

```
Server=http://wec-1.kcl.com:5985/wsman/SubscriptionManager/WEC,Refresh=60
```

Replace the FQDN with your WEC collector's fully qualified domain name. `Refresh=60` means source servers check for subscription updates every 60 seconds.

### 2.3 Enable the WinRM Service

Navigate to: **Computer Configuration → Administrative Templates → Windows Components → Windows Remote Management (WinRM) → WinRM Service → Allow remote server management through WinRM**

Enable it. Set the IPv4/IPv6 filters to `*` or your specific subnet.

Also ensure the WinRM service starts automatically:

**Computer Configuration → Windows Settings → Security Settings → System Services → Windows Remote Management (WS-Management)** → Set to **Automatic**.

### 2.4 Configure Windows Firewall

Navigate to: **Computer Configuration → Windows Settings → Security Settings → Windows Defender Firewall with Advanced Security → Inbound Rules**

Create a rule allowing TCP 5985 inbound (or enable the predefined "Windows Remote Management" rule).

### 2.5 Link the GPO

Link the GPO to the OU containing your source servers.

### 2.6 Test on a Source Server

Force a group policy update:

```powershell
gpupdate /force
```

Verify the subscription manager was applied by checking the registry:

```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\EventLog\EventForwarding\SubscriptionManager"
```

This should return the collector URL (e.g., `Server=http://wec-1.kcl.com:5985/wsman/SubscriptionManager/WEC,Refresh=60`).

> **Important:** `wecutil er` only works on the **WEC collector**, not on source servers. Source servers do not have local subscriptions — they are configured via the SubscriptionManager registry key to push events to the collector. To verify the source is connected, check the subscription status on the **collector** using `wecutil gs "Subscription-Name"`.

Also verify WinRM is running on the source server:

```powershell
Get-Service WinRM | Select-Object Name, Status
```

And test connectivity to the collector:

```powershell
Test-NetConnection wec-1.kcl.com -Port 5985
```

---

## Part 3 — Manual Source Server Configuration (Without GPO)

For single-server testing or lab environments where GPO is not practical.

### 3.1 Enable and Start WinRM

```powershell
winrm quickconfig /q
```

### 3.2 Set the Subscription Manager

```powershell
winrm set winrm/config/client @{TrustedHosts="WEC-SERVER-FQDN"}

reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\EventLog\EventForwarding\SubscriptionManager" /v 1 /t REG_SZ /d "Server=http://WEC-SERVER-FQDN:5985/wsman/SubscriptionManager/WEC,Refresh=60" /f
```

### 3.3 Restart Services

```powershell
Restart-Service WinRM
Restart-Service EventLog
```

### 3.4 Verify

Verify the subscription manager registry key was set:

```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\EventLog\EventForwarding\SubscriptionManager"
```

Confirm WinRM is running:

```powershell
Get-Service WinRM | Select-Object Name, Status
```

Test connectivity to the collector:

```powershell
Test-NetConnection WEC-SERVER-FQDN -Port 5985
```

> **Note:** `wecutil er` only works on the WEC collector, not on source servers. To confirm the source server is forwarding, check the subscription status on the collector with `wecutil gs "Subscription-Name"` and look for the source server listed with **Active** status.

If the source server doesn't appear in the subscription status immediately, wait 60 seconds (the Refresh interval) and check again. You can also restart the EventLog service to force a check:

```powershell
Restart-Service EventLog
```

---

## Part 4 — Verification

### 4.1 Verify Events on the WEC Collector

Open Event Viewer → **Windows Logs → Forwarded Events** to visually confirm events are arriving.

Or via PowerShell:

```powershell
wevtutil qe ForwardedEvents /c:5 /f:text /rd:true
```

### 4.2 Check Subscription Status

```powershell
# List all subscriptions
wecutil es

# Check a specific subscription's status and source computers
wecutil gs "Security-Events"
```

Look for source computers showing **Active** status. **Inactive** means the source servers aren't connecting.

### 4.3 Generate Test Events

Application log test event:

```powershell
Write-EventLog -LogName Application -Source "Application" -EventID 9999 -Message "WEF test event"
```

Security log events can be triggered by locking/unlocking the workstation, logging on/off, or running:

```powershell
# Trigger a failed logon (generates Event ID 4625)
net use \\localhost /user:fakeuser fakepassword 2>$null
```

> **Delivery Latency Note:** Events may take up to 15 minutes to appear in the ForwardedEvents log on the collector depending on the delivery optimization setting:
> - **Minimize Latency** — ~30 seconds (push mode, batch timeout 30 seconds)
> - **Normal** — up to 15 minutes (pull mode, batch timeout 15 minutes)
> - **Minimize Bandwidth** — up to 6 hours (pull mode, batches aggressively)
>
> For testing and demos, set delivery optimization to **Minimize Latency** for near real-time forwarding.

### 4.4 Verify Forwarded Events in Sentinel

Once events are confirmed in the ForwardedEvents log on the collector, verify they are reaching Sentinel:

```kql
WindowsEvent
| where TimeGenerated > ago(1h)
| take 20
```

If ForwardedEvents has data but Sentinel does not, check:
1. AMA is running: `Get-Service AzureMonitorAgent` on the collector
2. The correct connector was used (Windows Forwarded Events via AMA)
3. The DCR is associated with the collector machine (Azure Arc → Extensions)

### 4.5 Retrieve Log Analytics Workspace Details

If needed for configuration or troubleshooting:

**Portal:** Log Analytics workspace → Agents (under Settings) — displays both the Workspace ID and Primary Key.

**PowerShell:**

```powershell
# Workspace ID
(Get-AzOperationalInsightsWorkspace -ResourceGroupName "your-rg" -Name "your-workspace").CustomerId

# Shared Key
(Get-AzOperationalInsightsWorkspaceSharedKeys -ResourceGroupName "your-rg" -WorkspaceName "your-workspace").PrimarySharedKey
```

**Azure CLI:**

```bash
# Workspace ID
az monitor log-analytics workspace show --resource-group "your-rg" --workspace-name "your-workspace" --query customerId -o tsv

# Shared Key
az monitor log-analytics workspace get-shared-keys --resource-group "your-rg" --workspace-name "your-workspace" --query primarySharedKey -o tsv
```

---

## Part 5 — Sentinel KQL Queries

### Health Check — Are Forwarded Events Arriving?

```kql
WindowsEvent
| where TimeGenerated > ago(1h)
| summarize EventCount = count(), LastEvent = max(TimeGenerated) by Computer
| sort by EventCount desc
```

### What Event IDs Are Being Forwarded?

```kql
WindowsEvent
| where TimeGenerated > ago(1h)
| summarize count() by EventID, Channel
| sort by count_ desc
| take 25
```

### Logon Events (4624)

```kql
WindowsEvent
| where TimeGenerated > ago(1h)
| where EventID == 4624
| extend LogonType = tostring(EventData.LogonType)
| extend TargetUser = tostring(EventData.TargetUserName)
| extend SourceIP = tostring(EventData.IpAddress)
| project TimeGenerated, Computer, TargetUser, LogonType, SourceIP
| sort by TimeGenerated desc
| take 20
```

### Failed Logons (4625)

```kql
WindowsEvent
| where TimeGenerated > ago(1h)
| where EventID == 4625
| extend TargetUser = tostring(EventData.TargetUserName)
| extend FailReason = tostring(EventData.Status)
| extend SourceIP = tostring(EventData.IpAddress)
| project TimeGenerated, Computer, TargetUser, FailReason, SourceIP
| sort by TimeGenerated desc
```

### New Process Creation (4688)

```kql
WindowsEvent
| where TimeGenerated > ago(1h)
| where EventID == 4688
| extend NewProcess = tostring(EventData.NewProcessName)
| extend ParentProcess = tostring(EventData.ParentProcessName)
| extend User = tostring(EventData.SubjectUserName)
| project TimeGenerated, Computer, User, NewProcess, ParentProcess
| sort by TimeGenerated desc
| take 20
```

### Service Installations (7045)

```kql
WindowsEvent
| where TimeGenerated > ago(24h)
| where EventID == 7045
| extend ServiceName = tostring(EventData.ServiceName)
| extend ImagePath = tostring(EventData.ImagePath)
| project TimeGenerated, Computer, ServiceName, ImagePath
| sort by TimeGenerated desc
```

### Account Management Events

```kql
WindowsEvent
| where TimeGenerated > ago(24h)
| where EventID in (4720, 4722, 4725, 4726, 4732, 4733)
| extend TargetUser = tostring(EventData.TargetUserName)
| extend Actor = tostring(EventData.SubjectUserName)
| project TimeGenerated, Computer, EventID, Actor, TargetUser
| sort by TimeGenerated desc
```

### Volume Trending

```kql
WindowsEvent
| where TimeGenerated > ago(4h)
| summarize count() by bin(TimeGenerated, 15m), Computer
| render timechart
```

### Stale Source Servers — Who Stopped Forwarding?

```kql
WindowsEvent
| where TimeGenerated > ago(24h)
| summarize LastSeen = max(TimeGenerated) by Computer
| where LastSeen < ago(1h)
| sort by LastSeen asc
```

---

## Part 6 — Troubleshooting

Work backwards through the chain — start at the collector and trace back to the source.

### Step 1: Is the ForwardedEvents Log Populating on the WEC Collector?

```powershell
wevtutil qe ForwardedEvents /c:5 /f:text /rd:true
```

If empty → the problem is between source servers and the collector (WEF not working). Skip to Step 4.
If events are there → the problem is between AMA and Sentinel. Continue to Step 2.

### Step 2: Is AMA Running on the Collector?

```powershell
Get-Service AzureMonitorAgent | Select-Object Name, Status
Get-Service himds | Select-Object Name, Status
```

Both should show `Running`.

### Step 3: Is the DCR Using the Correct Connector?

In the portal go to **Azure Arc → your WEC machine → Extensions** and confirm `AzureMonitorWindowsAgent` shows `Provisioning succeeded`.

Verify the DCR uses the **Windows Forwarded Events via AMA** connector (not Windows Security Events via AMA).

### Step 4: Are Source Servers Forwarding?

> **Important:** `wecutil er` does NOT work on source servers — it only works on the WEC collector. On the source server, verify the subscription manager is configured via the registry:

```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\EventLog\EventForwarding\SubscriptionManager"
```

If this returns empty or the key doesn't exist, the GPO hasn't applied:

```powershell
gpupdate /force
```

Then check the registry again. Also verify WinRM is running:

```powershell
Get-Service WinRM | Select-Object Name, Status
winrm quickconfig
```

To confirm the source server is actually connected and forwarding, check from the **WEC collector**:

```powershell
wecutil gs "Your-Subscription-Name"
```

Look for the source server in the EventSource list with **Enabled: true** and **Active** status.

### Step 5: Can the Source Server Reach the Collector?

```powershell
Test-NetConnection WEC-SERVER-FQDN -Port 5985
```

If this fails, firewall rules on either the source or collector are blocking WinRM.

### Step 6: Check the Subscription on the Collector

```powershell
wecutil es
wecutil gs "Your-Subscription-Name"
```

Look for source computer status: **Active**, **Inactive**, or **Error**.

### Step 7: Check Event Logs for Errors

On the WEC collector, open **Event Viewer → Applications and Services Logs → Microsoft → Windows → EventCollector → Operational**.

This log shows WEC service errors, subscription problems, and source connection failures.

### Quick Checklist

| Check | Command / Location |
|-------|-------------------|
| Source server subscription manager configured? | `Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\...\SubscriptionManager"` (on source) |
| WinRM running on source? | `Get-Service WinRM` (on source) |
| Port 5985 open between them? | `Test-NetConnection WEC-FQDN -Port 5985` (from source) |
| Source server showing in subscription? | `wecutil gs "Subscription-Name"` (on WEC collector) |
| ForwardedEvents populating on WEC? | `wevtutil qe ForwardedEvents /c:5 /f:text /rd:true` (on WEC collector) |
| Correct Sentinel connector used? | Windows Forwarded Events via AMA (not Security Events) |
| AMA running on WEC? | `Get-Service AzureMonitorAgent` (on WEC collector) |
| DCR associated with WEC machine? | Azure Arc → Extensions → check |

> **Common Mistake:** `wecutil er` and `wecutil es` only work on the WEC collector. Running `wecutil er` on a source server returns "Command er is not supported. Error = 0x57." This is expected — source servers do not have local subscriptions.

---

## Part 7 — WEC Collector Sizing (At Scale)

### Capacity Guidelines

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 4 cores | 8+ cores |
| RAM | 16 GB | 32 GB (WecSvc can consume 4+ GB at scale) |
| OS Disk | 60 GB SSD | 100 GB SSD |
| ForwardedEvents Log Disk | 100 GB SSD (dedicated) | 200–500 GB NVMe/SSD on separate volume |
| Network | 1 Gbps | 1 Gbps |
| OS | Windows Server 2022 | Windows Server 2022 |

### Sizing Formula

Microsoft recommends 2,000 to no more than 4,000 source endpoints per WEC collector.

```
Collectors needed = (Total source servers ÷ 3,500) × 1.25 redundancy factor
```

Examples:
- 5,000 servers: (5,000 ÷ 3,500) × 1.25 = 2 collectors
- 15,000 servers: (15,000 ÷ 3,500) × 1.25 = 6 collectors
- 20,000 servers: (20,000 ÷ 3,500) × 1.25 = 8 collectors

### Best Practices

- Group by server role — separate collectors (or subscriptions) for domain controllers vs. member servers. DCs generate 10–40x the EPS of a file server.
- Use source-initiated (push) subscriptions — collector-initiated does not scale beyond a few hundred.
- Optimize XPath filters — forward only the Event IDs needed for security detection. Can reduce volume by 50–80%.
- Monitor WecSvc — track process memory and disk queue length. When WecSvc memory exceeds 4 GB or disk queue stays elevated, add another collector.
- Use AD security group filtering on the GPO to distribute servers across collectors and rebalance load.

---

## Part 8 — Common Gotchas

- **`wecutil er` on source servers:** Returns "Command er is not supported. Error = 0x57." This is expected — `wecutil er` only works on the WEC collector. Use the registry check on source servers instead.
- **Service Principal Secret Value vs Secret ID:** When creating a Service Principal for Arc onboarding, the Certificates & secrets blade shows two columns — **Value** and **Secret ID**. Use the **Value** (first column, longer random string). The Secret ID is just a GUID identifier. The Value is only shown once at creation; if you navigate away, you must create a new secret.
- **ForwardedEvents log default size:** The default is ~20 MB. At any meaningful scale this causes immediate silent event loss. Always increase it before onboarding source servers.
- **Wrong Sentinel data connector:** "Windows Security Events via AMA" reads the local Security log. "Windows Forwarded Events via AMA" reads the ForwardedEvents log. Using the wrong one means AMA is collecting from the wrong log and you'll see no forwarded data in Sentinel.
- **AMA not installing:** AMA auto-installs as an Arc extension when a DCR is associated with the machine. If AMA doesn't appear, verify the DCR has the machine checked under Resources and that the Arc agent status is **Connected** (`azcmagent show | grep "Agent Status"`).
- **Delivery optimization latency:** With Normal delivery optimization, events can take up to 15 minutes to appear on the collector. During testing, switch to Minimize Latency for near real-time forwarding.
- **Table-level RBAC in Sentinel:** If you query a table you don't have permissions on, KQL returns an empty result set — no error, no access denied. This can be mistaken for "no data flowing" when it's actually a permissions issue.

---

## Part 9 — Generating Test Security Events and Verifying in Sentinel

### 9.1 Generate Security Events on the Source Server

Run these commands on the source server (e.g., `win-log-1.kcl.com`) to generate Security log events that will be forwarded to the WEC collector and into Sentinel.

**Failed Logon — Event ID 4625:**

```powershell
net use \\localhost /user:fakeuser fakepassword 2>$null
```

**Create and Delete a Local User — Event IDs 4720 (created) and 4726 (deleted):**

```powershell
net user testuser P@ssw0rd123 /add
net user testuser /delete
```

**Add a User to a Group — Event ID 4732:**

```powershell
net localgroup "Remote Desktop Users" administrator /add
```

**Lock the Workstation — Event IDs 4800/4801:**

```powershell
rundll32.exe user32.dll,LockWorkStation
```

**Start New Processes — Event ID 4688:**

```powershell
cmd /c whoami
cmd /c ipconfig
cmd /c net share
```

### 9.2 Verify Events on the WEC Collector

After generating events, verify they arrived in the ForwardedEvents log on the WEC collector:

```powershell
wevtutil qe ForwardedEvents /c:10 /f:text /rd:true
```

Or refresh Event Viewer → **Windows Logs → Forwarded Events**.

> **Note:** If using Normal delivery optimization, events may take up to 15 minutes. With Minimize Latency (`wecutil ss "Security-Events" /cm:Custom` then `wecutil ss "Security-Events" /dmlt:30000`), events arrive within 30 seconds.

### 9.3 Verify Events in Sentinel via KQL

**Quick confirmation — all events from the last 30 minutes:**

```kql
WindowsEvent
| where TimeGenerated > ago(30m)
| summarize count() by Computer, EventID
| sort by count_ desc
```

**Failed logon (4625):**

```kql
WindowsEvent
| where TimeGenerated > ago(1h)
| where EventID == 4625
| extend TargetUser = tostring(EventData.TargetUserName)
| project TimeGenerated, Computer, TargetUser
```

**User created and deleted (4720, 4726):**

```kql
WindowsEvent
| where TimeGenerated > ago(1h)
| where EventID in (4720, 4726)
| extend TargetUser = tostring(EventData.TargetUserName)
| extend Actor = tostring(EventData.SubjectUserName)
| project TimeGenerated, Computer, EventID, Actor, TargetUser
```

**User added to group (4732):**

```kql
WindowsEvent
| where TimeGenerated > ago(1h)
| where EventID == 4732
| extend GroupName = tostring(EventData.TargetUserName)
| extend MemberAdded = tostring(EventData.MemberSid)
| extend Actor = tostring(EventData.SubjectUserName)
| project TimeGenerated, Computer, Actor, GroupName, MemberAdded
```

**Process creation (4688):**

```kql
WindowsEvent
| where TimeGenerated > ago(1h)
| where EventID == 4688
| extend NewProcess = tostring(EventData.NewProcessName)
| extend User = tostring(EventData.SubjectUserName)
| project TimeGenerated, Computer, User, NewProcess
| sort by TimeGenerated desc
| take 20
```

**All source computers reporting — confirms WEC pipeline end to end:**

```kql
WindowsEvent
| where TimeGenerated > ago(1h)
| summarize EventCount = count(), LastSeen = max(TimeGenerated) by Computer
```

If you see `win-log-1.kcl.com` as the Computer with your test Event IDs, the full chain is working: source server → WEF → WEC collector → AMA → Sentinel.

---

## Part 10 — Configuring HTTPS for WEF (TCP 5986)

HTTPS WEF encrypts the WinRM transport between source servers and the WEC collector using TLS certificates. This requires certificates on **both sides** — a server certificate on the WEC collector for the HTTPS listener, and a client certificate on each source server for client authentication.

### 10.1 Certificate Requirements

**WEC Collector certificate must:**
- Have the collector's FQDN (e.g., `wec-1.kcl.com`) in the Subject or Subject Alternative Name (SAN)
- Be in the Local Machine → Personal certificate store (`Cert:\LocalMachine\My`)
- Have a private key
- Include Server Authentication (1.3.6.1.5.5.7.3.1) in Enhanced Key Usage
- Be trusted by all source servers

**Source Server certificates must:**
- Have the source server's FQDN in the Subject or SAN
- Be in the Local Machine → Personal certificate store
- Have a private key
- Include Client Authentication (1.3.6.1.5.5.7.3.2) in Enhanced Key Usage
- Be trusted by the WEC collector

### 10.2 Certificate Options

| Option | How to Obtain | Trust Distribution | Best For |
|--------|--------------|-------------------|----------|
| **AD Certificate Services (AD CS)** | Enterprise CA auto-enrolls via GPO using the Machine template | Automatic — all domain-joined machines trust the Enterprise CA | Domain environments at scale |
| **Microsoft Cloud PKI (Intune)** | Cloud-native CA issues certs via SCEP/PKCS Intune profiles | Intune-managed devices trust the cloud CA | Hybrid Entra joined / Intune-managed environments |
| **Azure Key Vault** | Generate or import certificates, integrate with partner CAs (DigiCert, GlobalSign) | Manual distribution or automation via DevOps pipelines | Application certificates, non-domain environments |
| **Third-Party CA** | Purchase from DigiCert, GlobalSign, Sectigo, etc. | Import CA cert to Trusted Root on all servers | Non-domain environments, external-facing WEC collectors |
| **Self-Signed** | `New-SelfSignedCertificate` PowerShell cmdlet | Manual — export and import to Trusted Root on each server | Lab and testing only |

> **Recommendation:** For domain-joined environments, AD CS with the Machine template and GPO auto-enrollment is the simplest path. Every domain-joined server gets a certificate automatically and trust is handled by the domain.

### 10.3 Obtaining Certificates

#### Option A: AD CS (Enterprise CA with Auto-Enrollment)

If an Enterprise CA is available, request a machine certificate. On each server (WEC collector and source servers):

```powershell
Get-Certificate -Template "Machine" -CertStoreLocation "Cert:\LocalMachine\My"
```

For auto-enrollment at scale via GPO:
1. On the CA, ensure the **Computer** template grants **Domain Computers** the **Autoenroll** permission
2. Issue the template: CA console → Certificate Templates → New → Certificate Template to Issue → select **Computer**
3. Create a GPO: **Computer Configuration → Windows Settings → Security Settings → Public Key Policies → Certificate Services Client - Auto-Enrollment** → Enable

After `gpupdate /force`, every domain-joined server receives a certificate automatically.

#### Option B: Self-Signed (Lab/Testing Only)

On the **WEC collector**:

```powershell
$cert = New-SelfSignedCertificate `
    -DnsName "wec-1.kcl.com" `
    -CertStoreLocation "Cert:\LocalMachine\My" `
    -KeyLength 2048 `
    -NotAfter (Get-Date).AddYears(3)

Export-Certificate -Cert $cert -FilePath "C:\WEC-Cert.cer"
$cert.Thumbprint
```

On each **source server**, import the WEC collector's cert into Trusted Root:

```powershell
Import-Certificate -FilePath "C:\WEC-Cert.cer" -CertStoreLocation "Cert:\LocalMachine\Root"
```

Repeat for source server certs — generate a self-signed cert on each source, export, and import into Trusted Root on the WEC collector. Or distribute via GPO: **Computer Configuration → Windows Settings → Security Settings → Public Key Policies → Trusted Root Certification Authorities** → Import.

### 10.4 Configure the HTTPS Listener on the WEC Collector

All commands in this section run on the **WEC collector** (`wec-1.kcl.com`).

**Step 1: Identify the certificate thumbprint:**

```powershell
Get-ChildItem Cert:\LocalMachine\My | Format-List Subject, Thumbprint, DnsNameList, EnhancedKeyUsageList
```

Find the certificate with your WEC collector's FQDN in the Subject (e.g., `CN=wec-1.kcl.com`). Copy the thumbprint.

**Step 2: Create the HTTPS listener:**

```powershell
New-WSManInstance winrm/config/Listener `
    -SelectorSet @{Address="*"; Transport="HTTPS"} `
    -ValueSet @{Hostname="wec-1.kcl.com"; CertificateThumbprint="YOUR_THUMBPRINT_HERE"}
```

> **Important:** The `Hostname` value must exactly match the certificate's Subject CN or SAN entry. Case matters.

**Step 3: Verify the listener:**

```powershell
winrm enumerate winrm/config/Listener
```

You should see two listeners:
- HTTP on port 5985
- HTTPS on port 5986

**Step 4: Open the firewall for port 5986:**

```powershell
New-NetFirewallRule -DisplayName "WinRM HTTPS" -Direction Inbound -Protocol TCP -LocalPort 5986 -Action Allow
```

### 10.5 Issue a Machine Certificate on Each Source Server

Each source server needs its own machine certificate for client authentication over HTTPS.

On each **source server** (e.g., `win-log-1.kcl.com`):

```powershell
Get-Certificate -Template "Machine" -CertStoreLocation "Cert:\LocalMachine\My"
```

Verify the certificate was issued:

```powershell
Get-ChildItem Cert:\LocalMachine\My | Format-List Subject, Thumbprint
```

Confirm you see a certificate with the source server's FQDN (e.g., `CN=win-log-1.kcl.com`).

> **Note:** With AD CS auto-enrollment via GPO, this step happens automatically on every domain-joined machine. No manual per-server action needed.

### 10.6 Switch the Subscription to HTTPS

On the **WEC collector** (`wec-1.kcl.com`):

```powershell
wecutil ss "Security-Events" /tn:HTTPS
Restart-Service Wecsvc
```

### 10.7 Update Source Servers to Use HTTPS

**Manual (single server):**

On the **source server** (`win-log-1.kcl.com`):

```powershell
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\EventLog\EventForwarding\SubscriptionManager" /v 1 /t REG_SZ /d "Server=https://wec-1.kcl.com:5986/wsman/SubscriptionManager/WEC,Refresh=60" /f
Restart-Service WinRM
```

**Via GPO (at scale):**

Update the GPO at **Computer Configuration → Administrative Templates → Windows Components → Event Forwarding → Configure Target Subscription Manager**:

```
Server=https://wec-1.kcl.com:5986/wsman/SubscriptionManager/WEC,Refresh=60
```

Note the changes from HTTP: `https://` instead of `http://` and port `5986` instead of `5985`.

### 10.8 Verify HTTPS WEF Is Working

**Step 1: Test HTTPS connectivity from the source server:**

```powershell
Test-NetConnection wec-1.kcl.com -Port 5986
Test-WSMan -ComputerName wec-1.kcl.com -UseSSL
```

**Step 2: Check subscription status on the WEC collector:**

```powershell
wecutil gr "Security-Events"
```

Source servers should show **RunTimeStatus: Active** with **LastError: 0**.

**Step 3: Generate a test event on the source server:**

```powershell
net use \\localhost /user:fakeuser fakepassword 2>$null
```

**Step 4: Verify events arrived on the WEC collector:**

```powershell
wevtutil qe ForwardedEvents /c:5 /f:text /rd:true
```

**Step 5: Verify in Sentinel:**

```kql
WindowsEvent
| where TimeGenerated > ago(30m)
| where Computer == "win-log-1.kcl.com"
| summarize count() by EventID
| sort by count_ desc
```

### 10.9 Optional: Disable HTTP Listener

Once HTTPS is confirmed working end to end, you can remove the HTTP listener to force all WEF traffic over HTTPS. Run on the **WEC collector**:

```powershell
# Remove the HTTP listener
winrm delete winrm/config/Listener?Address=*+Transport=HTTP

# Close port 5985
Remove-NetFirewallRule -DisplayName "Windows Remote Management (HTTP-In)"
```

> **Warning:** Only disable HTTP after confirming HTTPS is fully operational and all source servers have been updated to use HTTPS. Removing the HTTP listener while source servers are still configured for HTTP will break forwarding immediately.

### 10.10 HTTPS Troubleshooting

| Issue | Likely Cause | Fix |
|-------|-------------|-----|
| `wecutil gr` shows error 0x6ba (RPC unavailable) | Wecsvc service crashed after transport change | `Restart-Service Wecsvc` |
| `wecutil gr` shows error 0x2 (file not found) | Subscription corrupted during transport switch | Delete and recreate the subscription |
| Certificate CN and hostname do not match | Hostname in `New-WSManInstance` doesn't match cert Subject | Check cert with `Get-ChildItem Cert:\LocalMachine\My | FL Subject, DnsNameList` and use exact match |
| Certificate thumbprint is blank or invalid | Wrong cert selected, or extra character in thumbprint | Verify thumbprint with `Get-ChildItem Cert:\LocalMachine\My | FL Thumbprint` |
| Source shows Inactive after switching to HTTPS | Source server missing machine certificate for client auth | Run `Get-Certificate -Template "Machine"` on the source server |
| Source can't connect on 5986 | Firewall blocking | `Test-NetConnection wec-1.kcl.com -Port 5986` from source; add firewall rule if needed |
| `Test-WSMan -UseSSL` fails with trust error | Source server doesn't trust the CA that issued the WEC cert | Import CA cert into `Cert:\LocalMachine\Root` on the source |

### 10.11 HTTPS Quick Checklist

| Step | Where | Command |
|------|-------|---------|
| Certificate on WEC collector | WEC | `Get-ChildItem Cert:\LocalMachine\My | FL Subject, Thumbprint` |
| HTTPS listener created | WEC | `winrm enumerate winrm/config/Listener` |
| Port 5986 open | WEC | `Get-NetFirewallRule -DisplayName "WinRM HTTPS"` |
| Subscription set to HTTPS | WEC | `wecutil gs "Security-Events"` (check TransportName) |
| Certificate on source server | Source | `Get-ChildItem Cert:\LocalMachine\My | FL Subject, Thumbprint` |
| Registry updated to HTTPS/5986 | Source | `Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\...\SubscriptionManager"` |
| Source showing Active | WEC | `wecutil gr "Security-Events"` |
| Events arriving | WEC | `wevtutil qe ForwardedEvents /c:5 /f:text /rd:true` |
| Events in Sentinel | Sentinel | `WindowsEvent | where TimeGenerated > ago(30m)` |

---

## Part 11 — Pending Items

- XPath filter optimization examples for common security detection use cases
- Multiple WEC collector load balancing via AD security group GPO distribution
- Certificate renewal and rotation procedures for HTTPS WEF
