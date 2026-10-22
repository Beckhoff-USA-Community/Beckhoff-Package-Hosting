# Example: a .NET service

The package that installs a .NET application as a Windows service: copies the published binaries to Program Files, registers the service, starts it, and keeps the configuration file across upgrades. The example is the ADS Gamepad Service package from the public [ADS-Gamepad-Service](https://github.com/Beckhoff-USA-Community/ADS-Gamepad-Service) project, `packaging\AdsGamepadService.Service` in that repository. It is a worker service, not a web application; the [last section](#when-it-is-an-aspnet-application) says what changes when it is.

## The folder

```
AdsGamepadService.Service\
  AdsGamepadService.Service.nuspec
  LICENSE.txt
  docs\README.md
  bin\                               the publish output, written right before packing
  tools\
    chocolateyinstall.ps1
    chocolateyuninstall.ps1
    chocolateybeforemodify.ps1
```

`bin\` is build output, not source, so keep it out of source control. Publish the application into it, then pack:

```powershell
dotnet publish .\src\AdsGamepadService -c Release -r win-x64 --self-contained -o .\packaging\AdsGamepadService.Service\bin
New-Item -ItemType Directory -Force .\packaging\bin\release | Out-Null
tcpkg pack .\packaging\AdsGamepadService.Service\AdsGamepadService.Service.nuspec -o .\packaging\bin\release
```

The output folder is created first, because `tcpkg pack` does not create it. Self contained, so the target needs no .NET runtime installed. A runtime PC is not a developer PC, and the package should not depend on what happens to be on it. The package gets bigger; nobody has minded.

## The nuspec

```xml
<metadata>
  <id>Beckhoff-USA-Community.XAR.Service.AdsGamepad</id>
  <version>2.12.1</version>
  <title>ADS Gamepad Service</title>
  <authors>Beckhoff Automation LLC</authors>
  <requireLicenseAcceptance>true</requireLicenseAcceptance>
  <license type="file">LICENSE.txt</license>
  <readme>docs/README.md</readme>
  <summary>Gamepad to PLC bridge service for TwinCAT runtime systems</summary>
  <tags>Beckhoff Setup SystemServiceRestart</tags>
  <description>ADS Gamepad Service reads game controllers on a TwinCAT system and serves their state to the PLC over ADS. ...</description>
  <dependencies>
    <dependency id="TwinCAT.XAR.AdsApi" version="0.0.0" />
  </dependencies>
</metadata>
<files>
  <file src="tools\**" target="tools" />
  <file src="bin\**" target="tools\bin" />
  <file src="docs\README.md" target="docs\README.md" />
  <file src="LICENSE.txt" target="LICENSE.txt" />
</files>
```

The publish output goes to `tools\bin` inside the package, so the install script finds it next to itself. The dependency is the ADS API package, because the service talks ADS and needs the router; a service that does not would depend on `TwinCAT.XAR.Base` or on nothing at all.

## Installing

The whole script is in the ADS-Gamepad-Service repository. This is its spine:

```powershell
$ErrorActionPreference = 'Stop'

$toolsDir    = Split-Path -Parent $MyInvocation.MyCommand.Definition
$sourceDir   = Join-Path $toolsDir 'bin'
$installDir  = 'C:\Program Files\Beckhoff USA Community\ADS Gamepad\Service'
$serviceName = 'AdsGamepadService'
$exePath     = Join-Path $installDir 'AdsGamepadService.exe'

# An upgrade runs this script over a running service: stop it first.
$existing = Get-Service -Name $serviceName -ErrorAction SilentlyContinue
if ($existing -and $existing.Status -ne 'Stopped') {
    Stop-Service -Name $serviceName -Force
}

New-Item -ItemType Directory -Force -Path $installDir | Out-Null

# Copy the publish output. An existing appsettings.json is kept, so an
# upgrade never overwrites the configuration.
$keepConfig = Test-Path (Join-Path $installDir 'appsettings.json')
$robocopyArgs = @($sourceDir, $installDir, '/E')
if ($keepConfig) {
    Write-Host 'Keeping the existing appsettings.json.'
    $robocopyArgs += @('/XF', 'appsettings.json')
}
robocopy @robocopyArgs | Out-Null
if ($LASTEXITCODE -ge 8) {
    throw "Copying the application files failed, robocopy code $LASTEXITCODE."
}
cmd /c exit 0

if (-not $existing) {
    New-Service -Name $serviceName `
        -BinaryPathName ('"' + $exePath + '"') `
        -DisplayName 'ADS Gamepad Service' `
        -StartupType Automatic `
        -Description 'Bridges Xbox and PlayStation gamepads to TwinCAT PLCs over ADS.' | Out-Null
}

# Delayed start keeps the service from racing the TwinCAT router at boot
& sc.exe config $serviceName start=delayed-auto | Out-Null
if ($LASTEXITCODE -ne 0) { throw "Configuring the service start type failed, sc.exe code $LASTEXITCODE." }

# Restart automatically on failure: twice after five seconds, then after thirty
& sc.exe failure $serviceName reset=86400 actions=restart/5000/restart/5000/restart/30000 | Out-Null
if ($LASTEXITCODE -ne 0) { throw "Configuring service recovery failed, sc.exe code $LASTEXITCODE." }

Start-Service -Name $serviceName
Write-Host "ADS Gamepad Service is installed and running from $installDir."
```

The habits worth copying:

* **Stop before you copy.** A running executable cannot be overwritten. The before-modify script stops the service too, so the files are free whichever way the Package Manager gets here.
* **Keep the configuration.** The publish output carries a default `appsettings.json`. The install copies it on a fresh install and skips it when one is already there. An upgrade therefore never resets a setting somebody changed on the machine. The uninstall leaves it in place as well, so a later reinstall finds it again.
* **robocopy exit codes.** Codes below 8 are success with information, and PowerShell's `$ErrorActionPreference = 'Stop'` does not care about native exit codes anyway. The script checks `$LASTEXITCODE` itself and then runs `cmd /c exit 0` so the non zero code does not leak into whatever the Package Manager looks at afterwards.
* **Delayed start.** A service that needs the TwinCAT router at startup should not race it. `delayed-auto` is the cheap way.
* **Recovery actions.** `sc.exe failure` makes Windows restart the service when it dies, which is what you want for something that is supposed to be there all the time.
* **Quote the executable path.** `New-Service` takes the path with the quotes in it, or a path with spaces breaks the registration.

The real script also handles a move from an older install location: it reads the registered image path, and when that differs from the new location, it carries the configuration over, deletes the old registration and creates a new one. Read it when you rename an install folder.

## Before an upgrade and on uninstall

```powershell
# chocolateybeforemodify.ps1
$existing = Get-Service -Name 'AdsGamepadService' -ErrorAction SilentlyContinue
if ($existing -and $existing.Status -ne 'Stopped') {
    Stop-Service -Name 'AdsGamepadService' -Force
    Start-Sleep -Seconds 2
}
```

```powershell
# chocolateyuninstall.ps1
$ErrorActionPreference = 'Continue'

$installDir  = 'C:\Program Files\Beckhoff USA Community\ADS Gamepad\Service'
$serviceName = 'AdsGamepadService'

$existing = Get-Service -Name $serviceName -ErrorAction SilentlyContinue
if ($existing) {
    if ($existing.Status -ne 'Stopped') { Stop-Service -Name $serviceName -Force }
    & sc.exe delete $serviceName | Out-Null
    if ($LASTEXITCODE -ne 0) { Write-Warning "Removing the service registration failed, sc.exe code $LASTEXITCODE. Run 'sc.exe delete $serviceName' by hand." }
}

# Program files are removed but an edited appsettings.json is kept so a
# later reinstall finds the configuration again
if (Test-Path $installDir) {
    Get-ChildItem $installDir -Force | Where-Object Name -ne 'appsettings.json' | Remove-Item -Recurse -Force
}
```

`sc.exe delete` marks the service for deletion; it is gone once the last handle closes, which is why a reinstall right after an uninstall sometimes needs a second of patience, and the real install script sleeps two seconds after a delete for that reason.

The uninstall script in the repository runs with `$ErrorActionPreference = 'Stop'` and throws when `sc.exe delete` fails. The version here reports and goes on, which is the rule from [step 5](../packages/scripts.md#step-5-chocolateyuninstallps1): a thrown error would leave the package half removed and still listed as installed.

## When it is an ASP.NET application

A web application as a Windows service is the same package with four additions. None of them belongs in the package script alone; the first two are in the application.

**Host it as a service.** The application has to tell the host it runs under the service control manager, or Windows kills it after thirty seconds for not reporting started. In `Program.cs`, with the package `Microsoft.Extensions.Hosting.WindowsServices`:

```csharp
var builder = WebApplication.CreateBuilder(new WebApplicationOptions
{
    Args = args,
    ContentRootPath = AppContext.BaseDirectory
});
builder.Host.UseWindowsService();
```

**Set the content root.** A Windows service starts with `C:\Windows\System32` as its working directory, so without the `ContentRootPath` line the application looks for `appsettings.json` and `wwwroot` in the wrong place and starts with no configuration. That is the single most common reason an ASP.NET service runs fine from the console and not as a service.

**Pin the port.** Put the listening address in the configuration the package ships, so it does not depend on launch settings that are not published:

```json
{
  "Kestrel": {
    "Endpoints": {
      "Http": { "Url": "http://0.0.0.0:5000" }
    }
  }
}
```

`0.0.0.0` listens on every interface, which is what a service on a controller usually wants. `localhost` would be reachable from that PC only.

**Open the firewall.** The install script adds a rule, the uninstall script removes it:

```powershell
# install
New-NetFirewallRule -DisplayName 'My Service (TCP 5000)' -Direction Inbound -Protocol TCP -LocalPort 5000 -Action Allow -Profile Any | Out-Null

# uninstall
Get-NetFirewallRule -DisplayName 'My Service (TCP 5000)' -ErrorAction SilentlyContinue | Remove-NetFirewallRule
```

Name the rule after the package so the uninstall finds exactly that rule and nothing else. Make the install idempotent: check with `Get-NetFirewallRule` before adding, because an upgrade runs the install script again and a second rule with the same name is harmless but untidy.

Everything else, the publish, the copy, the service registration, the kept configuration, is the script above.
