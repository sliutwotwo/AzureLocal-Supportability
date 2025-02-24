# Known Issue: There is not enough space on the disk. Observability subdirectory UdiSessions folder is too large

The folder C:\Observability\Download\UdiSessions is growing too large, causing low disk space issues. There is a pruner scheduled task that is responsible for managing space in C:\Observability to ensure that it does not grow too large, but the volume of data being written outpaces what the pruner can prune and so it is left in an error state.

# Symptoms
If "There is not enough space on disk" or an otherwise unexplained full C drive is observed, the issue may be due to the C:\Observability\Download\UdiSessions folder growing too large.

# Issue Validation
To confirm the scenario that you are encountering is the issue documented in this article, confirm you are seeing the following behavior:

```Powershell
(Get-ChildItem "C:\Observability\Download\UdiSessions" -File -Recurse -Force | Measure-Object Length -Sum).Sum / 1048576
```
The above command reveals that the folder size of C:\Observability\Download\UdiSessions greatly exceeds its designated quota of 256MB.

# Cause
Files in C:\Observability\Download\UdiSessions are written by the UDI API library that is provided by the OS update team. If the stamp has undergone multiple updates, then Scan operations on every lingering update folder will be writen to the UdiSessions folder, causing the folder size to outpace what the pruner can prune.

# Mitigation Details
In order to mitigate this issue, we must do the following:

1. Empty the contents of C:\Observability\Download\UdiSessions
2. Delete the stale update packages from the infrastructure share.
3. Ensure that the pruner scheduled task is working properly by stopping any running instances, disabling, and reenabling it.

**NOTE:** Step 2 only needs to be done on a single node since the files to be deleted are on the infrastructure share. However, Steps 1 and 3 need to be done on each node that is affected by the space issue.

## 1. Empty the contents of UdiSessions folder

```PowerShell
Get-ChildItem "C:\Observability\Download\UdiSessions" -Recurse | Remove-Item -Recurse -Force -Confirm:$true
```
## 2. Delete stale update packages.
The stale update packages are found on the infrastructure share, so we need to delete these each of these folders with format Solution\<version>, where \<version> is **LESS THAN OR EQUAL TO** than the current build version. 
```Powershell
# Determine stamp Solution version
$stampVersion = (Get-StampInformation).StampVersion

# Find update packages remaining in the infrastructure share that are obsolete
$packagesDirectory = Join-Path $env:InfraCSVRootFolderPath "Updates\Packages"
$solutionUpdates = Get-ChildItem $packagesDirectory | ? Name -Like "Solution*"
$obsoleteUpdates = $solutionUpdates | where { $solVersion = $_.Name -replace "Solution", ""; [version]$solVersion -le [version]$stampVersion }

# Remove the obsolete directories
foreach ($update in $obsoleteUpdates)
{
    Write-Host "Removing obsolete update $($update.Name)"
    Remove-Item $update.FullName -Recurse -Force
}
```
## 3. Restore ObservabilityVolumePruner 

``` Powershell
$prunerScheduledTask = Get-ScheduledTask -TaskPath "\Microsoft\AzureStack\Observability\" -TaskName "ObservabilityVolumePruner"
$prunerScheduledTask | Stop-ScheduledTask
$prunerScheduledTask | Disable-ScheduledTask 
$prunerScheduledTask | Enable-ScheduledTask
$prunerScheduledTask | Start-ScheduledTask
```
Wait 3 minutes for the scheduled task to run.
If the pruner has been restored, then the output of the following command should show a LastTaskResult of 0. 

``` Powershell
$prunerScheduledTask | Get-ScheduledTaskInfo
```

### What if there is a nonzero LastTaskResult?

If LastTaskResult is 0, great! You're done as the pruner is restored. However if the above command shows a nonzero LastTaskResult, then the scheduled task is still in an error state. This is likely due to an outdated reference to the executing task's script, so it can be restored by updating to the latest version of the TelemetryAndDiagnostics extension. 
    
# <font color='red'>**IMPORTANT: Only if LastTaskResult is nonzero**, proceed with steps a through d to uninstall and reinstall the TelemetryAndDiagnostics extension: </font>


a. Run the following commands. The output is necessary to supply variables to Connect-AzAccount

```Powershell
$arcInfo = & azcmagent show -j | ConvertFrom-Json
$rgName = $arcInfo.resourceGroup
$subscriptionId = $arcInfo.subscriptionId
$tenantId = $arcInfo.TenantId
$region = $arcInfo.location
```
b. Connect to Azure. For example, if using Device authentication, the command would be:

```Powershell
Connect-AzAccount -UseDeviceAuthentication -SubscriptionId $subscriptionId -TenantId $tenantId

```

c. After successful exection of Close-ProcessHandles.ps1 and connection to Azure, we must get the new version of the TelemetryAndDiagnostics extension onto the node.
To get the newest extension version, we will remove the old extension and install the new one directly.
```Powershell
Remove-AzConnectedMachineExtension -ResourceGroupName $rgName -MachineName $env:COMPUTERNAME -Name "AzureEdgeTelemetryAndDiagnostics"
New-AzConnectedMachineExtension `
    -Name "AzureEdgeTelemetryAndDiagnostics" `
    -ResourceGroupName $rgName `
    -SubscriptionId $subscriptionId `
    -MachineName $($env:COMPUTERNAME) `
    -Location $region `
    -Publisher "Microsoft.AzureStack.Observability" `
    -ExtensionType "TelemetryAndDiagnostics" `
    -NoWait

```

d. To verify that the new version of the extension has been installed, run the following command:
```Powershell
Get-AzConnectedMachineExtension -ResourceGroupName $rgName -SubscriptionId $subscriptionId -MachineName $($env:COMPUTERNAME) -Name "AzureEdgeTelemetryAndDiagnostics"
```