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
Get-WinEvent -LogName Microsoft.AzureStack.LCMController.EventSource/Admin | ? ID -eq 31 | ? LevelDisplayName -eq Error | fl
```

You should see output with an error message similar to the following:

```
TimeCreated  : 25/12/2024 1:37:57 AM
ProviderName : Microsoft.AzureStack.LCMController.EventSource
Id           : 31
Message      : Error event message from edge common logger: Exception while checking time of notifcation sent, error = System.FormatException: String was not recognized as a valid DateTime.
                  at System.DateTimeParse.Parse(String s, DateTimeFormatInfo dtfi, DateTimeStyles styles)
                  at EdgeCommonClient.NotificationClientUtils.IsNotificationWithinExpiry(String notification, IEdgeCommonLogger logger)
```
You can also validate what the current culture settings are for the System account, this is the account that the LCM Controller Service runs under:

```
$User = New-Object System.Security.Principal.NTAccount('NT Authority\System')
$sid = $User.Translate([System.Security.Principal.SecurityIdentifier]).value

get-Itemproperty -Path "Registry::HKEY_USERS\${sid}\Control Panel\International\" -name LocaleName
get-Itemproperty -Path "Registry::HKEY_USERS\${sid}\Control Panel\International\" -name sLongDate
get-Itemproperty -Path "Registry::HKEY_USERS\${sid}\Control Panel\International\" -name sShortDate
```
If the above returns values that do not match the below then you have hit the issue and can use one of the suggestions below to address the issue.

```
LocalName  != en-US
sLongDate  != dddd, MMMM d, yyyy
sShortDate != M/d/yyyy
```

# Cause
Nodes are not set with Culture/Language settings matching en-US / English (United States) resulting in the DateTime format not being recognized. This issue is a regression in the latest build of the LCM Extension (version 30.2411.1.760), which includes the fix for the previous (from 2411.0 release) issue, but now results in this new issue. The fix for this is in progress. **The current workaround/mitigation steps are outlined below**.

# Workaround/Mitigation Details

## Workaround/Mitigation Considerations
- Only one of these options is required to move past the Validation Timeout
- The Workaround outlined above does not require node reboots
- The Mitigation outlined above does require node reboots
- The Workaround only changes the DateTime format in the Registry, while the Mitigation changes the current format to English (United States)

# Workaround Details
1. From the Azure Portal, clean up all prior cluster resources in the Resource Group
2. On each Node, run the following PowerShell Commands:

**Note:** The following commands will only change the DateTime format for the values stored in the registry for the System account which is what is used by the service during Validation. They will not change the Culture or Language Settings for the Nodes. This is enough to workaround the Validation issue Timeout for Deployment to start.

```
$User = New-Object System.Security.Principal.NTAccount('NT Authority\System')`
$sid = $User.Translate([System.Security.Principal.SecurityIdentifier]).value`
Set-ItemProperty -Path "Registry::HKEY_USERS\${sid}\Control Panel\International\" -name sLongDate -value "dddd, MMMM d, yyyy"`
Set-ItemProperty -Path "Registry::HKEY_USERS\${sid}\Control Panel\International\" -name sShortDate -value "M/d/yyyy"`
Get-Service LcmController | Restart-Service`
```
3. Start the Azure Local Deployment Wizards over again.

**Note:** Attempting Validation again from this point will fail, you will need to cleanup the cluster object in the Azure portal and start the deployment wizard over in the Azure Portal. If you attempt to validate without doing this the error you will see will be the following: "Validate Operation is not allowed in Current State [Disconnected] for HciCluster....."

# Mitigation Details
1. From the Azure Portal, clean up all prior cluster resources in the Resource Group
2. On each Node, perform the following steps to change the current format to `English (United States)`:
   1. From sconfig, choose Selection 9
   2. On the `Date and Time` Window, click on `Change date and time...`
   3. On the `Date and Time Settings` Window, click on `Change calendar settings`
   4. On the `Region` Window in the `Formats` tab, change the `Format` dropdown option to: `English (United States)`
   5. On the `Region` Window in the `Formats` tab, click `Apply`
   6. On the `Region` Window in the `Administrative` tab, click on `Copy settings...`
   7. On the `Welcome screen and new user account settings` Window, check the box for `Welcome screen and system accounts`
   8. On the `Welcome screen and new user account settings` Window, click `OK`
   9. With these settings applied, Restart the Node
3. Start the Azure Local Deployment Wizards over again.

**Note:** Attempting Validation again from this point will fail, you will need to cleanup the cluster object in the Azure portal and start the deployment wizard over in the Azure Portal. If you attempt to validate without doing this the error you will see will be the following: "Validate Operation is not allowed in Current State [Disconnected] for HciCluster....."
