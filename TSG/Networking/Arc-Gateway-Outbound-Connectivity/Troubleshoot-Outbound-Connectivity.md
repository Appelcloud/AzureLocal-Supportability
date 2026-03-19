# Azure Local - Troubleshoot Outbound Network Connectivity

## Overview

Connected instances of Azure Local require outbound / egress network connectivity from the management network of each Azure Local instance to a list of public endpoints. Network connectivity to these endpoints is required for Azure Local to use Azure as a reliable management and control plane, such as for initial instance deployment, applying updates and for workload provisioning operational capabilities. Allowing connectivity to the list of endpoints is a prerequisite for deployment, but ongoing connectivity to the list of endpoints is vital for support, manageability and licensing compliance. The list of required endpoints varies depending on the customer scenario, for example the list of endpoints is reduced when using an Azure Arc Gateway, and varies (slightly) based on which Azure region is selected.

For additional information on Azure Local Firewall requirements, please review - [Azure Local Firewall documentation](https://learn.microsoft.com/azure/azure-local/concepts/firewall-requirements).

## Symptoms

Integrating Azure Local into your existing Firewall and/or Proxy Server infrastructure can be challenging depending on your organization's network security policies, such as requirements to define a strict access control list of the URL endpoints that are allowed to communicate from your Azure Local instance(s) "management network" to the required public endpoints.

There are several symptoms or issues that will occur if the required endpoints are not accessible from the Azure Local instance(s), below are just a few examples:

1. Azure Local instance cloud deployment and/or update operations fail with network failure or timeout.
1. Physical machines show their "Azure Arc" status as "Disconnected" in Azure portal experience (_Arc agent not connected_).
1. Deployment of new Azure Local VMs fails with RPC failure in Azure portal as the ARM deployment status.
1. Updates fail at "update ARB and extensions" step with "SSL: CERTIFICATE_VERIFY_FAILED" shown in Azure portal update blade.
1. There are many other network related issues that could occur when the required endpoints are not accessible from the management (_physical machines and ARB_) network address space.

## Issue Validation

Azure Local includes built-in automation modules that are executed as part of Solution Update Readiness and Environment Checker modules, both of these modules include connectivity tests that validate network connectivity status to critical endpoints. The output of these can be viewed locally using PowerShell, or using Azure portal during updates.

> Note: For examples of how to use and view the output of Azure Local "Solution Update Environment" and "Environment Checker" built-in modules, see the [Appendix section](#appendix) at the bottom of this article.

If you are finding it difficult to isolate the cause of the network related issue(s) or failure(s), and are experiencing issues as described in 'Symptoms' section, you can follow the steps in the 'Mitigation Details' below to gain further insights and diagnostic data to validate the required endpoints are accessible from your on-premises network.

## Mitigation Details

To help with troubleshooting or root causing network connectivity issues, you can use the **Test-AzureLocalConnectivity** function which is included in the **AzStackHCI.DiagnosticSettings** module. This function can help automate testing that connectivity is working correctly from Azure Local physical machines to the required public endpoints. The function supports Arc Gateway scenarios and has an `-AzureRegion` parameter to allow testing against a specific Azure region that matches your Azure Local instance deployment.

The 'Test-AzureLocalConnectivity' function has a dependency on the Azure Local Environment Checker module being installed, which is installed by default on all Azure Local physical machines. If Environment Checker module (_AzStackHci.EnvironmentChecker_) is not installed on the device running the connectivity test, you will be prompted to install the module first. The device used to install the AzStackHCI.DiagnosticSettings module and test connectivity must have access to the PowerShell Gallery, in order to download the module (_nuget package_) to install it.

### Install and run connectivity tests

To install the AzStackHCI.DiagnosticSettings module and perform connectivity tests for a support or troubleshooting scenario, use the commands below:

```PowerShell
# Install the AzStackHCI.DiagnosticSettings module, this can be on an Azure Local
# physical machine (recommended), or any device inside your network (if it is using
# the same firewall / proxy configuration as your Azure Local instance).
Install-Module -Name "AzStackHci.DiagnosticSettings" -Repository PSGallery

# Test Azure Local Connectivity for a specific target Azure region.
# /// ACTION: Update <AzureRegionName> and <YourKeyVaultName> to match the values
# of your Azure Region and Key Vault.
Test-AzureLocalConnectivity -AzureRegion "<AzureRegionName>" -KeyVaultURL "https://<YourKeyVaultName>.vault.azure.net"

# Optional parameters for more detailed output, add: "-Verbose" and "-Debug" to the
# function above, which will output full diagnostic level responses from the remote
# endpoint web server.
# The output from the function is automatically saved in the PowerShell transcript.
```

### Arc Gateway deployments

If your Azure Local deployment uses Arc Gateway, use the `-ArcGatewayDeployment` and `-ArcGatewayURL` parameters together. When `-ArcGatewayDeployment` is specified, the function only tests URLs that do **not** support Arc Gateway (i.e., the endpoints that must remain directly accessible even with Arc Gateway enabled).

```PowerShell
# Test connectivity for Arc Gateway deployment.
# Both -ArcGatewayDeployment and -ArcGatewayURL are required together.
# /// ACTION: Update parameters below to match your Azure Region, Key Vault, and Arc Gateway URL.
Test-AzureLocalConnectivity -AzureRegion "<AzureRegionName>" `
    -KeyVaultURL "https://<YourKeyVaultName>.vault.azure.net" `
    -ArcGatewayDeployment `
    -ArcGatewayURL "https://<YourArcGatewayID>.gw.arc.azure.com"
```

### OEM hardware partner endpoints

Hardware detection based on your hardware OEM vendor is built into the module automatically, however if you want to override this or target a specific OEM's endpoints you can use the `-IncludeOEMUrls` parameter:

```PowerShell
# Include OEM-specific endpoints for your hardware vendor.
# Valid values: DataOn, Dell, HPE, Hitachi, Lenovo, TestAll
Test-AzureLocalConnectivity -AzureRegion "<AzureRegionName>" `
    -KeyVaultURL "https://<YourKeyVaultName>.vault.azure.net" `
    -IncludeOEMUrls "<YourOEMPartner>"
```

### Supported Azure regions

For the most recent / up to date list of supported Azure regions review the ["Azure requirements" - System requirements for Azure Local](https://learn.microsoft.com/azure/azure-local/concepts/system-requirements-23h2#azure-requirements) article. At the time of publishing this article, the list of valid Azure Region names for Azure Local include:

* `EastUS`, `WestEurope`, `AustraliaEast`, `CanadaCentral`, `CentralIndia`, `JapanEast`, `SouthCentral`, `SouthEastAsia`, `USGovVirginia`

### Testing an individual endpoint

If you would like to test an individual public endpoint using PowerShell for troubleshooting or support purposes, you can use the **Test-Layer7Connectivity** function with the `-Debug` switch. Example syntax is shown below:

```PowerShell
# Install the "AzStackHci.DiagnosticSettings" module
Install-Module -Name "AzStackHci.DiagnosticSettings" -Repository PSGallery

# To test an individual endpoint (after installing the module), with
# Verbose and Debug output, use the "Test-Layer7Connectivity" function, as shown below:
$url = 'https://graph.microsoft.com/v1.0/'
Test-Layer7Connectivity -url $url -port 443 -Verbose -Debug
```

## Output format

### HTML report (default)

The function now generates an **HTML report** by default (replacing the previous CSV format). The HTML report includes:

* **Color-coded rows** — Failed endpoints are highlighted in red, successful in green, and skipped in yellow for quick visual identification.
* **Summary section** — Hostname, timestamp, Azure region, hardware OEM, and download speed are displayed at the top of the report.
* **Scrollable table** — A synchronized dual-scrollbar table allows horizontal scrolling of the wide results table from both the top and bottom.
* **Full endpoint details** — Each row includes the URL, port, Arc Gateway support status, source, IP address, Layer 7 status, response, response time, certificate chain details (leaf, intermediate, root), and notes.

### JSON output (always generated)

A JSON output file is **always generated** in addition to the primary report format. The JSON file includes all test results plus summary metadata (hostname, timestamp, Azure region, hardware OEM, download speed). This is useful for programmatic analysis or integration with monitoring tools.

### CSV format (optional)

To generate a CSV file instead of HTML, use the `-OutputFormat` parameter:

```PowerShell
Test-AzureLocalConnectivity -AzureRegion "EastUS" -OutputFormat CSV
```

### Output file location

All output files are saved to: `C:\ProgramData\AzStackHci.DiagnosticSettings\`

Output files generated:

| File | Description |
| `AzureLocal_ConnectivityTest_<Region>_<Hostname>_<DateTime>.html` | HTML report (default) or `.csv` if `-OutputFormat CSV` is used |
| `AzureLocal_ConnectivityTest_<Region>_<Hostname>_<DateTime>.json` | JSON test results with summary metadata (always generated) |
| `Transcript_AzureLocal_ConnectivityTest_<Region>_<Hostname>_<DateTime>.log` | PowerShell transcript log |

## Share test results with Microsoft (Optional)

The 'Test-AzureLocalConnectivity' function includes an option to upload the test results to Microsoft, this is controlled by a User Prompt that asks if you would like to **Upload the Transcript file and report file to Microsoft**. If you **answer "Y"** to the prompt, the function will automatically upload the output files to Microsoft, the transfer uses the built-in log transfer method that uses secure protocols, more information on the upload process is available [here](https://learn.microsoft.com/azure/azure-local/manage/collect-logs?view=azloc-24113&tabs=powershell#about-on-demand-log-collection).

To skip the upload prompt, use the `-ExcludeUploadResults` switch.

If you are working with Microsoft customer service and support (CSS), and have a support request (SR) case open, you could share some of the "Share Test Results" log upload text output that shows your cluster's "AEORegion", "ARODeviceARMResourceUri" and "CorrelationId" with the SR case owner.

## Demo and example output

Example output is shown in the animated GIF image below, which shows an interactive console demo.

The primary source of information is **opening the HTML output file** in a web browser on your laptop or desktop PC. The HTML report provides an interactive, color-coded view of all test results. Alternatively, use the JSON output for programmatic analysis or the CSV format for spreadsheet workflows.

![Test-AzureLocalConnectivity Demo](./images/Test-AzureLocalConnectivity_Demo.gif)

### Test-AzureLocalConnectivity function parameters

```PowerShell
[CmdletBinding(DefaultParameterSetName = 'Default')]
param (
    # Target Azure region for endpoint testing.
    [ValidateSet("EastUS", "WestEurope", "AustraliaEast", "CanadaCentral",
                 "CentralIndia", "JapanEast", "SouthCentral", "SouthEastAsia",
                 "USGovVirginia")]
    [string]$AzureRegion,

    # Custom KeyVault URL including https:// prefix.
    # Example: https://yourhcikeyvaultname.vault.azure.net
    [System.Uri]$KeyVaultURL,

    # Switch to ONLY test URLs that do NOT support Arc Gateway.
    # Must be used together with -ArcGatewayURL (mandatory parameter set).
    [switch]$ArcGatewayDeployment,

    # Custom Arc Gateway URL including https:// prefix.
    # Must be used together with -ArcGatewayDeployment (mandatory parameter set).
    # Example: https://1be59945-12c0-4cda-9580-84a66a1120a0.gw.arc.azure.com
    [System.Uri]$ArcGatewayURL,

    # Custom DNS Name for NTP Time Server (no http:// or https:// prefix).
    # Example: yourtimeserver.fqdn, this can be your on-premises AD domain FQDN, if using AD integrated NTP service (PDC).
    [string]$NTPTimeServer,

    # Include tests for TCP connectivity (for scenarios not using a Proxy).
    [switch]$IncludeTCPConnectivityTests,

    # Exclude testing of Redirected endpoints.
    [switch]$ExcludeRedirectedUrls,

    # Exclude testing manually defined subdomains for Wildcard endpoints.
    [switch]$ExcludeWildcardTests,

    # Exclude the prompt to upload test results to Microsoft.
    [switch]$ExcludeUploadResults,

    # Include OEM hardware partner specific endpoints.
    # Valid values: DataOn, Dell, HPE, Hitachi, Lenovo, TestAll
    [string]$IncludeOEMUrls,

    # Skip automatic module version check against PowerShell Gallery.
    [switch]$NoAutoUpdate,

    # Suppress all console output from the function.
    [switch]$NoOutput,

    # Return the $Results array object for further processing in PowerShell.
    [switch]$PassThru,

    # Output report format. Default is HTML. CSV is also available.
    # JSON is always generated in addition to the selected format.
    [ValidateSet('HTML', 'CSV')]
    [string]$OutputFormat = 'HTML'
)
```

### Key parameter changes from previous versions

| Change | Details |
|--------|---------|
| `-ArcGatewayDeployment` and `-ArcGatewayURL` | Now a **mandatory parameter set** — both must be specified together. Previously they were independent optional parameters. |
| `-OutputFormat` | **New parameter.** Controls the report format: `HTML` (default) or `CSV`. JSON is always generated alongside. |
| `-IncludeOEMUrls` | **New parameter.** Allows testing OEM hardware partner specific endpoints (DataOn, Dell, HPE, Hitachi, Lenovo, or TestAll). |
| `-NoAutoUpdate` | **New parameter.** Skips automatic version check against PowerShell Gallery. |
| `-NoOutput` | **New parameter.** Suppresses all console output for automation/scripting scenarios. |
| `USGovVirginia` | **New Azure region** added to the `-AzureRegion` validated set. |
| HTML output | **Default output format changed** from CSV to HTML with color-coded rows and summary section. |
| JSON output | **Always generated** alongside the primary report format. |
| Download speed test | Now uses **parallel multi-session downloads** for more accurate bandwidth measurement. |
| Private Link detection | **New feature.** Detects and warns if endpoints resolve to RFC1918 private IP addresses (possible Private Link configuration). |

## Appendix

### Environment Checker Connectivity Tests

To view the output from Azure Local **Environment Checker** Connectivity Validation tests, use the PowerShell command below:

```PowerShell
Invoke-AzStackHciConnectivityValidation -PassThru | Where-Object -Property Status -eq FAILURE | Sort-Object TargetResourceName | Format-Table TargetResourceName -Autosize
```

For additional information for how to use Azure Local Environment Checker module, review the [Troubleshooting External Connectivity Failures in Environment Checker](../../EnvironmentValidator/Troubleshooting-External-Connectivity-Failures-in-Environment-Checker.md) article.

And the Microsoft Learn article is here: [Readiness of your environment for Azure Local - "Run readiness checks" section](https://learn.microsoft.com/azure/azure-local/manage/use-environment-checker?view=azloc-24113&tabs=connectivity#run-readiness-checks).

### Solution Update Environment Tests

To view the output from **all tests** included Azure Local **Solution Update Readiness**, which includes connectivity validation and tests for critical public endpoints, use the PowerShell command below:

```PowerShell
# Check Solution Update Environment
$Result = Get-SolutionUpdateEnvironment -FullHealthCheckDetails

# View "not equal to SUCCESS" alerts
$Result.HealthCheckResult | Where-Object {$_.Status -ne "SUCCESS"} | Format-List Title, Status, Severity, Description, AdditionalData, Remediation

# Create "C:\Temp" folder, if it does not exist
if(-not(Test-Path "C:\Temp\")) { New-Item -Path "C:\Temp\" -Type Directory | Out-Null }

# Output to Text format
$Result.HealthCheckResult | Out-File "C:\Temp\HealthResult-$((Get-Cluster).Name).txt"

# Output to JSON format
$Result.HealthCheckResult | ConvertTo-Json -Depth 10 | Out-File "C:\Temp\HealthResult-$((Get-Cluster).Name).json"
```

For additional information for how to analyze and understand the **$Results.HealthCheckResult** array, refer to this article: [Solution Update Readiness Checker - "using PowerShell" section](https://learn.microsoft.com//azure/azure-local/update/update-troubleshooting-23h2?view=azloc-24113#using-powershell).

## How to get additional support

If you need assistance with connectivity, please open a Support Request (SR) case with Microsoft CSS support using Azure portal.
