# Example: plain files

Most packages do not install a program. They put a file where TwinCAT looks for it and, sometimes, tell TwinCAT to look. This page has three of those: a folder of files that needs nothing more, a certificate that also needs a tool run and environment variables set, and a route file that needs a service restart and a rule the metadata cannot express. The certificate and the route file come from packages built for Beckhoff USA's own feed.

## A folder of files

The simplest package there is: copy a folder, delete it on uninstall. It is the package the [Building a package](../packages/nuspec.md) section builds step by step, so the scripts are there.

The same shape installs a TcCOM module: the module package of the public [ADS-Gamepad-Service](https://github.com/Beckhoff-USA-Community/ADS-Gamepad-Service) project copies a folder tree into `C:\ProgramData\Beckhoff\TwinCAT\3.1\Repository\<Vendor>\<Module>\<Version>\`, checks that the `.tmc` file arrived, and removes exactly that version folder on uninstall. TwinCAT finds it there with no further step.

## A certificate

An OEM certificate package. Signing a TwinCAT C++ or MATLAB module needs a certificate in the TwinCAT certificate store, its password granted to the Windows user through `TcSignTool`, and two environment variables that name the certificate. Four steps, and each one has a catch.

```powershell
$ErrorActionPreference = 'Stop'

$toolsDir = Split-Path -Parent $MyInvocation.MyCommand.Definition
$certDir  = Join-Path $env:ProgramData 'Beckhoff\TwinCAT\3.1\CustomConfig\Certificates'
$envNames = @('TcSignTwinCatCertName', 'TWINCATCERTIFICATENAME')

# ---- the certificate and the password, bundled from the secrets folder at pack time ----
$certSource = Get-ChildItem -Path $toolsDir -Filter '*.tccert' -File | Select-Object -First 1
if (-not $certSource) { throw "No .tccert file found in $toolsDir. The package was built without the certificate." }
$certFile   = $certSource.Name
$certTarget = Join-Path $certDir $certFile

$passFile = Join-Path $toolsDir 'Password.txt'
$certPassword = Get-PackageParameterValue 'CertPassword'          # --params-sensitive, when given
if (-not $certPassword -and (Test-Path $passFile)) { $certPassword = Get-Content -Path $passFile -TotalCount 1 }
if (-not $certPassword) { throw 'No certificate password: pass --params-sensitive "CertPassword=<password>" or build the package with Password.txt.' }

# ---- TcSignTool: 4026 keeps it under Program Files (x86), 4024 under C:\TwinCAT ----
$candidates = @("${env:ProgramFiles(x86)}\Beckhoff\TwinCAT\3.1\SDK\Bin\TcSignTool.exe", 'C:\TwinCAT\3.1\SDK\Bin\TcSignTool.exe')
$tcSignTool = $candidates | Where-Object { Test-Path $_ } | Select-Object -First 1
if (-not $tcSignTool) { throw "TcSignTool.exe not found. Install TwinCAT.XAE.Base first." }

# ---- 1. copy ----
New-Item -ItemType Directory -Force -Path $certDir | Out-Null
Copy-Item -Path $certSource.FullName -Destination $certTarget -Force

# ---- 2. grant ----
# TcSignTool (4026.14) returns exit code 1 on success AND on failure, and it does not validate the
# password (a wrong one shows up at the first signed build). The output text is the only signal.
$grantOutput = (& $tcSignTool grant /f "$certTarget" /p "$certPassword" 2>&1 | ForEach-Object { "$_" }) -join "`n"
Write-Host $grantOutput
if ($grantOutput -notmatch '(?m)^granted password for ') { throw "TcSignTool grant did not succeed: $grantOutput" }

# ---- 3. environment variables, machine wide and for this user ----
foreach ($name in $envNames) {
    foreach ($scope in 'Machine', 'User', 'Process') {
        [System.Environment]::SetEnvironmentVariable($name, $certFile, [System.EnvironmentVariableTarget]::$scope)
    }
}

# ---- 4. do not leave the password lying around ----
if (Test-Path $passFile) { Remove-Item -Path $passFile -Force -ErrorAction SilentlyContinue }

Write-Host 'Reboot or sign out and back in so Visual Studio and MATLAB see the new environment variables.'
```

`Get-PackageParameterValue` is the helper shown under [package parameters](../packages/scripts.md#package-parameters).

The catches:

* **The exit code lies.** `TcSignTool grant` returns 1 whether it worked or not. The script reads the output and looks for the success line. Whenever you call a tool from a script, find out what its exit code means before you trust it.
* **The grant does not check the password.** It stores whatever you give it; a wrong password fails the first signed build, not the install. There is nothing a script can do about that except say so.
* **The grant is per Windows user.** TcSignTool stores it under `HKCU`, so install the package as the user who will build the projects.
* **Environment variables need a new sign in.** Visual Studio and MATLAB read them at start. The script sets them in all three scopes and then tells the user to sign out, because that is the one part it cannot do.
* **The password travels in the package.** It comes from a secrets folder outside source control, is bundled at pack time, is read once and deleted from the unpacked package folder. It is still in the `.nupkg` on the feed. That is acceptable only because the feed is LAN only; see [secrets in a package](../packages/packing.md#secrets-in-a-package).

The uninstall revokes the grant (`TcSignTool grant /f <cert> /r`), deletes the certificate file, and clears the variables by setting them to `$null`. It never throws.

## A route file

An ADS-over-MQTT route package. ADS-over-MQTT lets TwinCAT systems find each other through an MQTT broker instead of static routes; the configuration is one XML file in the router's route folder. Beckhoff documents it at [infosys](https://infosys.beckhoff.com/content/1033/tc3_ads_over_mqtt/). The file looks like this, with the login filled in:

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<!-- BAUS.AdsOverMqtt.XAE: installed by the TwinCAT Package Manager from the MyFeed feed. Reinstall the package instead of editing this file. -->
<TcConfig xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="http://www.beckhoff.com/schemas/2015/12/TcConfig">
  <RemoteConnections>
    <Mqtt>
      <Name>MyNetwork</Name>
      <Address Port="1883">mqtt.example.com</Address>
      <Topic>MyNetwork</Topic>
      <User>YourUser</User>
      <Pwd>YourPwd</Pwd>
    </Mqtt>
  </RemoteConnections>
</TcConfig>
```

The comment line on top is not decoration. It names the package, and the uninstall script removes only a file that carries it.

Three things make this harder than "copy a file":

**TwinCAT has two route folders.** The kernel mode router reads `<TwinCAT3Dir>\Target\Routes`, which is `C:\Program Files (x86)\Beckhoff\TwinCAT\3.1\Target\Routes` on 4026. The Usermode Runtime reads `%ProgramData%\Beckhoff\TwinCAT\3.1\Runtimes\<runtime>\3.1\Target\Routes`, one per runtime. The package copies the file into every folder it finds, whatever mode the PC is in, so switching the mode later needs no reinstall.

```powershell
function Get-RouteDirectories {
    $dirs = [ordered]@{}
    $tcDir = $env:TwinCAT3Dir
    if ($tcDir) { $dirs['KM'] = Join-Path $tcDir 'Target\Routes' }
    $umRoot = Join-Path $env:ProgramData 'Beckhoff\TwinCAT\3.1\Runtimes'
    foreach ($rt in (Get-ChildItem -Path $umRoot -Directory -ErrorAction SilentlyContinue)) {
        $target = Join-Path $rt.FullName '3.1\Target'
        if (Test-Path $target) { $dirs["UM:$($rt.Name)"] = Join-Path $target 'Routes' }
    }
    return $dirs
}
```

**The router reads route files only when it starts.** Copying the file does nothing until the TwinCAT System Service restarts. The script restarts `TcSysSrv`, and that works in both modes: in kernel mode the service hosts the router, and in usermode the service process is the Usermode Runtime's own system service, so it restarts with it. Verified on 4026.27 with the runtime in UM mode: the broker connection was up a few seconds after the restart, no reboot. A running PLC stops for that moment, so the package parameter `NoRestart=true` skips it for the user who would rather reboot at a time of their choosing.

```powershell
$svc = Get-Service -Name 'TcSysSrv' -ErrorAction SilentlyContinue
if ($svc) {
    Restart-Service -Name 'TcSysSrv' -Force -ErrorAction Stop
    (Get-Service -Name 'TcSysSrv').WaitForStatus('Running', [TimeSpan]::FromSeconds(60))
}
```

**Two variants must not meet.** There is an engineering route file with engineering credentials and a runtime one with runtime credentials, in two packages, and a PC must have one or the other. A nuspec cannot say that, so the install script checks before it copies: is the other package's folder under `C:\ProgramData\Beckhoff\TcPkg\lib`, and does any route folder hold a file with the other package's marker comment?

```powershell
function Test-TcPkgInstalled([string]$Id) {
    $lib = Join-Path $env:ProgramData 'Beckhoff\TcPkg\lib'
    foreach ($dir in (Get-ChildItem -Path $lib -Directory -ErrorAction SilentlyContinue)) {
        if ($dir.Name -match ('^(?i)' + [regex]::Escape($Id) + '\.\d') -and (Get-ChildItem -Path $dir.FullName -Filter '*.nupkg' -File -ErrorAction SilentlyContinue)) { return $true }
    }
    return $false
}
```

It reads the disk rather than asking `tcpkg list`, because `tcpkg` is running this very script and a second instance would fail on the single-instance lock.

The two packages share every script byte for byte; only `AdsOverMqtt.Variant.ps1` differs, with the package id, the other variant's ids and the marker text. Keep those copies identical, so a fix in one package cannot be forgotten in the other. When two packages are the same thing with different data, build them that way from the start.

The uninstall walks the same folders, removes the file only where the marker matches, and restarts the service once when it removed anything.
