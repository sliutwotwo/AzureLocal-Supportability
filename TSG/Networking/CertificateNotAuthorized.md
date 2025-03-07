# Symptoms
*   Error `CertificateNotAuthorized` appears in Network Controller configuration state
*   Connections failing between Network Controller and Hyper-V hosts
*   SLB MUX connectivity issues showing certificate authorization errors
*   Configuration changes failing to propagate
*   Error message "Certificate not authorized" in logs
# Issue Validation

Check configuration state and look for the `CertificateNotAuthorized` error code:

```Powershell
Debug-NetworkControllerConfigurationState -NetworkController <FQDN or NC IP> [-Credential <PS Credential>]
```
Verify certificate properties:

```Powershell
# Check that certificate has correct EKUs (Server Authentication and Client Authentication)
dir cert:\\localmachine\my | Where-Object {$_.Subject -like "*hostname*"} | Select-Object -ExpandProperty EnhancedKeyUsageList
```

Check whether multiple certificates with the same subject name exist on the Hyper-V host:

```Powershell
dir cert:\localmachine\my | Where-Object {$_.Subject -like "*hostname.domain.com*"} | Format-List Subject, Thumbprint, NotAfter
```

# Cause
The certificate is trusted (chain validation succeeds) but is not authorized for the specific communication purpose. Common causes include:
1.  Incorrect certificate usage extensions (missing Server Authentication or Client Authentication EKUs)
2.  Incorrect subject name (doesn't match FQDN of host)
3.  Multiple certificates with the same subject name causing confusion
4.  Missing custom Enhanced Key Usage property with OID 1.3.6.1.4.1.311.95.1.1.1 when multiple certificates with same subject name exist
5. Multiple certificates have the same subject name (CN) as the host FQDN in the Personal certificate store, the Network Controller Host Agent may randomly choose one, which may not match the expected certificate configured in Network Controller.

# Mitigation Details

## Generic Mitigation Details
1. Verify certificate subject name matches the FQDN of the host:

```Powershell
# Compare certificate subject name with the expected value
dir cert:\\localmachine\my | Where-Object {$_.Subject -like "*hostname*"} | Format-List Subject,Thumbprint
```
2. Check that certificates have both Server Authentication and Client Authentication EKUs:

```Powershell
# Verify certificate has the correct EKUs
dir cert:\\localmachine\my | Where-Object {$_.Subject -like "*hostname*"} | 
  Select-Object -ExpandProperty EnhancedKeyUsageList
```

3. If multiple certificates have the same subject name, ensure the certificate intended for SDN has the additional custom OID (1.3.6.1.4.1.311.95.1.1.1), or delete all but one certificate with the same subject name.
4. Restart the Network Controller Host Agent after making certificate changes:

```Powershell
# Restart Host Agent
Start-Service NcHostAgent -Force
```

5. In VMM deployments, if issues persist, you might need to:
*   Delete the Hyper-V Server from VMM
*   Remove the HostId registry key
*   Add the server through VMM again

```Powershell
# Remove HostId registry key if recreating the host in VMM
Remove-ItemProperty "hklm:\system\currentcontrolset\services\nchostagent\parameters" -Name HostId
```

## Multiple Certificates with Same Subject Name
1. Ensure only one certificate with the host FQDN as the subject name exists in the personal store:

```Powershell
# List certificates with the same subject name
dir cert:\localmachine\my | Where-Object {$_.Subject -like "*hostname.domain.com*"} | Format-List Subject, Thumbprint, NotAfter

# Export and backup any certificates you plan to remove
Export-PfxCertificate -Cert cert:\localmachine\my\<thumbprint> -FilePath backup.pfx -Password (ConvertTo-SecureString -String "password" -Force -AsPlainText)

# Remove extra certificates, keeping only the one you want to use
Remove-Item -Path cert:\localmachine\my\<thumbprint>
```

2. If you must have multiple certificates with the same subject name, add the custom Enhanced Key Usage OID 1.3.6.1.4.1.311.95.1.1.1 to the certificate that should be used for SDN.
3. Restart the Network Controller Host Agent:

```Powershell
Start-Service NcHostAgent -Force
```