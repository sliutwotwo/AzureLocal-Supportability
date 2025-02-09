# Symptoms
When monitoring update progress in the Azure Update Management portal, you may notice that the state has not changed for a long time. Additionally, when checking the state on the Azure Local cluster node using Get-SolutionUpdate, the cmdlet may be unresponsive and eventually fail with a GatewayTimeout error.

![ps-gatewaytimeout.png](/TSG/Update/ps-gatewaytimeout.png)

# Issue Validation
The unresponsiveness of the update service can be verified by calling the Get-SolutionUpdate cmdlet.

If Get-SolutionUpdate returns an update object, then the mitigation details below do not need to be applied.

# Cause
The update service can occasionally take an unusually long time to resolve content from its dependencies. When this happens, combined with frequent polling of the update progress via requests to the service, it can cause the service process to become unresponsive due to resource contention between threads in the process.

# Mitigation Details
The mitigation is to terminate the running update service process and restart the service.

This can be done by moving the Failover Cluster group for the update service to a different node.

Display the current cluster node on which the update service is running:
```powershell
Get-ClusterGroup "Azure Stack HCI Update Service Cluster Group"
```

Sample output:
```
[v-host1]: PS C:\> Get-ClusterGroup "Azure Stack HCI Update Service Cluster Group"

Name                                         OwnerNode State
----                                         --------- -----
Azure Stack HCI Update Service Cluster Group v-Host1   Online
```

Move the update service to a different node.
```powershell
Get-ClusterGroup "Azure Stack HCI Update Service Cluster Group" | Move-ClusterGroup
```

In some cases, the process may take a long time to terminate, causing the Move-ClusterGroup cmdlet to hang and the cluster group to be in a StopPending state. In this case, the update service process can be terminated using Stop-Process.

```powershell
$group = Get-ClusterGroup "Azure Stack HCI Update Service Cluster Group"
$ownerNode = $group.OwnerNode.Name

$serviceCimInstance = Get-CimInstance Win32_Service -CimSession $ownerNode | where Name -eq "Azure Stack HCI Update Service" 

Invoke-Command $ownerNode { Stop-Process -Id $using:serviceCimInstance.ProcessId -Force }
```

Confirm that the update service is now once again in the running state.

```powershell
Get-ClusterGroup "Azure Stack HCI Update Service Cluster Group"
```

Finally, use Get-SolutionUpdate to check the state of the update and verify that the update service is able to respond to commands again.
