# Install, upgrade and uninstall

The nuspec says what a package is. The scripts in `tools\` do the work. This page writes them for the `MyCompany.XAE.Docs.MyMachine` package from [the previous page](nuspec.md), and then shows the commands a user runs to install, upgrade and remove it.

## How the scripts run

What follows was measured with `tcpkg` 2.6.24 and test packages whose scripts wrote down where, when and how they ran.

* **As administrator.** Installing and uninstalling run elevated.
* **With the Package Manager's own PowerShell.** `tcpkg` 2.6.24 carries PowerShell 7.4 in its program folder and runs the scripts with it, whatever the PC has installed: the test scripts reported 7.4.14 on a PC whose own PowerShell 7 was 7.6, and they did so although `tcpkg` itself was started from Windows PowerShell 5.1. Which shell you call `tcpkg` from does not matter. Older Package Manager versions were not checked for this guide. If your package may meet one, keep to what Windows PowerShell 5.1 also understands; that means leaving out what only PowerShell 7 has, such as `??`, the ternary operator or `-SkipHttpErrorCheck`.
* **From a folder that does not stay.** `chocolateyinstall.ps1` runs from the Package Manager's download cache, `%LOCALAPPDATA%\Temp\TcPkg\cache\<id in lower case>\<version>\tools\`, which is cleaned up afterwards. `chocolateybeforemodify.ps1` and `chocolateyuninstall.ps1` run from `C:\ProgramData\Beckhoff\TcPkg\lib\<Id>.<version>\tools\`, which the Package Manager unpacks from the stored `.nupkg` for the occasion; between installs, that folder holds only the `.nupkg`. So a script finds its payload relative to itself, with `Split-Path -Parent $MyInvocation.MyCommand.Definition`, and copies it to where it belongs. Never leave anything pointing at the script's own folder: a service registered to run from there loses its executable.
* **With the caller's working directory.** The working directory is wherever `tcpkg` was started from, not the package folder. Build every path from the script's folder or an absolute location.
* **With Chocolatey's helper functions loaded.** The Package Manager runs every script through a Chocolatey script runner, and the functions Chocolatey packages use, `Install-ChocolateyPackage`, `Get-PackageParameters`, `Set-PowerShellExitCode` and the rest, are there; see [the helper functions](#the-helper-functions).
* **Inside `tcpkg`.** The Package Manager is running your script, and it is single instance. A second `tcpkg` from a script fails with "TcPkg is already running", unless it only reads and passes `--ignore-mutex`. To check whether a package is installed, look for its folder under `C:\ProgramData\Beckhoff\TcPkg\lib\` instead.

### What the Package Manager tells the script

A set of environment variables, set for every script run. The test package printed them all; these are the ones worth reading:

| Variable | Holds | Example |
|---|---|---|
| `ChocolateyPackageName`, `packageName` | the id of the package whose script is running | `MyCompany.XAE.Docs.MyMachine` |
| `chocolateyPackageVersion`, `packageVersion` | its version | `1.0.0` |
| `packageFolder` | the folder the package is unpacked in; the script's own folder is `tools\` under it | `...\cache\mycompany.xae.docs.mymachine\1.0.0` |
| `chocolateyPackageParameters`, `packageParameters` | the `--params` string, after a parameter of the Package Manager's own | `/DISABLEUPDATECONFIGURATION=1  /Color:Blue` |
| `chocolateyPackageParametersSensitive` | the `--params-sensitive` string; not set when none was given | `/Secret:...` |
| `previousInstalledPackageName`, `previousInstalledPackageVersion` | on an upgrade or a downgrade, in the install script of the incoming version: the id and version being replaced. Not set on a first install, and not set in the before-modify or uninstall scripts. | `1.0.0` |
| `TcPkgCliVersion`, `TcPkgCliThreePartVersion` | the Package Manager's version | `2.6.24` |
| `TcPkgCliFeatureKMUM`, `TcPkgCliFeatureKMUMXOR`, `TcPkgCliFeaturePackageDeprecation`, `TcPkgCliFeaturePackageDisclaimer` | `True` on a Package Manager that supports the feature; unset on one that does not. Beckhoff's own scripts refuse an old Package Manager with these; see [the helper functions](#the-helper-functions). | `True` |
| `TcPkgInstall`, `TcPkgData` | the Package Manager's program folder and data folder | `C:\Program Files\Beckhoff\TcPkg\`, `C:\ProgramData\Beckhoff\TcPkg\` |
| `MSI_LOG`, `MSI_LOG_SUFFIX` | the folder the Package Manager keeps MSI logs in, and a timestamp for a log file name | `C:\ProgramData\Beckhoff\TcPkg\logs\msi`, `20261027_111845` |

`previousInstalledPackageVersion` is the one that changes how you write an upgrade: the install script of 2.0.0 can tell that it is replacing 1.0.0 and do what the plain copy would not, such as removing a file that 1.0.0 shipped and 2.0.0 does not.

### Which script runs when

| The user | Runs |
|---|---|
| installs a package | `chocolateyinstall.ps1` of that version |
| installs a version that is already installed | nothing; `tcpkg` says the package is already installed |
| upgrades | `chocolateybeforemodify.ps1` of the installed, older version, then `chocolateyinstall.ps1` of the new version |
| goes back with `tcpkg install <Id>=<version> --allow-downgrade` | the same as an upgrade: before-modify of the installed version, install of the older one. `tcpkg` reports it as an upgrade, "2.0.0 -> 1.0.0" |
| uninstalls | `chocolateybeforemodify.ps1`, then `chocolateyuninstall.ps1`, both of the installed version |
| installs a package that [replaces this one](lifecycle.md#retiring-an-id) | before-modify and uninstall of this package, then the install script of the new one |

Three things follow from the table:

* **An upgrade never runs the uninstall script.** It is a second run of the install script over the files of the old version. Write the install script so that running it over an older install is harmless. When a new version drops a file, the install script has to remove it; copying over the old files leaves it behind, and `previousInstalledPackageVersion` tells the script which version it is cleaning up after.
* **The before-modify script that runs on an upgrade is the old one.** A fix to it reaches a PC one upgrade later.
* **Package parameters are not remembered.** On an upgrade, the parameters given at install time are gone; see [package parameters](#package-parameters).

## Step 4: chocolateyinstall.ps1

```powershell
$ErrorActionPreference = 'Stop'

