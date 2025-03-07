# Symptoms

*   Virtual machines have intermittent connectivity issues
*   Packet drops when transferring large amounts of data between virtual machines
*   Performance degradation when using SDN features
*   Network traffic failures between hosts when MTU size exceeds standard limits
*   SDN health diagnostic reports showing "InvalidEncapOverheadConfiguration" faults

# Issue Validation
You can validate this issue using the following PowerShell commands:

##1. Check EncapOverhead configuration on network adapters:
```Powershell
Get-NetAdapterAdvancedProperty -DisplayName "*Encap*" | Format-Table Name, DisplayName, DisplayValue, RegistryValue
```
The expected configuration should show `EncapOverheadEnabled` as `Enabled` with a value of `160`.

##2. If EncapOverhead is not supported, check JumboPacket configuration:

```Powershell
Get-NetAdapterAdvancedProperty -DisplayName "*Jumbo*" | Format-Table Name, DisplayName, DisplayValue, RegistryValue
```
The JumboPacket value should be at least `1674` bytes (standard 1514 MTU + 160 encapsulation overhead).

##3. Check if MTU issues are occurring by testing with different packet sizes:
```Powershell
# Test with standard packet size (should work)
Test-NetConnection -ComputerName [ProviderAddress] -InformationLevel Detailed

# Test with larger packet size (may fail if EncapOverhead is misconfigured)
ping [ProviderAddress] -f -l 1600
```

##4. View network health diagnostics for EncapOverhead issues:
```Powershell
$healthTest = Debug-SdnFabricInfrastructure -Role Server
$healthTest | Where-Object {$_.RoleTest.HealthTest.Name -like "Test-SdnEncapOverhead"} | Select-Object -ExpandProperty RoleTest | Select-Object -ExpandProperty HealthTest
```


# Cause
The EncapOverhead misconfiguration occurs due to the following common reasons:
1.  **Network adapter driver/firmware limitation**: Some network adapters do not support the EncapOverhead feature, particularly if using older or default inbox drivers.
2.  **Incomplete network configuration**: During deployment, the proper JumboPacket settings were not applied as a fallback for adapters that don't support EncapOverhead.
3.  **Incorrect MTU configuration**: The underlying physical network infrastructure may not be properly configured to support the larger MTU required for encapsulated packets.
4.  **Network adapter configuration changed**: Manual changes or driver updates may have reset the configuration.

# Mitigation Details

Pre-requisite: Please download the SDNDiagnosticsModule from PSGallery.

##If your network adapter supports EncapOverhead:

1. Update to the latest network adapter drivers and firmware:

```Powershell
Update-VMNetworkAdapter -VMName [VMName] -Force
```

2. Enable EncapOverhead and set the correct value:

```Powershell
Set-NetAdapterAdvancedProperty -Name [AdapterName] -DisplayName "Encapsulated Packet Offload" -RegistryValue 1
Set-NetAdapterAdvancedProperty -Name [AdapterName] -DisplayName "Encapsulated Overhead" -RegistryValue 160
```

##If your network adapter does not support EncapOverhead:

1. Configure JumboPacket as a workaround:

```Powershell
Set-NetAdapterAdvancedProperty -Name [AdapterName] -DisplayName "Jumbo Packet" -RegistryValue 1674
```
2. For Intel network adapters, the setting may be called "Jumbo Frame":
```Powershell
Set-NetAdapterAdvancedProperty -Name [AdapterName] -DisplayName "Jumbo Frame" -RegistryValue 9014
```

##For physical network infrastructure:

1. Ensure all physical switches and routers between SDN hosts support and are configured for jumbo frames (minimum MTU of 1674 bytes).
2. If using NetworkATC (Network Advanced Threat Control), use it to configure optimal networking settings:
```Powershell
Import-Module NetworkATC
Get-NetAdapterAdvancedPropertyInventory -InterfaceName [AdapterName]
Set-NetAdapterAdvancedProperty -Name [AdapterName] -DisplayName "Jumbo Packet" -RegistryValue 1674
```

3. After making changes, restart the network controller host agent:

```Powershell
Restart-Service NCHostAgent -Force
```
4. Validate configuration was properly applied and test connectivity with jumbo packets.


##Additional Information
The EncapOverhead value of 160 bytes accounts for various encapsulation headers:
*   VXLAN: 50 bytes (Outer Ethernet + IP + UDP + VXLAN)
*   NVGRE: 42 bytes (Outer Ethernet + IP + GRE + NVGRE)
*   Additional overhead is included for future protocol extensions

For more information on SDN network requirements, refer to the Microsoft SDN documentation.
