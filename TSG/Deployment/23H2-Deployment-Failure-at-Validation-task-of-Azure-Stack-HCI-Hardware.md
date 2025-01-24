

  

# Symptoms

When deploying an Azure Local 23H2 instance via Azure Portal, you may hit a failure in the Validation task "Azure Stack HCI hardware" during the validation stage. If you click the "Error(View detail) of the above failed task, you will see the below exception.  

```Text

Type 'ValidateHardware' of Role 'EnvironmentValidator' raised an exception: {
"ExceptionType": "json", "ErrorMessage": { "Message": "Hardware requirements not met. 
Review output and remediate.", "Results": [ { "Name": 
"AzStackHci_Hardware_Test_Processor_Instance_Property_VMMonitorModeExtensions", 
"DisplayName": "Test Processor Property VMMonitorModeExtensions
```

# Issue Validation

To confirm the scenario that you are encountering is the issue documented in this article, please run this cmdlet on each node to check the result:
```PowerShell

$result = SystemInfo | Select-String "Virtualization-based security"
Write-Host "$result"
if ($result -eq "Virtualization-based security: Status: Not enabled" -or $result -eq "Virtualization-based security: Status: Enabled but not running") {
    Write-Host "The known issue is hit. You can follow the below steps to mitigate the issue"
}
else {
   Write-Host "This is not a known issue addressed in this article. Please skip the blow steps and reach out to CSS team for troubleshooting" 
}
```
If you see the message "The known issue is hit" in a node, follow the below mitigation steps to resolve the issue on the node. Otherwise, if you do see the message "This is not a known issue addressed in this article" on all nodes, please skip below sections.
 

# Cause

"Virtualization Technology" was not enabled on BIOS or was not enabled in registry.

  

# Mitigation Details


On each node where the issue is hit, follow below steps:

1. Validate that BIOS setting of "Virtualization Technology" is enabled. If not, enable it in BIOS.

2. If enabled, validate that the OS contains the below keys and create them if missing.

```PowerShell

reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\HypervisorEnforcedCodeIntegrity" /v "Enabled" /t REG_DWORD /d 1 /f

reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\HypervisorEnforcedCodeIntegrity" /v "WasEnabledBy" /t REG_DWORD /d 0 /f

reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\SystemGuard" /v "Enabled" /t REG_DWORD /d 1 /f
```

3. Restart the node
4. In the Azure Portal, retry the failed validation by clicking "Try Again" button in the deploy blade.

  

