Great question, Kevin — this is a super common problem in large enterprise tenants, especially as customers prepare for the Unified Defender Portal migration. Here's how I'd approach it systematically:

## Azure Activity Log + Resource Graph — The Technical Approach

**1. Azure Resource Graph — Find All Sentinel Instances Fast**

Start by querying every workspace with Sentinel enabled across the tenant:

```kusto
resources
| where type == "microsoft.operationsmanagement/solutions"
| where name startswith "SecurityInsights("
| extend workspaceName = extract("SecurityInsights\\((.+)\\)", 1, name)
| project workspaceName, resourceGroup, subscriptionId, location
```

This gives you your master list of all 60 workspaces with their subscription and resource group context. The resource group and subscription names alone often reveal ownership (e.g., `rg-soc-prod`, `sub-finance-security`).

**2. Activity Log — Who Created or Touched It**

For each workspace, query the Activity Log to find the original creator and recent operators. You can do this at scale with PowerShell:

```powershell
# Get all Sentinel-enabled workspaces
$solutions = Search-AzGraph -Query @"
resources
| where type == "microsoft.operationsmanagement/solutions"
| where name startswith "SecurityInsights("
| extend workspaceName = extract("SecurityInsights\\((.+)\\)", 1, name)
| project workspaceName, resourceGroup, subscriptionId, id
"@

# For each, pull Activity Log to find who created/modified it
foreach ($sol in $solutions) {
    Set-AzContext -SubscriptionId $sol.subscriptionId
    
    # Look back 90 days (max for Activity Log)
    $logs = Get-AzActivityLog `
        -ResourceId $sol.id `
        -StartTime (Get-Date).AddDays(-90) `
        -EndTime (Get-Date) |
        Where-Object { $_.OperationName -like "*write*" -or $_.OperationName -like "*create*" } |
        Select-Object Caller, OperationName, EventTimestamp -First 5

    [PSCustomObject]@{
        Workspace      = $sol.workspaceName
        ResourceGroup  = $sol.resourceGroup
        Subscription   = $sol.subscriptionId
        RecentCallers  = ($logs.Caller | Sort-Object -Unique) -join "; "
        EarliestAction = ($logs | Sort-Object EventTimestamp | Select-Object -First 1).EventTimestamp
    }
}
```

The `Caller` field gives you the UPN or service principal that created or last modified the Sentinel solution — that's your strongest ownership signal.

**3. IAM Role Assignments — Who Has Access**

This is often more reliable than Activity Logs (which only go back 90 days). Pull RBAC assignments on each workspace:

```powershell
foreach ($sol in $solutions) {
    Set-AzContext -SubscriptionId $sol.subscriptionId
    
    $workspaceResourceId = "/subscriptions/$($sol.subscriptionId)/resourceGroups/$($sol.resourceGroup)/providers/Microsoft.OperationalInsights/workspaces/$($sol.workspaceName)"
    
    Get-AzRoleAssignment -Scope $workspaceResourceId |
        Where-Object { $_.RoleDefinitionName -in @(
            "Microsoft Sentinel Contributor",
            "Microsoft Sentinel Responder", 
            "Owner", 
            "Contributor",
            "Log Analytics Contributor"
        )} |
        Select-Object DisplayName, SignInName, RoleDefinitionName, Scope
}
```

Look for who has **Owner** or **Sentinel Contributor** — those are your likely owners. If it's a group, you can resolve group membership via Entra ID.

**4. Tags — Low-Hanging Fruit**

Check if anyone was disciplined enough to tag these:

```kusto
resources
| where type == "microsoft.operationalinsights/workspaces"
| where isnotempty(tags)
| project name, resourceGroup, subscriptionId, tags
```

Look for tags like `owner`, `team`, `costCenter`, `department`.

**5. Data Connectors — What's Flowing In**

The connectors enabled on each workspace tell you a lot about who set it up and what team it serves:

```kusto
resources
| where type == "microsoft.securityinsights/dataconnectors"
| extend workspace = extract("/workspaces/(.+?)/", 1, id)
| summarize connectors = make_list(kind) by workspace
```

If a workspace only has Office 365 and Entra ID connectors, it's probably an identity/IT team. If it has Syslog, CEF, and Defender for Servers, it's probably a SOC or infra security team.

## Recommended Execution Plan

I'd suggest packaging this up as a single script that outputs an Excel workbook — one row per workspace — with columns for workspace name, subscription, resource group, tags, RBAC owners, Activity Log callers, and active connectors. That gives the customer a single artifact they can circulate internally to confirm ownership before the portal migration.

Want me to build that consolidated discovery script or an Excel output version they can use?
