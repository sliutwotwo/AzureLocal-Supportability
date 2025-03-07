# Symptoms
*   Virtual machines unable to communicate properly on SDN networks
*   New network policies not being applied to hosts
*   Network Controller shows hosts as disconnected
*   Configuration changes not propagating to hosts
*   VM network adapters showing incorrect or missing policies
*   VFP rules not being updated on host
*   SDN health diagnostic reports showing connection failures

# Issue Validation
You can validate this issue using the following PowerShell commands:
# 1. Check TCP connections to the Network Controller API Service:
```Powershell
Get-NetTCPConnection -RemotePort 6640 | Format-Table LocalAddress, LocalPort, RemoteAddress, RemotePort, State
``` 
Look for connections in the `Established` state. If no connections exist or they show a state other than `Established`, there's a connectivity issue.
# 2. Verify the NCHostAgent service is running:
```Powershell
Get-Service -Name NCHostAgent | Format-Table Name, Status, StartType
``` 
The service should be in the `Running` state.
# 3. Check local certificate configuration:
```Powershell
Get-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\NcHostAgent\Parameters\
``` 
This will show the certificate configuration for the host agent, including the `PeerCertificateCName` which indicates the Network Controller it should connect to.
# 4. Test basic network connectivity to Network Controller:
```Powershell
$ncFqdn = (Get-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\NcHostAgent\Parameters\).PeerCertificateCName
Test-NetConnection -ComputerName $ncFqdn -Port 6640
``` 
# 5. Check SDN health diagnostics for connection issues:
```Powershell
$healthTest = Debug-SdnFabricInfrastructure -Role Server
$healthTest | Where-Object {$_.RoleTest.HealthTest.Name -eq "Test-SdnHostAgentConnectionStateToApiService"} | Select-Object -ExpandProperty RoleTest | Select-Object -ExpandProperty HealthTest
``` 

# Cause
The connection issues between Host Agent and Network Controller API Service typically occur due to:
1.  **Network Controller API Service unavailability**: The ApiService role in Network Controller is not running or has failed over.
2.  **Network connectivity issues**: Firewall rules or network configuration preventing TCP/TLS communication on port 6640.
3.  **Certificate issues**: Invalid, expired, or mismatched certificates used for TLS communication.
4.  **Host Agent service issues**: The NCHostAgent service is stopped, crashed, or misconfigured.
5.  **DNS resolution failure**: The host cannot resolve the Network Controller FQDN.
6.  **Network Controller overload**: The API service is overloaded and unable to accept new connections.
7.  **TLS handshake failures**: Security protocol version mismatches or cipher suite incompatibilities.

# Mitigation Details
## 1. Verify and restart Network Controller API Service:
```Powershell
# On Network Controller node
Get-ClusterGroup -Name ApiService | Format-Table Name, State, OwnerNode

# If not Online, bring it online
Start-ClusterGroup -Name ApiService
```
## 2. Restart the Network Controller Host Agent:
```Powershell
Restart-Service NCHostAgent -Force
```
## 3. Check and fix certificate issues:
```Powershell
# Check certificate in Host Agent parameters
$ncParams = Get-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\NcHostAgent\Parameters\
$cert = Get-ChildItem -Path Cert:\LocalMachine\My | Where-Object {$_.Thumbprint -eq $ncParams.ClientCertificateThumbprint}
$cert | Format-List Subject, Thumbprint, NotBefore, NotAfter, EnhancedKeyUsageList

# Verify the certificate is valid and has Client Authentication EKU
```
## 4. Verify firewall rules:
```Powershell
Get-NetFirewallRule | Where-Object {$_.DisplayName -like "*Network Controller*" -or $_.DisplayName -like "*SDN*"} | Format-Table DisplayName, Enabled, Direction, Action
```
## 5. Check DNS resolution:
```Powershell
$ncFqdn = (Get-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\NcHostAgent\Parameters\).PeerCertificateCName
Resolve-DnsName -Name $ncFqdn
```
## 6. For persistent issues, reset the host configuration:
```Powershell
# Stop the host agent service
Stop-Service NCHostAgent

# Clear the connection state
Remove-Item -Path HKLM:\SYSTEM\CurrentControlSet\Services\NcHostAgent\Parameters\ServiceGlueState -Force -ErrorAction SilentlyContinue

# Restart the service
Start-Service NCHostAgent
```
## 7. Verify Network Controller health:
```Powershell
# On Network Controller node
Debug-SdnNetworkController
```
## Additional Information
*   The connection between NCHostAgent and Network Controller API Service uses TCP port 6640 with TLS encryption.
*   The connection uses mutual certificate authentication, so both the host and Network Controller need valid certificates.
*   In high-availability configurations, the Network Controller API Service may be on a different node than expected due to failover.
*   Check the Windows event logs for specific error messages related to the connection:
```Powershell
Get-WinEvent -LogName Microsoft-Windows-NetworkController/Operational -MaxEvents 100 | Where-Object {$_.Message -like "*connection*" -or $_.Message -like "*host agent*"}
```
*   The NCHostAgent service will continuously attempt to reconnect to the Network Controller API Service if disconnected.
*   Review SDN diagnostic logs at `C:\Windows\Logs\SDN\` for detailed connection troubleshooting information and open up a support ticket.

For more information on SDN network connectivity requirements, refer to the Microsoft SDN documentation.