# The package is unpacked to a folder of the Package Manager's choosing.
# The files to install sit next to this script, so find them from here.
$toolsDir   = Split-Path -Parent $MyInvocation.MyCommand.Definition
$sourceDir  = Join-Path $toolsDir 'docs'
$installDir = 'C:\Program Files\MyCompany\MyMachine\Documentation'

if (-not (Test-Path $sourceDir)) {
    throw "The package has no docs folder next to $toolsDir. It was packed without its files."
}

New-Item -ItemType Directory -Force -Path $installDir | Out-Null
Copy-Item -Path (Join-Path $sourceDir '*') -Destination $installDir -Recurse -Force

Write-Host "The MyMachine documentation is installed in $installDir."
```

The rules it follows:

* **`$ErrorActionPreference = 'Stop'` at the top, and `throw` when the install did not happen.** A thrown error fails the install, and the GUI and `tcpkg` show the message. Make the message say what to do about it.
* **Check the payload is there.** A `<file src>` in the nuspec that matched nothing at pack time leaves the folder missing. The check turns that into a clear error instead of an empty install.
* **Safe to run over an older version.** `New-Item -Force` and `Copy-Item -Force` do not mind that the folder and the files already exist, so an upgrade overwrites them. A file that a newer version no longer ships would stay behind; when that matters, delete the folder before copying.
* **Say what happened.** Every `Write-Host` line shows up in the output of `tcpkg install` and in the Package Manager log.

## Step 5: chocolateyuninstall.ps1

```powershell
$ErrorActionPreference = 'Continue'

$installDir = 'C:\Program Files\MyCompany\MyMachine\Documentation'
$productDir = Split-Path $installDir -Parent
$vendorDir  = Split-Path $productDir -Parent

if (Test-Path $installDir) {
    Remove-Item -Path $installDir -Recurse -Force
}

# Other packages from MyCompany may share these folders. Remove them only when empty.
foreach ($dir in $productDir, $vendorDir) {
    if ((Test-Path $dir) -and -not (Get-ChildItem -Path $dir -Force)) {
        Remove-Item -Path $dir -Force
    }
}

