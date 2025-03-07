# Symptoms
*   Error `CertificateNotTrusted` appears in Network Controller configuration state output
*   Failed communications between Network Controller and host agents
*   Hosts not connecting to Network Controller properly
*   Error message "Certificate is not trusted" in logs
*   Network policy doesn't propagate to Hyper-V hosts
*   SLB/Gateway connectivity issues
*   HTTP 403.7 or 403.16 errors in IIS logs

# Issue Validation

Check configuration state to confirm certificate trust issues:

```powershell
Debug-NetworkControllerConfigurationState -NetworkController <FQDN or NC IP> [-Credential <PS Credential>]
```
Look for errors with code `CertificateNotTrusted` and message `Certificate is not trusted`.

Check the certificate store on the affected components:

```powershell
# On Hyper-V Host or Network Controller node 
dir cert:\localmachine\root | Format-List Subject, Issuer, Thumbprint
```
# Cause
There are several potential causes for CertificateNotTrusted errors:
1.  The certificate presented by the Hyper-V host to the Network Controller (or vice versa) cannot be validated through the certificate chain
2.  The CA root certificate is missing from the Trusted Root Certification Authorities store
3.  Non-self-signed certificates in the Trusted Root Certification Authorities store causing authentication issues
4.  Certificates where "Issued to" isn't the same as "Issued by" in the Trusted Root Certification Authorities store
5.  Certificate chain validation failure

# Mitigation Details
1.  Check certificates on both Network Controller and Hyper-V hosts:

```powershell
# On Hyper-V Host 
dir cert:\\localmachine\my 
dir cert:\\localmachine\root 

# On Network Controller Node VM 
dir cert:\\localmachine\root
```
2.  Ensure the certificate used by the Hyper-V host has its issuing CA in the trusted root store of the Network Controller, and vice versa.

3.  Remove any non-self-signed certificates from the Trusted Root Certification Authorities store:

```powershell
# First identify certificates where "Issued to" isn't the same as "Issued by" 
dir cert:\localmachine\root | Where-Object { $_.Subject -ne $_.Issuer } | Format-List Subject, Issuer, Thumbprint 

# Then remove problematic certificates or move them to the Intermediate Certification Authorities store 
Move-Item -Path cert:\localmachine\root\<thumbprint> -Destination cert:\localmachine\ca
```
4.  Restart the Network Controller Host Agent service after certificate changes:

```powershell
# Restart NC Host Agent
Start-Service NcHostAgent -Force
```

5. Please open up a support case if still running into issues.
