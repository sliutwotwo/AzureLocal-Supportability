# UPDATE - Known Issue Fixed in LCM Extension Build: 30.2411.2.789

**As of January 6th, 2025:** A new LCM Extension has been pushed to all regions
- **LCM Extension version 30.2411.2.789** is now available with the fix for this issue
- All net-new deployments should no longer need any workarounds/mitigations to proceed past this issue
- Any clusters previously requiring the workaround/mitigation can take the updated LCM Extension, cleanup the Cluster Resource and start the Azure Local Deployment Wizard over again (basically steps 1 and 3 from the [Mitigation Section Below](https://github.com/Azure/AzureLocal-Supportability/edit/main/TSG/Deployment/Triggering-deployment-settings-validation-call-results-in-OperationTimeout-2411-1-and-LCM-Extension-2411-1.md#mitigation) once the LCM Extension has been updated to 30.2411.2.789 on the Registered Nodes)

# Symptoms
  
When deploying a new 23h2 cluster based on the newly available 2411.1 image in combination of the newly released LCM Extension (version 30.2411.1.760), there may be occurrences where you will experience a timeout while initiating the validation process "**Triggering deployment settings validation call....**"

**Note:** This article is specific to the issue associated with the 2411.1 release and the associated LCM Extension (version 30.2411.1.760) release. There is a previous issue for the previous build (2411.0) which presents with the same error symptom in the Portal, please see the previous article [Triggering Deployment Settings Validation Call Results in Operation Timeout in 2411.0](https://github.com/Azure/AzureLocal-Supportability/blob/main/TSG/Deployment/Triggering-deployment-settings-validation-call-results-in-OperationTimeout-2411-0.md) for more information on that issue.

# Issue Validation
To confirm the scenario that you are encountering is one of the issues documented in this article, please confirm that you see a similar error as showed below, from the Portal:

About 10 minutes after validation is started, the following error message will be seen in the portal:
"**Could not complete the operation. 200: OperationTimeout , No updates received from device for operation (...) beyond timeout of [600000] ms**.

<img width="400" alt="Items" src="https://github.com/user-attachments/assets/4a397318-7d01-4674-9db5-e406fc15a0fe">

### Additonal Validation Steps
From the seed node run the following PowerShell command to retrieve the detailed error from the LCM Controller Event log:

```
(Get-WinEvent -LogName Microsoft.AzureStack.LCMController.EventSource/Admin | ? ID -eq 31 | ? LevelDisplayName -eq Error | ? Message -match "String was not recognized as a valid DateTime.").message
```

You should see output with an error message similar to the following:

_Error event message from edge common logger: Exception while checking time of notifcation sent, error = System.FormatException: String was not recognized as a valid DateTime.<br>
&emsp;&emsp;at System.DateTimeParse.Parse(String s, DateTimeFormatInfo dtfi, DateTimeStyles styles)<br>
&emsp;&emsp;at EdgeCommonClient.NotificationClientUtils.IsNotificationWithinExpiry(String notification, IEdgeCommonLogger logger)_

You can also validate what the current culture settings are for the System account, this is the account that the LCM Controller Service runs under:

```
$dateformat = Get-ItemProperty -Path "Registry::HKEY_USERS\S-1-5-18\Control Panel\International\" 
if ($dateformat.sShortDate -ne "M/d/yyyy" -or $dateFormat.sLongDate -ne "dddd, MMMM d, yyyy") {cls;Write-Host -ForeGroundColor red "`nDate format is not expected! Please run the recommended mitigation steps.`n"} else {cls;Write-Host -ForeGroundColor green "`nExpected Date format!`n"}
```

# Cause
Nodes are not set with Culture/Language settings matching en-US / English (United States) resulting in the DateTime format not being recognized. This issue is a regression in the latest build of the LCM Extension (version 30.2411.1.760), which includes the fix for the previous (from 2411.0 release) issue, but now results in this new issue. The fix for this is in progress. **The current workaround/mitigation steps are outlined below**.

# Mitigation
1. From the Azure Portal, you only need to cleanup the cluster object in the resource group. Note it will have type Azure Local and has a lock on it which must be removed prior to deletion. You do not need to remove the Arc Machine, Storage Account or Keyvault as they can be reused.
2. On each Node, run the following PowerShell Commands:

**Note:** The following commands will only change the DateTime format for the values stored in the registry for the System account which is what is used by the service during Validation. They will not change the Culture or Language Settings for the Nodes. This is enough to workaround the Validation issue Timeout for Deployment to start.

```
Set-ItemProperty -Path "Registry::HKEY_USERS\S-1-5-18\Control Panel\International\" -name sLongDate -value "dddd, MMMM d, yyyy"
Set-ItemProperty -Path "Registry::HKEY_USERS\S-1-5-18\Control Panel\International\" -name sShortDate -value "M/d/yyyy"
Get-Service LcmController | Restart-Service
```
3. Start the Azure Local Deployment Wizards over again.

**Note:** If you did not do step 1 and instead attemp Validation again it will fail with the following error: "Validate Operation is not allowed in Current State [Disconnected] for HciCluster....."
