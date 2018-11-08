---
services: service-fabric
platforms: dotnet, windows
author: vipulm-msft
---

# Service Fabric Windows Patch Installer Application
Service Fabric Windows Patch Installer is a sample application for installing custom patches or software on [Microsoft Azure Service Fabric](https://azure.microsoft.com/services/service-fabric/) clusters.

> Use [Service Fabric Patch Orchestration Service (POA)](https://docs.microsoft.com/en-us/azure/service-fabric/service-fabric-patch-orchestration-application) as a supported mechanism for patching machines in a Service Fabric cluster.

This application should be used for:
- Installing a patch not available through Service Fabric Patch Orchestration Service
- Getting out of a stuck situation or a live site issue
- Installing or verifying the installation of custom software on the machine


## About this application
The application verifies and installs the patch described in the data package of the PatchInstallerService. 

Patch installer service provides a common framework installing, verifying, and reporting on the patches. 

A patch is described as a data package for the patch installer service. The package consists of an installation script `install.cmd` and an optional verification script `verify.cmd` along with patch binaries. 

The result of the verification and installation are reported as the health events on the node. If the patch is not found a `Warning` health event is reported on the node. If the patch is found then an `Ok` health event is reported on the node.

## Patches

### Patch Description
|Patch|Description|
|:-:|:-|
|EXAMPLE_PATCH|An example patch that shows how to build patches. |

### Patch Install and Verification Information
|Patch|Verification Control Parameter|Installation Control Parameter|Reboots after installation?|Other Considerations|
|:-:|:-:|:-:|:-:|:-|
|EXAMPLE_PATCH|`Verify_EXAMPLE_PATCH`|`Install_EXAMPLE_PATCH`|`No`|None|

## Usage

## Build and Deploy Application
If you do not have the patch application already deployed in the cluster, build the application and then deploy it in your cluster. Once deployed, you can selectively enable the verification and installation of particular patches by upgrading application with specific application parameters. 

[Setup your development environment with Visual Studio 2017](https://docs.microsoft.com/azure/service-fabric/service-fabric-get-started). 

* Open PowerShell command prompt and run `build.ps1` script. It should produce an output like below.

```PowerShell
PS E:\patch-installer-app> .\build.ps1
Using msbuild from C:\Program Files (x86)\Microsoft Visual Studio\2017\Professional\MSBuild\15.0\Bin\MSBuild.exe
  Restore completed in 59.06 ms for E:\patch-installer-app\src\PatchInstallerService\PatchInstallerService.cspro
  j.
  PatchInstallerService -> E:\patch-installer-app\src\PatchInstallerService\bin\Release\netcoreapp2.0\win7-x64\P
  atchInstallerService.dll
  PatchInstallerService -> E:\patch-installer-app\src\PatchInstallerService\bin\Release\netcoreapp2.0\win7-x64\P
  atchInstallerService.dll
  PatchInstallerService -> E:\patch-installer-app\src\PatchInstallerApp\pkg\Release\PatchInstallerServicePkg\Cod
  e\
  PatchInstallerApp -> E:\patch-installer-app\src\PatchInstallerApp\pkg\Release
PS E:\patch-installer-app>
```


* By default the script will create a `release` package of the application in `src\PatchInstallerApp\pkg\Release` folder. 

* Connect to the Service Fabric Cluster where you want to deploy the application using [`Connect-ServiceFabricCluster`](https://docs.microsoft.com/en-us/powershell/module/servicefabric/connect-servicefabriccluster?view=azureservicefabricps) PowerShell command. 

* Deploy the application using the following PowerShell command

```PowerShell
. src\PatchInstallerApp\Scripts\Deploy-FabricApplication.ps1 -ApplicationPackagePath 'src\PatchInstallerApp\pkg\Release' -PublishProfileFile 'src\PatchInstallerApp\PublishProfiles\Cloud.xml' -UseExistingClusterConnection
```
## Install Patches
### Patches that do not reboot the machine
Patches that do not reboot the machine can be enabled for verification and installation via application parameters at the time of the application deployment or by running an application upgrade after the deployment using the following commands.

* Connect to the Service Fabric Cluster using [`Connect-ServiceFabricCluster`](https://docs.microsoft.com/en-us/powershell/module/servicefabric/connect-servicefabriccluster?view=azureservicefabricps) PowerShell command. 

* Get the current set of application parameters of the application.

```PowerShell
 $app = Get-ServiceFabricApplication fabric:/PatchInstallerApp 
```

* Upgrade the application with the new set of application parameters that enables installation and verification of the patch. See the table in above section for the names of the parameters for each patch. 

For example, the command below enables installation and verification of the `EXAMPLE_PATCH`.

Create the new parameters application.
```PowerShell
$newAppParams = @{} 

foreach($p in $app.ApplicationParameters) { $newAppParams.add($p.Name,$p.Value) }

$newAppParams['Verify_EXAMPLE_PATCH']='true'
$newAppParams['Install_EXAMPLE_PATCH']='true'
```

Upgrade the application with new parameters. 
```PowerShell
Start-ServiceFabricApplicationUpgrade -ApplicationName fabric:/PatchInstallerApp -Appl
icationTypeVersion $app.ApplicationTypeVersion -ApplicationParameter $newAppParams -UnmonitoredAuto
```

The patch installer application runs the verification script and installation script if required. Check the health events of the nodes to see the status of the patch verification and installation.

### Patches that reboot the machine

Patches whose installation reboot the machine should not be enabled during the application deployment, but should be controlled via monitored upgrade.

* Connect to the Service Fabric Cluster using [`Connect-ServiceFabricCluster`](https://docs.microsoft.com/en-us/powershell/module/servicefabric/connect-servicefabriccluster?view=azureservicefabricps) PowerShell command. 

* Upgrade the application with the new set of application parameters that enables just the verification of the patch. See the table in above section for the names of the parameters for each patch. For example, the command below enables verification of the `EXAMPLE_PATCH`.
    
    ```PowerShell
    # Get current application parameters
    $app = Get-ServiceFabricApplication fabric:/PatchInstallerApp 

    # Create the new parameters application.
    $newAppParams = @{} 

    foreach($p in $app.ApplicationParameters) { $newAppParams.add($p.Name,$p.Value) }

    $newAppParams['Verify_EXAMPLE_PATCH']='true'

    # Upgrade the application with new parameters. 
    Start-ServiceFabricApplicationUpgrade -ApplicationName fabric:/PatchInstallerApp -ApplicationTypeVersion $app.ApplicationTypeVersion -ApplicationParameter $newAppParams -UnmonitoredAuto
    ```

Once the upgrade is completed, the nodes that do not have the patch installed will be in the `Warning` state because of the missing patch health report on them.

*  Upgrade the application with the new set of application parameters that enables the installation of the patch (See the table in above section for the names of the parameters for each patch), but this time perform monitored upgrade with `FailureAction` set to `Manual` (or manual upgrade). The `HealthCheckWaitDurationSec` must be greater than the sum of `ScanIntervalSeconds` and `ScanIntervalRandomizationSeconds` settings in the `Settings.xml` of the service. If the patch installation is supposed to take time, adjust the `HealthCheckWaitDurationSec` and `UpgradeDomainTimeoutSec` timeouts appropriately (see, [Start-ServiceFabricUpgrade](https://docs.microsoft.com/en-us/powershell/module/servicefabric/start-servicefabricapplicationupgrade?view=azureservicefabricps) command for details). As an example, the command below enables installation of the `EXAMPLE_PATCH`. 

    ```PowerShell
     # Get current application parameters
    $app = Get-ServiceFabricApplication fabric:/PatchInstallerApp 

    # Create the new parameters application.
    $newAppParams = @{} 

    foreach($p in $app.ApplicationParameters) { $newAppParams.add($p.Name,$p.Value) }

    $newAppParams['Install_EXAMPLE_PATCH']='true'

    # Upgrade the application with new parameters. 
    Start-ServiceFabricApplicationUpgrade -ApplicationName fabric:/PatchInstallerApp -ApplicationTypeVersion $app.ApplicationTypeVersion -ApplicationParameter $newAppParams -Monitored -FailureAction Manual  -HealthCheckWaitDurationSec 600
    ```

The patch is going to be installed upgrade domain by upgrade domain. Once the upgrade is completed, nodes that did not have the patch installed and were in the `Warning` state should become `Ok` again.

## Guidelines for contributing patch packages
* Do not remove existing patch packages

* Ensure that you have added application parameters with default value as `false` to control the installation and verification of the patch. See below for details.

* Provide description of the patch and instructions on how to enable the verification and installation of the patch. Indicate if the patch reboots the machine and requires safe installation procedure.

## Creating new patch package
To create a patch package, follow the instructions below.

### 1. Create patch data package
Create a data package that describes the patch.

- Open Visual Studio solution and go to `PackageRoot` folder of `PatchInstallerService`.

- Create a folder with the name of the patch (for example, see folder `EXAMPLE_PATCH`)

- Add a new data package in the `ServiceManifest.xml`. For example,
    ```XML
    <DataPackage Name="EXAMPLE_PATCH" Version="1.0.0" />

    ```
### 2. Add installation and verification scripts
The patch package consists of an installation script `install.cmd` and an optional verification script `verify.cmd` along with patch binaries. 

- Add `install.cmd` file under the data package folder created above. The installation file must have all necessary steps for installing the patch including rebooting the machine if necessary. Add all binaries used by the installation script to the data package as well. 


- Add `verify.cmd` file under the data package folder created above. This file is required for patches that **can not** be verified using `Get-WmiObject -Class Win32_QuickFixEngineering | SELECT HotFixID,InstalledOn` PowerShell command. The `verify.cmd` file verifies if the patch is installed on the machine or not. If the patch is installed, the verification is successful and the script must return `0` exit code. The script must return a non-zero exit code if the verification fails or if the patch is not found. If the patch can be found using `Get-WmiObject -Class Win32_QuickFixEngineering | SELECT HotFixID,InstalledOn` PowerShell command, then its name must match the `HotFixID` name returned by this command.

### 3. Enable verification and installation of the patch
The  `PatchVerificationSettings` and `PatchInstallationSettings` controls the verification and installation of the specific patches. Add the name of the patch and using a boolean value of `true` or `false`, enable or disable the installation and verification in `Settings.xml` file of the service. For example,

```XML
  <Section Name="PatchVerificationSettings">
    <Parameter Name="EXAMPLE_PATCH" Value="false" />
  </Section>

  <Section Name="PatchInstallationSettings">
    <Parameter Name="EXAMPLE_PATCH" Value="false" />
  </Section>
```

Add parameters to override and control the behavior of the patch verification and installation in the application manifest (`ApplicationManifest.xml`) with default value of `false`. For example,

For example,
```XML
  <Parameters>

    ...  
    <Parameter Name="Verify_EXAMPLE_PATCH" DefaultValue="false" />
    <Parameter Name="Install_EXAMPLE_PATCH" DefaultValue="false" />
    
  </Parameters>

...
      <ConfigOverride Name="Config">
        <Settings>
          <Section Name="PatchVerificationSettings">
            <Parameter Name="EXAMPLE_PATCH" Value="[Verify_EXAMPLE_PATCH]" />
          </Section>
          <Section Name="PatchInstallationSettings">
            <Parameter Name="EXAMPLE_PATCH" Value="[Install_EXAMPLE_PATCH]" />
          </Section>
        </Settings>
      </ConfigOverride>
```

This way user can control the verification and installation of specific patch at the time of deployment. They also allow controlled installation of the patches that reboot the machine via application upgrade.