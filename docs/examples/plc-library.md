# Example: a PLC library

The package that installs a PLC library into the XAE library repository, so that it shows up under *Add Library* in every project on that PC. The example is the AdsGamepad library package from the ADS-Gamepad-Service project, `packaging\AdsGamepad.PlcLibrary` in that repository.

## The folder

```
AdsGamepad.PlcLibrary\
  AdsGamepad.PlcLibrary.nuspec
  LICENSE.txt
  docs\README.md                     the readme the GUI shows
  library\AdsGamepad.library         the library, saved from XAE
  tools\
    chocolateyinstall.ps1
    chocolateyuninstall.ps1
    chocolateybeforemodify.ps1
```

The library file is whatever you save from XAE through *Save as library*. A `.library` carries the source and installs with it; a `.compiled-library` installs without source. Both install the same way.

## The nuspec

```xml
<metadata>
  <id>Beckhoff-USA-Community.XAE.PLC.Lib.AdsGamepad</id>
  <version>2.3.0</version>
  <title>AdsGamepad PLC Library</title>
  <authors>Beckhoff Automation LLC</authors>
  <readme>docs/README.md</readme>
  <summary>TwinCAT PLC library for the ADS Gamepad Service</summary>
  <tags>Beckhoff TwinCAT XAE PLC Library AllowMultipleVersions</tags>
  <description>AdsGamepad is the TwinCAT 3 PLC library for the ADS Gamepad Service. ...</description>
  <dependencies>
    <dependency id="TwinCAT.Standard.XAE" version="0.0.0" />
  </dependencies>
</metadata>
<files>
  <file src="tools\**" target="tools" />
  <file src="library\AdsGamepad.library" target="tools\AdsGamepad.library" />
  <file src="docs\README.md" target="docs\README.md" />
  <file src="LICENSE.txt" target="LICENSE.txt" />
</files>
```

Two things to notice. The tag `AllowMultipleVersions` tells the Package Manager that two versions of this package may be installed at the same time, which is how PLC libraries work: a project pinned to 2.2.1 keeps compiling after 2.3.0 arrives. Beckhoff's own library packages carry the tag. And the package version is the library version. Keeping them equal is not required, but the uninstall script has to name the exact library version to remove, so it is one less thing to get wrong.

The dependency is `TwinCAT.Standard.XAE` at any version, because the install needs an engineering installation with the PLC component and nothing more specific.

## Installing

TwinCAT 4026 ships a command line tool, `RepTool.exe`, that installs a library into the repository exactly the way the Library Repository dialog does. The install script finds it, runs it, and then checks the result on disk:

```powershell
$ErrorActionPreference = 'Stop'

$toolsDir = Split-Path -Parent $MyInvocation.MyCommand.Definition
$libraryFile = Join-Path $toolsDir 'AdsGamepad.library'

# RepTool ships with every 4026 engineering installation.
$repTools = @(Join-Path 'C:\Program Files (x86)\Beckhoff\TwinCAT\3.1\Components\Plc\Build_4026.*\' 'Common\RepTool.exe' -Resolve)
if ($repTools.Length -eq 0) {
    throw 'No TwinCAT XAE 4026 installation with RepTool.exe was found. This package needs an engineering system.'
}

# Wildcard resolution sorts alphabetically, which puts Build_4026.9 after
# Build_4026.14, so the build number is compared numerically instead
$repTool = $repTools | Sort-Object { [version]((Split-Path (Split-Path (Split-Path $_ -Parent) -Parent) -Leaf) -replace '^Build_', '') } | Select-Object -Last 1
$buildName = Split-Path (Split-Path (Split-Path $repTool -Parent) -Parent) -Leaf
$repToolArgs = "--profile=`"TwinCAT PLC Control_$buildName`"", "--installLib `"$libraryFile`""
$proc = Start-Process $repTool -ArgumentList $repToolArgs -Wait -WindowStyle Hidden -PassThru
if ($proc.ExitCode -ne 0) {
    Write-Warning "RepTool returned code $($proc.ExitCode)."
}

# RepTool exit codes are not documented, so the outcome is checked directly:
# the library must exist in the repository after the install
$repoEntry = Join-Path $env:ALLUSERSPROFILE 'Beckhoff\TwinCAT\PlcEngineering\Managed Libraries\Beckhoff Community\AdsGamepad\2.3.0'
if (-not (Test-Path $repoEntry)) {
    throw "The library did not appear in the library repository under $repoEntry. Close XAE and install the package again."
}
Write-Host 'The AdsGamepad library is installed in the XAE library repository.'
```

The pieces worth keeping when you copy this:

* A PC can have more than one 4026 PLC build installed, and the folder names sort as text. The script picks the highest build by number, not by name.
* RepTool wants a profile name, `TwinCAT PLC Control_Build_4026.14` for example, and the build folder name is exactly the suffix it wants.
* RepTool's exit code is not documented and not trusted. The library repository is a folder tree under `C:\ProgramData\Beckhoff\TwinCAT\PlcEngineering\Managed Libraries\<Company>\<Name>\<Version>`, so the script checks that the folder exists afterwards. Company, name and version come from the library's project information in XAE; look them up there and put them in the script.
* XAE open during the install is the usual reason for a failure, and the error message says so.

## Uninstalling

RepTool removes a library by its identity string, `Name, Version (Company)`:

```powershell
$ErrorActionPreference = 'Stop'
$libraryIdentity = 'AdsGamepad, 2.3.0 (Beckhoff Community)'

$repTools = @(Join-Path 'C:\Program Files (x86)\Beckhoff\TwinCAT\3.1\Components\Plc\Build_4026.*\' 'Common\RepTool.exe' -Resolve)
if ($repTools.Length -gt 0) {
    $repTool = $repTools | Sort-Object { [version]((Split-Path (Split-Path (Split-Path $_ -Parent) -Parent) -Leaf) -replace '^Build_', '') } | Select-Object -Last 1
    $buildName = Split-Path (Split-Path (Split-Path $repTool -Parent) -Parent) -Leaf
    $repToolArgs = "--profile=`"TwinCAT PLC Control_$buildName`"", "--uninstallLib `"$libraryIdentity`""
    Start-Process $repTool -ArgumentList $repToolArgs -Wait -WindowStyle Hidden
}
Write-Host 'The AdsGamepad library was removed.'
```

The identity line is the one thing that has to move in step with the package version. When you bump the library, bump this line, the repository check in the install script, and the nuspec version together.

## Before an upgrade

XAE caches the content of the library repository, and a stale cache after a library change produces confusing errors in the next project that opens. The before-modify script clears it:

```powershell
$manLibDir = Join-Path $env:ALLUSERSPROFILE 'Beckhoff\TwinCAT\PlcEngineering\Managed Libraries'
if (Test-Path $manLibDir) {
    Get-ChildItem -Path $manLibDir -Filter 'cache*' -Force | Remove-Item -Recurse -Force
}
```

## Checking it

After the install, open XAE, open any PLC project, *References*, *Add library*, and find the library under its company. Or look at the folder the install script checks. After the uninstall, the folder is gone and the library is gone from the dialog. Those two checks, by hand, before the package goes on the feed.
