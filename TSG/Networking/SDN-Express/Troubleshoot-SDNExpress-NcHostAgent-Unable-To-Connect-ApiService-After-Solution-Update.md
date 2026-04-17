# NcHostAgent Unable to Connect to ApiService After Solution Update

<table border="1" cellpadding="6" cellspacing="0" style="border-collapse:collapse; margin-bottom:1em;">
  <tr>
    <th style="text-align:left; width: 180px;">Component</th>
    <td><strong>SDN Express / Network Controller</strong></td>
  </tr>
  <tr>
    <th style="text-align:left; width: 180px;">Severity</th>
    <td><strong>Critical</strong></td>
  </tr>
  <tr>
    <th style="text-align:left;">Applicable Scenarios</th>
    <td><strong>Solution Update</strong></td>
  </tr>
  <tr>
    <th style="text-align:left;">Affected Versions</th>
    <td><strong>All versions</strong></td>
  </tr>
</table>

## Overview

After completing an Azure Local solution update, Network Controller (NC) loses the ability to manage Hyper-V hosts due to a certificate rotation failure. The solution update rotates host certificates, but the corresponding AzureStackCertificationAuthority certificate is not properly propagated to the Network Controller VMs. This causes the NcHostAgent on each host to fail authentication when connecting to the NC ApiService, effectively breaking SDN management across the cluster.

This issue recurs after every subsequent solution update until the underlying root cause is addressed.

## Symptoms

**Common error messages:**

ConfigurationState reported for /servers/{resourceId} may show failure indicating:

```
PolicyConfigurationFailure
```

If you run `Debug-SdnFabricInfrastructure`, you may see the following test reporting failure:

```
Test-SdnHostAgentConnectionStateToApiService
Description = "Network Controller Host Agent is not connected to the Network Controller API Service."
Impact = "Policy configuration may not be pushed to the Hyper-V host(s) if no southbound connectivity is available."
```

**Observable behaviors:**

- Network Controller is unable to program VFP policies to VMs, resulting.
- VM traffic for VMs, especially after live-migration may break.

## Root Cause

During a solution update, the host certificates are rotated by ECE (Environment Configuration Engine). For the certificate rotation to also propagate the AzureStackCertificationAuthority certificate to the Network Controller VMs, several conditions must be met. A failure in any of the following causes the NcHostAgent-to-ApiService connection to break after the update.

### 1. Missing NetworkControllerNodeNames Registry Key

The certificate rotation process depends on the following registry key being present and populated on each Hyper-V host:

```
HKLM:\SYSTEM\CurrentControlSet\Services\NcHostAgent\Parameters\NetworkControllerNodeNames
```

This registry key is created by Windows Admin Center (WAC) during SDN deployment. In environments where WAC was not used (e.g., SdnExpress-based deployments or brownfield configurations), this registry key does not exist. When the key is missing, the secret rotation process silently skips the NC and SLB certificate injection and incorrectly reports success.

### 2. HCI Admin User Not Added to Local Administrators on NC VMs

The certificate rotation process uses PowerShell remoting (`Invoke-Command`) from the Hyper-V hosts to the Network Controller VMs to install the updated certificates. If the cluster node machine accounts or the HCI admin user are not members of the Local Administrators group on the NC VMs, the remote commands fail and certificate propagation does not complete.

### 3. Code Defect in Certificate Rotation Logic

There is a known defect in the certificate rotation logic where:

- The `Get-VM` lookup assumes VM names match the FQDN or NetBIOS name, which is not always true
- An `Invoke-Command` argument handling issue (extra comma) results in a null argument being passed, causing the rotation to be silently skipped

> **Note:** A fix for these code defects is currently in development and will be addressed in a future update.

## Resolution

### Prerequisites

