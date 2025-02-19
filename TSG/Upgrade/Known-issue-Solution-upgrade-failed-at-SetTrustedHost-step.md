The issue is during Solution upgrade, the step to set trusted host failed due to the existing group policy settings on some customer settings. This TSG will explain how to modify the group policies and resume upgrade. 
# Symptoms
This exception message will be thrown during Solution upgrade and can be seen from the Portal. 
**Exception**:
`Type 'SetTrustedHosts' of Role 'BareMetal' raised an exception: The config setting TrustedHosts cannot be changed because is controlled by policies. The policy would need to be set to "Not Configured" in order to change the config setting.`



# Cause
For Deployment customers, we don't allow group policy settings. However, for existing customers doing Solution upgrade, we don't have such requirements. Therefore there could be cases where the trusted hosts can not be changed due to the group policy settings. 

# Mitigation Details

**Modify GPO**
1. Open Edit Group Policy on the host.
1. Go to Computer Configuration → Administrative Templates → Windows Components → Windows Remote Management (WinRM) → WinRM Client.
1. Set "Trusted Hosts" to be **"Not Configured"** as shown in the screenshot below.
![Image](./images/settrustedhosts.png)
1. Run `gpupdate /force` from Powershell session.
1. Repeat 1-4 on all hosts.

**Resume Solution Upgrade**
- Click resume Solution upgrade through Azure Portal. 