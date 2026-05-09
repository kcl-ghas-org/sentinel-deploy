## Lab Setup - Syslog forwarder from Syslog Linux Servers and PFSense Firewall to Microsoft Sentinel

Using Linux VMs (Rocky 9) as the syslog forwarder. For Firewall using pfSense for generating firewall logs and other Linux VMs for syslog sources.

**The scenario that is simulating:**
- pfSense (10.10.10.1) → sends firewall syslog to the Rocky forwarder
- Other Linux VMs → send standard syslog to the same forwarder
- Rocky forwarder has Arc + AMA → two DCRs route to Sentinel

## Step 1: Configure rsyslog on the Rocky Forwarder

SSH into the Rocky 9 VM (syslog forwarder) and enable rsyslog to listen for remote syslog:

```bash
sudo nano /etc/rsyslog.d/10-remote.conf
```

Add:

```
# Listen for incoming syslog on UDP and TCP 514
module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")

# Route firewall logs (pfSense sends on local0 by default) to a separate file
if $syslogfacility-text == 'local0' then {
    action(type="omfile" file="/var/log/firewall/pfsense.log")
    stop
}

# Everything else goes to standard syslog locations
```

Then:

```bash
sudo mkdir -p /var/log/firewall
sudo systemctl restart rsyslog
sudo firewall-cmd --permanent --add-port=514/udp --add-port=514/tcp
sudo firewall-cmd --reload
```

## Step 2: Point pfSense at the Forwarder

In pfSense go to **Status → System Logs → Settings**. Under Remote Logging Options, enable remote logging and set the remote log server to your Rocky VM's IP (e.g., `10.10.10.118:514`). Select the log categories you want to forward (firewall events, system, etc.). pfSense sends on `local0` facility by default.

## Step 3: Point Other Linux VMs at the Forwarder

On any other Linux VM in the lab, add a forwarding rule:

```bash
echo "*.* @10.10.10.118:514" | sudo tee /etc/rsyslog.d/99-forward.conf
sudo systemctl restart rsyslog
```

## Step 4: Verify Logs Are Landing

Back on the Rocky forwarder:

```bash
# Check firewall logs from pfSense
tail -f /var/log/firewall/pfsense.log

# Check standard syslog from other VMs
tail -f /var/log/messages
```

Generate some traffic — browse through pfSense, SSH between VMs, run some `sudo` commands — and confirm you see logs from both sources.

## Step 5: Arc + AMA on the Forwarder

If Arc isn't already on this Rocky VM, onboard it with your Service Principal:

```bash
azcmagent connect \
  --service-principal-id "<APP_ID>" \
  --service-principal-secret "<SECRET>" \
  --resource-group "rg-kevslab" \
  --tenant-id "<TENANT_ID>" \
  --location "southcentralus" \
  --subscription-id "<SUB_ID>"
```

## Step 6: Create the Two DCRs in Sentinel

In Sentinel → **Data connectors**:

**DCR 1 — Syslog via AMA:** Open the connector, create a data collection rule. Add your Rocky forwarder as a resource. Select facilities like `auth`, `authpriv`, `daemon`, `syslog`, `kern` at minimum severity `LOG_INFO` or `LOG_NOTICE`. This captures the standard Linux syslog traffic.

**DCR 2 — Common Event Format (CEF) via AMA:** If pfSense sends CEF-formatted data, use this connector. If pfSense sends standard syslog (which it does by default), you'll see those logs in the Syslog table under `local0` facility instead. Either way works for the demo.

Wait a few minutes for AMA to install and the DCRs to take effect.

## Step 7: Validate in Sentinel

```kql
// Check standard syslog from Linux VMs
Syslog
| where TimeGenerated > ago(30m)
| where Computer contains "your-rocky-hostname"
| summarize count() by Facility, SeverityLevel, HostName
| sort by count_ desc

// Check for pfSense firewall logs (local0 facility)
Syslog
| where TimeGenerated > ago(30m)
| where Facility == "local0"
| take 20

// If using CEF connector, check CommonSecurityLog
CommonSecurityLog
| where TimeGenerated > ago(30m)
| take 20
```

## Step 8: Simulate the Customer's Multi-Folder Pattern

If you want to demo the custom text log approach (Approach 2 from earlier) where AMA reads from separate folders like the customer's Splunk setup, create a second DCR using the **Custom Text Logs** data source pointing at `/var/log/firewall/*.log`. This sends to a custom table you define, showing the customer that their existing folder-based routing can be preserved.

That gives you both patterns to demo side by side — native Syslog/CEF collection and custom file-based collection — so the customer can see which approach fits their workflow better.

## Step 9: Verify the logs are following

sudo tail -f /var/log/firewall/pfsense.log

sudo tail -f /var/log/messages

## Step 9: Verify via KQL in Sentinel
```kql
// Quick health check — is anything arriving?
Syslog
| where TimeGenerated > ago(1h)
| summarize count() by Computer, Facility, SeverityLevel
| sort by count_ desc
```

```kql
// pfSense firewall logs (local0 facility)
Syslog
| where TimeGenerated > ago(1h)
| where Facility == "local0"
| project TimeGenerated, HostName, ProcessName, SyslogMessage
| take 20
```

```kql
// pfSense filterlog — parse out blocked vs passed traffic
Syslog
| where TimeGenerated > ago(1h)
| where Facility == "local0"
| where ProcessName == "filterlog"
| extend Action = case(SyslogMessage has "block", "Block",
                       SyslogMessage has "pass", "Pass",
                       "Other")
| summarize count() by Action
```

```kql
// Linux VM syslog — auth events (SSH logins, sudo, etc.)
Syslog
| where TimeGenerated > ago(1h)
| where Facility in ("auth", "authpriv")
| project TimeGenerated, HostName, ProcessName, SeverityLevel, SyslogMessage
| sort by TimeGenerated desc
| take 20
```

```kql
// All sources reporting — which hosts are sending logs?
Syslog
| where TimeGenerated > ago(1h)
| summarize EventCount = count(), LastSeen = max(TimeGenerated) by HostName, Facility
| sort by EventCount desc
```

```kql
// Volume over time — good for demo dashboards
Syslog
| where TimeGenerated > ago(4h)
| summarize count() by bin(TimeGenerated, 5m), Facility
| render timechart
```

#Folder structure on Syslog server explained
Based on the rsyslog config you created in `/etc/rsyslog.d/10-remote.conf`:

**`/var/log/firewall/pfsense.log`** — Only pfSense firewall logs land here. The rsyslog rule filters on `local0` facility (which is what pfSense sends on by default) and writes to this dedicated file. The `stop` directive prevents these logs from also going to the general log.

**`/var/log/messages`** — Everything else lands here. Standard syslog from kcl-lnx01, kcl-lnx02, Rocky-BR-Demo, the forwarder itself, and any pfSense logs that aren't on `local0` (like nginx, sshd, kernel). This is Rocky's default catch-all log location.

So the split is:

```
Incoming syslog on UDP/TCP 514
        │
        ├── Facility = local0 (pfSense firewall)
        │       └── /var/log/firewall/pfsense.log
        │
        └── Everything else (Linux VMs, other pfSense services)
                └── /var/log/messages
```