- Administrative access to the Hyper-V host nodes
- Administrative access to the Network Controller VMs
- [SdnDiagnostics](https://github.com/microsoft/SdnDiagnostics/wiki) PowerShell module installed

### Steps

1. **Validate the registry key exists and is populated**

   On each Hyper-V host, verify that the `NetworkControllerNodeNames` registry key exists and contains the Network Controller VM names.

   ```powershell
   # Check if the registry key exists and has a value
   $regPath = "HKLM:\SYSTEM\CurrentControlSet\Services\NcHostAgent\Parameters"
   $regValue = Get-ItemProperty -Path $regPath -Name "NetworkControllerNodeNames" -ErrorAction SilentlyContinue

   if ($null -eq $regValue) {
       Write-Host "NetworkControllerNodeNames registry key is MISSING." -ForegroundColor Red
   }
   elseif ([string]::IsNullOrWhiteSpace($regValue.NetworkControllerNodeNames)) {
       Write-Host "NetworkControllerNodeNames registry key EXISTS but is EMPTY." -ForegroundColor Yellow
   }
   else {
       Write-Host "NetworkControllerNodeNames: $($regValue.NetworkControllerNodeNames)" -ForegroundColor Green
   }
   ```

   If the key is missing or empty, populate it with the Network Controller VM names (comma-separated):

   ```powershell
   # Replace <NC-VM1>,<NC-VM2>,<NC-VM3> with your actual NC VM names
   $ncNodeNames = "<NC-VM1>,<NC-VM2>,<NC-VM3>"
   Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\NcHostAgent\Parameters" -Name "NetworkControllerNodeNames" -Value $ncNodeNames
   ```

   > **Important:** Repeat this step on every Hyper-V host in the cluster.

2. **Validate local administrator permissions on Network Controller VMs**

   The certificate rotation process requires that the Hyper-V hosts can remotely manage the Network Controller VMs. Verify that the cluster nodes have local administrator permissions on each NC VM.

   ```powershell
   # Run from a Hyper-V host against each NC VM
   # Replace <NC-VM> with your NC VM name
   Invoke-Command -ComputerName <NC-VM> -ScriptBlock {
       $adminGroup = Get-LocalGroupMember -Group "Administrators" -ErrorAction SilentlyContinue
       $adminGroup | Format-Table Name, ObjectClass, PrincipalSource
   }
   ```

   If the cluster nodes are not listed as local administrators, add them:

   ```powershell
   # Run on each NC VM
   # Replace <DOMAIN\ClusterNode$> with your cluster node machine account
   Invoke-Command -ComputerName <NC-VM> -ScriptBlock {
       Add-LocalGroupMember -Group "Administrators" -Member "<DOMAIN\ClusterNode$>"
   }
   ```

   > **Important:** Repeat this for each Hyper-V host against every NC VM.

3. **Run certificate rotation to restore connectivity (short-term mitigation)**

   Use `Start-SdnServerCertificateRotation` from the SdnDiagnostics module to manually trigger certificate rotation and restore the NcHostAgent-to-ApiService connection.

   ```powershell
   # Import the SdnDiagnostics module if not already loaded
   Import-Module SdnDiagnostics

   # Generate the SdnDiagnostics credential (NC admin credentials)
   $credential = Get-Credential

   # Start the certificate rotation
   Start-SdnServerCertificateRotation -Credential $credential
   ```

   > **Note:** This is a short-term mitigation. The issue will recur on the next solution update unless the registry key (Step 1) is properly configured on all hosts before the update.

4. **Verify resolution**

   Confirm that the NcHostAgent service on each host can now communicate with the Network Controller.

   ```powershell
   # Check NcHostAgent service status on each host
   Get-Service -Name NcHostAgent | Select-Object Name, Status

   # Verify Network Controller connectivity by querying server resources
   $nchaParams = Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\NcHostAgent\Parameters"
   $ncUri = "https://$($nchaParams.PeerCertificateCName)"
   Get-NetworkControllerServer -ConnectionUri $ncUri
   ```

   A successful response listing server resources confirms that connectivity has been restored.

## Prevention

To prevent this issue from recurring on future solution updates:

- Ensure the `NetworkControllerNodeNames` registry key is populated on **all** Hyper-V hosts before performing a solution update
- Verify local administrator permissions on NC VMs are correctly configured before updates
- After any solution update, validate NcHostAgent connectivity to the Network Controller as part of post-update verification

## Related Issues

- [Troubleshoot: Host Not Connected to Controller](Troubleshoot-SDNExpress-HealthAlert-HostNotConnectedToController.md)
- [Troubleshoot: Host Unreachable](Troubleshoot-SDNExpress-HealthAlert-HostUnreachable.md)
- [SdnDiagnostics Wiki](https://github.com/microsoft/SdnDiagnostics/wiki)

---