Write-Host 'The MyMachine documentation was removed.'
```

The rules it follows:

* **It is the only thing that removes your files.** The script is optional: a test package with an install script and no uninstall script uninstalled without complaint, the Package Manager removed its own record and stored copy, and the folder the install script had created was still there afterwards. Most of Beckhoff's packages have no uninstall script or an empty one, because their payload is an MSI that Windows Installer knows how to remove. A package that copies files has no such help, so it needs this script.
* **`$ErrorActionPreference = 'Continue'`, and never throw.** A failing uninstall leaves the package half removed while the Package Manager still lists it as installed. Remove what you can, report what you could not, and finish.
* **Remove only what is yours.** This package owns `Documentation` and nothing else. The product and company folders are shared with your other packages, so they go only when they are empty: the last package out turns off the light. When a package writes a single file into a folder it does not own, mark the file (a comment line naming the package) and check the mark before deleting. The [route file example](../examples/files.md#a-route-file) does that.
* **Leave what the user changed, if they would want it back.** The [service example](../examples/dotnet-service.md) keeps an edited configuration file, so a reinstall finds it again.

## Step 6: chocolateybeforemodify.ps1

```powershell
# Runs before an upgrade or an uninstall changes this package's files.
# Nothing to stop or release for a folder of documents.
```

This script runs before an upgrade or an uninstall touches the files. It exists for packages whose files are in use: stop a Windows service so its executable can be overwritten, or clear a cache that would go stale. The [service example](../examples/dotnet-service.md#before-an-upgrade-and-on-uninstall) stops its service here, and the [PLC library example](../examples/plc-library.md#before-an-upgrade) clears the XAE library cache. It is optional like the uninstall script, and most of Beckhoff's are empty. When there is nothing to do, keep the file with a comment that says so, so the next person knows it was considered.

## The helper functions

The Package Manager loads Chocolatey's helper functions before it runs a script, so [Chocolatey's documentation](https://docs.chocolatey.org/en-us/create/functions/) of them applies. Three are worth knowing.

**`Install-ChocolateyPackage` runs an installer.** Nearly every Beckhoff package is an MSI, and its install script is this template, from `TwinCAT.XAE.PLC.Lib.Tc2_Math` with the names changed:

```powershell
$ErrorActionPreference = 'Stop'
$toolsDir     = Split-Path -Parent $MyInvocation.MyCommand.Definition
$fileLocation = Join-Path $toolsDir 'MyMachine-Tools.msi'

