# Install, upgrade and uninstall

The nuspec says what a package is. The three scripts in `tools\` do the work. This page writes them for the `MyCompany.XAE.Docs.MyMachine` package from [the previous page](nuspec.md), and then shows the commands a user runs to install, upgrade and remove it.

## How the scripts run

What follows was measured with `tcpkg` 2.6.24 and a test package whose three scripts wrote down where, when and how they ran.

* **As administrator.** Installing and uninstalling run elevated.
* **With the Package Manager's own PowerShell.** `tcpkg` 2.6.24 carries PowerShell 7.4 in its program folder and runs the scripts with it, whatever the PC has installed: the test scripts reported 7.4.14 on a PC whose own PowerShell 7 was 7.6. Earlier Package Manager versions ran them with Windows PowerShell 5.1, where PowerShell 7 syntax fails with a parse error. Your package may meet either, so write for both: nothing that only exists in PowerShell 7, such as `??`, the ternary operator or `-SkipHttpErrorCheck`.
* **From a folder that does not stay.** `chocolateyinstall.ps1` runs from the Package Manager's download cache, `%TEMP%\TcPkg\cache\<id in lower case>\<version>\tools\`, which is cleaned up afterwards. `chocolateybeforemodify.ps1` and `chocolateyuninstall.ps1` run from `C:\ProgramData\Beckhoff\TcPkg\lib\<Id>.<version>\tools\`, which the Package Manager unpacks from the stored `.nupkg` for the occasion; between installs, that folder holds only the `.nupkg`. So a script finds its payload relative to itself, with `Split-Path -Parent $MyInvocation.MyCommand.Definition`, and copies it to where it belongs. Never leave anything pointing at the script's own folder: a service registered to run from there loses its executable.
* **With the caller's working directory.** The working directory is wherever `tcpkg` was started from, `C:\` in the test, not the package folder. Build every path from the script's folder or an absolute location.
* **Knowing who it is.** `$env:ChocolateyPackageName` and `$env:chocolateyPackageVersion` hold the id and the version of the package whose script is running.
* **Inside `tcpkg`.** The Package Manager is running your script, and it is single instance. A second `tcpkg` from a script fails with "TcPkg is already running", unless it only reads and passes `--ignore-mutex`. To check whether a package is installed, look for its folder under `C:\ProgramData\Beckhoff\TcPkg\lib\` instead.

Which script runs when:

| The user | Runs |
|---|---|
| installs a package | `chocolateyinstall.ps1` of that version |
| installs a version that is already installed | nothing; `tcpkg` says the package is already installed |
| upgrades | `chocolateybeforemodify.ps1` of the installed, older version, then `chocolateyinstall.ps1` of the new version |
| goes back with `tcpkg install <Id>=<version> --allow-downgrade` | the same as an upgrade: before-modify of the installed version, install of the older one |
| uninstalls | `chocolateybeforemodify.ps1`, then `chocolateyuninstall.ps1`, both of the installed version |

Three things follow from the table:

* **An upgrade never runs the uninstall script.** It is a second run of the install script over the files of the old version. Write the install script so that running it over an older install is harmless. When a new version drops a file, the install script has to remove it; copying over the old files leaves it behind.
* **The before-modify script that runs on an upgrade is the old one.** A fix to it reaches a PC one upgrade later.
* **Package parameters are not remembered.** On the upgrade, the parameters given at install time were gone; see [package parameters](#package-parameters).

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

* **`$ErrorActionPreference = 'Continue'`, and never throw.** A failing uninstall leaves the package half removed while the Package Manager still lists it as installed. Remove what you can, report what you could not, and finish.
* **Remove only what is yours.** This package owns `Documentation` and nothing else. The product and company folders are shared with your other packages, so they go only when they are empty: the last package out turns off the light. When a package writes a single file into a folder it does not own, mark the file (a comment line naming the package) and check the mark before deleting. The [route file example](../examples/files.md#a-route-file) does that.
* **Leave what the user changed, if they would want it back.** The [service example](../examples/dotnet-service.md) keeps an edited configuration file, so a reinstall finds it again.

## Step 6: chocolateybeforemodify.ps1

```powershell
# Runs before an upgrade or an uninstall changes this package's files.
# Nothing to stop or release for a folder of documents.
```

This script runs before an upgrade or an uninstall touches the files. It exists for packages whose files are in use: stop a Windows service so its executable can be overwritten, or clear a cache that would go stale. The [service example](../examples/dotnet-service.md#before-an-upgrade-and-on-uninstall) stops its service here, and the [PLC library example](../examples/plc-library.md#before-an-upgrade) clears the XAE library cache. When there is nothing to do, keep the file with a comment that says so, so the next person knows it was considered.

## Package parameters

A user can hand values to the scripts at install time:

```powershell
tcpkg install <Id> --params "NoRestart=true"
tcpkg install <Id> --params-sensitive "CertPassword=..."
```

The first is from the [route file example](../examples/files.md#a-route-file), which skips its service restart when asked, the second from the [certificate example](../examples/files.md#a-certificate). `--params-sensitive` is the variant `tcpkg` offers for secrets such as passwords. Beckhoff documents parameters for the HMI Server package only, and does not say how a package reads them. `tcpkg` 2.6.24 hands them to the scripts in the environment variables `chocolateyPackageParameters` and `packageParameters`, after a parameter of its own. With `--params "Color=Blue;Size=2"` both read:

```
/DISABLEUPDATECONFIGURATION=1  Color=Blue;Size=2
```

On a later `tcpkg upgrade` without `--params`, they held only `/DISABLEUPDATECONFIGURATION=1`. Parameters given at install time are not remembered.

The packages behind this guide read a value with this helper, which looks in every environment variable whose name contains `PackageParameters`:

```powershell
function Get-PackageParameterValue([string]$Name) {
    foreach ($var in (Get-ChildItem env: | Where-Object { $_.Name -like '*PackageParameters*' })) {
        $text = [string]$var.Value
        if ($text -match "(?i)(?:^|[\s;/])$Name\s*[=:]\s*(?:`"([^`"]*)`"|'([^']*)'|(\S+))") {
            return @($Matches[1], $Matches[2], $Matches[3]) | Where-Object { $_ } | Select-Object -First 1
        }
    }
}
```

Separate parameters with spaces: `--params "Color=Blue Size=2"`. The helper reads a value up to the next space, so given `Color=Blue;Size=2` it returns `Blue;Size=2` for `Color`. Treat parameters as best effort, and always have a default, if only for the upgrade.

## What the user runs

From an administrator PowerShell, or with the matching buttons in the Package Manager GUI:

```powershell
tcpkg install MyCompany.XAE.Docs.MyMachine -n MyFeed -y           # latest version from the feed MyFeed
tcpkg install MyCompany.XAE.Docs.MyMachine=1.0.0 -n MyFeed -y     # one particular version
tcpkg upgrade MyCompany.XAE.Docs.MyMachine -n MyFeed -y           # to the latest version
tcpkg uninstall MyCompany.XAE.Docs.MyMachine -y
```

`tcpkg` needs the administrator shell because installing and uninstalling run your scripts as administrator. The GUI runs without administrator rights and asks for elevation when an action needs it. `-n` names the feed to install from, and `-y` skips the confirmation prompt. `--dry-run` on `install` resolves the dependencies and prints what would happen without touching the PC; it needs an elevated shell all the same. `--show-files` lists the files of the package and whether they were installed. `tcpkg list -i` lists what is installed and `tcpkg list -o` what has a newer version on the feed; the GUI shows the same on its *Updates* tab.

For a workload, uninstalling is different; see [workloads](workloads.md#upgrading-and-uninstalling).

## When something goes wrong

* The output of `tcpkg` includes what the scripts printed, and the error text of a `throw`.
* The full log, with the command lines, is `C:\ProgramData\Beckhoff\TcPkg\logs\TcPkg.log`.
* "Running Microsoft Visual Studio or Beckhoff TwinCAT XAE Shell (64) process found" (exit code 1051) means what it says: close the engineering and install again.
* "TcPkg is already running" means another `TcPkg.exe` holds the single-instance lock. The GUI keeps a `TcPkg.exe` child process, and closing the GUI can leave it behind. Check with `Get-Process TcPkg`, and stop it only when the GUI is really gone.

Next: [versioning](versioning.md).
