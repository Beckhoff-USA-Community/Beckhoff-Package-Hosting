# Building and testing

Building a package is one command. Everything around it, the checks before and the test after, is what keeps a broken package off the feed. This page walks through the bare commands first and then through what the build script in this repository adds.

## What you need

* The TwinCAT Package Manager on the PC you build on. It puts `tcpkg` on the PATH, and `tcpkg pack` is the packer. On a PC without it, the `nuget` command line tool packs the same nuspec: `nuget pack <nuspec> -OutputDirectory <folder> -NoPackageAnalysis`.
* PowerShell 7 for the build scripts in this repository (they declare `#Requires -Version 7.0`). The package scripts themselves stay PowerShell 5.1, see [the scripts](anatomy.md#the-scripts).
* The Package Manager GUI closed. `tcpkg` is single instance, and the GUI runs `tcpkg` for every click. The GUI also keeps a `TcPkg.exe` child process around that holds the single-instance lock, and closing the GUI can leave it behind. When a pack or a `tcpkg source` command fails with "TcPkg is already running", look for it:

```powershell
Get-Process TcPkg
```

Stop it only when the GUI is really gone.

## Packing

```powershell
tcpkg pack .\packaging\packages\BAUS.Development.OEMCertificate\BAUS.Development.OEMCertificate.nuspec -o .\packaging\feed
```

That writes `BAUS.Development.OEMCertificate.1.2.0.nupkg` into `packaging\feed`. The id and version come from the nuspec; the output folder is whatever you name. Pack component packages before workloads, so that when you test the workload its parts are already on the feed.

Then look inside. A `.nupkg` is a zip:

```powershell
Add-Type -AssemblyName System.IO.Compression.FileSystem
$zip = [System.IO.Compression.ZipFile]::OpenRead('.\packaging\feed\BAUS.Development.OEMCertificate.1.2.0.nupkg')
$zip.Entries | Select-Object FullName, Length
$zip.Dispose()
```

What you want to see: the nuspec, `tools\` with the three scripts, the payload where the install script expects it, and the release notes file at the path `<readme>` names. What you do not want to see: a payload that was supposed to be there and is not. A nuspec `<file src>` that matches nothing is silently skipped by the packer, and the package fails on the target instead of on your desk.

## Testing on a local folder feed

A folder of `.nupkg` files is a feed. Register it once:

```powershell
tcpkg config unset -n VerifySignatures
tcpkg source add -n Local -s C:\feed --priority 2
tcpkg source list
```

The first line turns signature checking off, because your packages carry no Beckhoff signature. It has to come before the feed is added, and it stays off on that PC. Beckhoff's own feed is priority 1; the number just orders the list.

Use a local drive for the folder. When the build output lives on a network share, copy the files to `C:\feed` first; an elevated `tcpkg` does not always see a mapped drive, and on one PC starting an elevated shell with a network drive as its working directory made the elevation itself fail. Start elevated shells with `-WorkingDirectory C:\`.

Then, from an administrator PowerShell:

```powershell
tcpkg list -n Local --no-cache
tcpkg install BAUS.Development.OEMCertificate -n Local --dry-run -y
tcpkg install BAUS.Development.OEMCertificate -n Local -y
tcpkg uninstall BAUS.Development.OEMCertificate -y
```

The dry run resolves dependencies and tells you what would be installed without touching the PC; it needs an elevated shell even though it changes nothing. The real install runs your script. Read its output: the Package Manager prints what the script printed, and every `Write-Host` you put in the script is a line in that output. The full log, including the command lines, is at `C:\ProgramData\Beckhoff\TcPkg\logs\TcPkg.log`.

Open the GUI afterwards, pick the feed *Local* in the feed selector, and look at the card: title, description, icon, release notes, dependencies. The GUI is what your users see, and it is pickier than the command line about the metadata.

To test a change, bump the version, pack again, and install again; the Package Manager treats a version as immutable and will not pick up a changed file with the same version. On a local folder feed you can delete the old file and clear the cache instead, but once you do that from habit you will one day do it on a server where it is impossible, so bump the version.

`tcpkg` caches feed metadata for about 30 minutes, in `%LOCALAPPDATA%\NuGet\v3-cache`. `--no-cache` on a `list` or `install` bypasses it; deleting the folder for that feed resets it for the GUI too.

## What the build script checks

`packaging\scripts\Build-Packages.ps1` packs every folder under `packaging\packages` and lints each nuspec first. The lint is the list of mistakes that have already happened once:

* The version has three parts.
* The id does not start with `TcPkg` (the GUI hides those).
* A workload has `<packageType name="Workload"/>`, the three tags `Workload`, `Category<X>` and `VariantXAE` or `VariantXAR`, an `icon.png` that is 64 by 64 pixels, and an `<iconUrl>` that matches where the icon will be hosted (read from `feed.psd1`).
* A package has no `<iconUrl>`; the GUI never shows it.
* `<readme>` names a file that exists.
* Every `<file src>` matches at least one file. This is the one that catches a missing certificate, password or route file in `packaging\secrets`, so a package is never packed without the thing it exists to install.
* A dependency on a Beckhoff package uses `0.0.0` unless a floor is intended (a note, not an error).
* The two route packages, which share their scripts, have byte identical copies of every shared script. A fix in one cannot be forgotten in the other.

After packing it lists what landed in `tools\` of each package, so you see the bundled files with your own eyes.

```powershell
cd .\packaging\scripts
.\Build-Packages.ps1 -Clean                       # everything
.\Build-Packages.ps1 -Only BAUS.AdsOverMqtt.XAE   # one package
```

`-Clean` empties `packaging\feed` first. The script refuses to run while a `TcPkg.exe` is running, which is the single-instance lock again.

## Secrets at build time

Some packages exist to install something secret: a certificate with its password, a route file with a broker login. Those files must not be in git, and they must be in the package. The arrangement in this repository:

* The secret files live in `packaging\secrets\<Id>\`, and `packaging\secrets\*` is git-ignored except for its README and an example credentials file.
* The nuspec reaches over with a relative `src`: `<file src="..\..\secrets\BAUS.Development.OEMCertificate\*.tccert" target="tools" />`. The packer resolves it relative to the nuspec and the file lands in `tools\` inside the package.
* The build lint fails when the file is missing, so a clone without the secrets cannot build a hollow package by accident.
* The install script treats a missing file as an error with a message that says where it should have come from.

The built package contains the secret in plain text, and so does the feed. That is fine on a LAN only feed where everyone who can read the feed is allowed to have the secret, and not fine anywhere else. The [pitfalls page](../pitfalls.md) says it again.
