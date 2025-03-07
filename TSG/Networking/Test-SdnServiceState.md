# Symptoms
*   SDN features are not functioning properly
*   Virtual machines unable to communicate on SDN networks
*   Network Controller unable to manage network resources
*   Load balancer functionality not working
*   Network policy enforcement failures
*   SDN health diagnostic reports showing "ServiceDown" faults
*   Event log entries indicating service failures

# Issue Validation

You can validate service state issues using these PowerShell commands:

##1. Check SDN agent service status:

```Powershell
Get-Service -Name NCHostAgent, SlbHostAgent | Format-Table Name, Status, StartType
```

These services should be in a `Running` state for SDN to function properly.
##2. For Network Controller nodes, check cluster service roles:

```Powershell
# If using Failover Cluster
Get-ClusterGroup | Where-Object {$_.Name -in @("ApiService", "ControllerService", "FirewallService", "FnmService", "GatewayManager", "ServiceInsertion", "VSwitchService")} | Format-Table Name, State, OwnerNode

# If using Service Fabric
Get-ServiceFabricService -ApplicationName fabric:/NetworkController
```
##3. Check service dependencies to ensure they're also running:

```Powershell
Get-Service -Name NCHostAgent | Select-Object -ExpandProperty DependentServices | Format-Table Name, Status
```
##4. Check recent service-related events in the event log:

```Powershell
Get-EventLog -LogName System -EntryType Error -Source "Service Control Manager" -Newest 10
```
##5. View network health diagnostics for service issues:

```Powershell
$healthTest = Debug-SdnFabricInfrastructure
$healthTest | Where-Object {$_.RoleTest.HealthTest.Name -eq "Test-SdnServiceState"} | Select-Object -ExpandProperty RoleTest | Select-Object -ExpandProperty HealthTest
```

# Cause

Service state issues typically occur due to the following reasons:
1.  **Service crash**: The service process terminated unexpectedly due to an unhandled exception or resource issue.
2.  **Service dependency failure**: A dependent service is not running, preventing the SDN service from starting.
3.  **Incorrect service configuration**: Service account, permissions, or other settings may be incorrectly configured.
4.  **Resource constraints**: Insufficient CPU, memory, or disk resources causing service instability.
5.  **Cluster transition issues**: In clustered environments, failures during node failover can leave services in incorrect states.
6.  **Software updates**: Recent patching or updates may have affected service configuration.
7.  **Network connectivity issues**: For distributed services, network problems can cause service communication failures.

# Mitigation Details

## For Agent Services (NCHostAgent, SlbHostAgent):
1. Start stopped services:
```Powershell
Start-Service -Name NCHostAgent
Start-Service -Name SlbHostAgent
```
2. If services fail to start, restart the server:
```Powershell
Restart-Computer -Force
```
3. Check service logs for specific failures:
```Powershell
Get-WinEvent -LogName Microsoft-WindowsAzure-Diagnostics/Operational | Where-Object {$_.Message -like "*NCHostAgent*"} | Select-Object TimeCreated, Message -First 10
```
And open a Support Ticket

## For Network Controller Cluster Services:
1. For Failover Cluster-based Network Controller, bring resources online:
```Powershell
Start-ClusterGroup -Name "ApiService"
```
2. For Service Fabric-based Network Controller, restart the fabric host service:
```Powershell
Restart-Service FabricHostSvc
```
3. Verify service health after restart:
```Powershell
Get-Service -Name NCHostAgent, SlbHostAgent | Format-Table Name, Status
```
## For Persistent Issues:
1. Check service logs for recurring errors:
```Powershell
# Look for SDN-related errors
Get-WinEvent -FilterHashtable @{LogName='System'; Level=2} | Where-Object {$_.Message -like "*NCHostAgent*" -or $_.Message -like "*SlbHostAgent*"} | Format-List
```
2. Repair SDN components if needed:
```Powershell
# On Network Controller node
Repair-SdnNetworkController

# On host node
Repair-SdnHostConfiguration
```
3. If service configuration appears corrupted, consider reinstalling the SDN components.
4. For cluster issues, check cluster health:
```Powershell
Test-Cluster
```
## Additional Information
*   The `NCHostAgent` service is responsible for configuring host networking components including virtual switches and VFP policies.
*   The `SlbHostAgent` service manages Software Load Balancer (SLB) components on hosts.
*   In a failover cluster, service roles like "ApiService" and "ControllerService" are managed as cluster resources.
*   Verify that the Windows Firewall allows communication between SDN components (ports 6640 and 8560 are particularly important).
*   For persistent issues, check the SdnDiagnostics logs located at `C:\Windows\Logs\SDN`.
For more detailed information on SDN services and their dependencies, refer to the Microsoft SDN documentation.