$packageArgs = @{
  packageName   = $env:ChocolateyPackageName
  unzipLocation = $toolsDir
  fileType      = 'MSI'
  file          = $fileLocation
  softwareName  = 'MyCompany MyMachine Tools'
  silentArgs    = "/qn /norestart /l*v `"$($env:TEMP)\$($env:ChocolateyPackageName).$($env:chocolateyPackageVersion).log`""
  validExitCodes= @(0, 3010, 1641)
}
Install-ChocolateyPackage @packageArgs
```

`softwareName` is the display name in *Programs and Features*, `silentArgs` the switches for a silent install with a log, and `validExitCodes` the codes that count as success, here 0 and the two Windows Installer codes for "done, reboot needed". If your payload is an MSI, use this instead of `Start-Process msiexec`. The uninstall of an MSI package is Windows Installer's job; Beckhoff's MSI packages mostly carry no uninstall script.

**`Get-PackageParameters` parses the package parameters**; see the next section.

**`Set-PowerShellExitCode` sets the exit code of a failing script.** Beckhoff's packages that need a Package Manager feature refuse an old Package Manager with it, and the GUI shows the message:

```powershell
if (-not $env:TcPkgCliFeaturePackageDisclaimer) {
    Set-PowerShellExitCode 15603
    throw "A newer version of the TwinCAT Package Manager is required to install $env:ChocolateyPackageName $env:chocolateyPackageVersion."
}
```

## Package parameters

A user can hand values to the scripts at install time:

```powershell
tcpkg install MyCompany.XAE.Docs.MyMachine -n MyFeed --params "/Color:Blue /Size:2 /Quiet"
tcpkg install MyCompany.XAE.Cert.Signing -n MyFeed --params-sensitive "/CertPassword:..."
```

`--params-sensitive` is the variant for secrets such as passwords; the [certificate example](../examples/files.md#a-certificate) uses it. Beckhoff documents parameters for the HMI Server package only, but the way a script reads them is in every Beckhoff package that takes one: the helper `Get-PackageParameters`, which the Package Manager loads for every script.

```powershell
$pp = Get-PackageParameters
$color = if ($pp['Color']) { $pp['Color'] } else { 'Green' }
if ($pp['Quiet']) { ... }
```

What it does, measured with `tcpkg` 2.6.24 and `--params "/Color:Blue /Size=2 /Quiet Plain=1" --params-sensitive "/Secret:hush"`:

* It returns one hashtable with `Color = Blue`, `Size = 2`, `Quiet = True` and `Secret = hush`, the normal and the sensitive string merged. The forms it reads are `/Name:value`, `/Name=value` and a bare `/Switch`, which becomes `True`.
* **`Plain=1` was not in the hashtable.** A parameter without the leading slash is ignored. Write every parameter as `/Name:value`.
* The hashtable also holds `DISABLEUPDATECONFIGURATION = 1`. That is the Package Manager's own parameter, which it puts in front of yours on every run; Beckhoff's Visual Studio packages read it. Ignore it.
* Always have a default. On a later `tcpkg upgrade` without `--params`, the scripts see only the Package Manager's parameter: parameters given at install time are not remembered. `tcpkg upgrade --params` hands new parameters to both the old version's before-modify script and the new version's install script.
* **Never print a sensitive parameter.** The Package Manager masks the `--params-sensitive` value in its own log line, `--params-sensitive *****`, but every line a script writes with `Write-Host` goes into `C:\ProgramData\Beckhoff\TcPkg\logs\TcPkg.log` as it is. A test script that printed its hashtable put the secret in the log.

Earlier versions of this guide carried a hand-written parser for the `Name=value` form. The built-in one is what Beckhoff's packages use; follow it.

## What the user runs

From an administrator PowerShell, or with the matching buttons in the Package Manager GUI:

```powershell
tcpkg install MyCompany.XAE.Docs.MyMachine -n MyFeed -y           # latest version from the feed MyFeed
tcpkg install MyCompany.XAE.Docs.MyMachine=1.0.0 -n MyFeed -y     # one particular version
tcpkg upgrade MyCompany.XAE.Docs.MyMachine -n MyFeed -y           # to the latest version
tcpkg uninstall MyCompany.XAE.Docs.MyMachine -y
```

`tcpkg` needs the administrator shell because installing and uninstalling run your scripts as administrator. The GUI runs without administrator rights and asks for elevation when an action needs it. `-n` names the feed to install from, and `-y` skips the confirmation prompt. `--dry-run` on `install` resolves the dependencies and prints what would happen without touching the PC; it needs an elevated shell all the same. `--show-files` lists the files of the package and whether they were installed. `tcpkg list -i` lists what is installed and `tcpkg list -o` what has a newer version on the feed; the GUI shows the same on its *Updates* tab.

In a script or a remote session, `-y` is not optional. `tcpkg` does not read answers from a pipe: without `-y` it stops with "Input is not possible from a non-interactive shell. You can pass '-y' to bypass the confirmation prompt." and exit code 1223. A package that shows an [upgrade disclaimer](lifecycle.md#a-disclaimer-before-a-major-upgrade) needs `--accept-string` as well; `-y` does not accept a disclaimer.

For a workload, uninstalling is different; see [workloads](workloads.md#upgrading-and-uninstalling).

## When something goes wrong

* The output of `tcpkg` includes what the scripts printed, and the error text of a `throw`.
* The full log, with the command lines, is `C:\ProgramData\Beckhoff\TcPkg\logs\TcPkg.log`.
* "TcPkg is already running" means another `TcPkg.exe` holds the single-instance lock. The GUI keeps a `TcPkg.exe` child process, and closing the GUI can leave it behind. Check with `Get-Process TcPkg`, and stop it only when the GUI is really gone.

The exit codes seen while writing this guide, all with `tcpkg` 2.6.24:

| Exit code | Message | Meaning |
|---|---|---|
| 1051 | "Running Microsoft Visual Studio or Beckhoff TwinCAT XAE Shell (64) process found." | close the engineering and install again, or tag the package `NoVisualStudioRestart` |
| 1223 | "Input is not possible from a non-interactive shell", "User aborted operation", "You need to accept the disclaimer" | a prompt that could not be answered: add `-y`, or `--accept-string` |
| 574 | "Package integrity verification failed" | an unsigned or untrusted package with [signature verification](../hosting/signatures.md) on |
| 4201 | "The provided accept string does not match the required accept string" | a wrong `--accept-string` |
| 4204 | `tcpkg verify`: "Failure_MissingP7S", "Failure_NoSignerCertificatePathValidates" | no signature, or a signer this PC does not trust |
| 15603 | the script's own message | what Beckhoff's scripts set with `Set-PowerShellExitCode` when the Package Manager is too old |

Next: [versioning](versioning.md).
