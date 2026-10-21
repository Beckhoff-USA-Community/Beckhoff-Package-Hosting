# What makes up a package

A TwinCAT package is a NuGet package. Rename a `.nupkg` to `.zip` and open it; everything on this page is in there. The Package Manager adds two conventions on top of NuGet: the metadata file describes the package the way the GUI wants to show it, and a `tools` folder holds the PowerShell scripts that do the actual install. Those scripts follow the Chocolatey naming, which is why they are called `chocolateyinstall.ps1` and not something Beckhoff picked.

## The folder

One folder per package, named after the package id. This is the certificate package from this repository:

```
BAUS.Development.OEMCertificate\
  BAUS.Development.OEMCertificate.nuspec           metadata
  BAUS.Development.OEMCertificate.ReleaseNote.md   the GUI's Feature Release Notes tab
  tools\
    chocolateyinstall.ps1                           runs on install and on upgrade
    chocolateyuninstall.ps1                         runs on uninstall
    chocolateybeforemodify.ps1                      runs before an upgrade or uninstall touches files
```

The payload, the files the package actually installs, goes either into `tools\` next to the scripts or into a folder of its own that the nuspec maps into the package. The PLC library example keeps its `.library` file in `library\`, the service example publishes its binaries into `bin\`. Where the payload sits in the source folder does not matter; where the nuspec puts it inside the package does, because the install script finds it relative to itself.

## The nuspec

The nuspec is XML. Every field the GUI shows comes from here, so the shape is worth knowing line by line. This is the certificate package's, shortened only in the comments:

```xml
<?xml version="1.0" encoding="utf-8"?>
<package xmlns="http://schemas.microsoft.com/packaging/2013/05/nuspec.xsd">
  <metadata>
    <id>BAUS.Development.OEMCertificate</id>
    <version>1.2.0</version>
    <title>Beckhoff USA AE Development Certificate</title>
    <authors>Beckhoff Automation LLC</authors>
    <projectUrl>https://github.com/0w8States/Iris-Manor-Custom-Nexus-Packages</projectUrl>
    <copyright>(c) Beckhoff Automation LLC</copyright>
    <requireLicenseAcceptance>false</requireLicenseAcceptance>
    <readme>BAUS.Development.OEMCertificate.ReleaseNote.md</readme>
    <summary>Installs and grants the Beckhoff USA OEM development certificate.</summary>
    <description>Installs the Beckhoff USA OEM development certificate into the TwinCAT certificate store, grants it with TcSignTool, and sets the TcSignTwinCatCertName and TWINCATCERTIFICATENAME environment variables.</description>
    <tags>Beckhoff TwinCAT XAE</tags>
    <dependencies>
      <dependency id="TwinCAT.XAE.Base" version="0.0.0" />
    </dependencies>
  </metadata>
  <files>
    <file src="tools\**" target="tools" />
    <file src="..\..\secrets\BAUS.Development.OEMCertificate\*.tccert" target="tools" />
    <file src="..\..\secrets\BAUS.Development.OEMCertificate\Password.txt" target="tools" />
    <file src="BAUS.Development.OEMCertificate.ReleaseNote.md" target="BAUS.Development.OEMCertificate.ReleaseNote.md" />
  </files>
</package>
```

Field by field:

`id`
:   The name the package is installed and upgraded by. Pick it once and never change it, because an upgrade only works when the id matches. The community convention is `<Vendor>.<XAE|XAR>.<Kind>.<Product>` for component packages, for example `Beckhoff-USA-Community.XAE.PLC.Lib.AdsGamepad`. This repository uses a shorter `BAUS.<Product>.<XAE|XAR>` because the feed is small. Both work; consistency within one feed is what matters. Ids starting with `TcPkg` are hidden by the GUI.

`version`
:   Three parts, `1.2.0`. NuGet normalises a four part version with a trailing zero (`1.2.0.0` becomes `1.2.0`), which makes a mess of folder names and feed listings, so stick to three. Every change to a published package needs a new version; feeds treat a version as immutable.

`title`
:   What the GUI prints on the card and in the list. Plain words, spaces allowed.

`description`
:   The GUI prints this raw, line breaks and all, on the card and on the Description tab. One or two plain sentences. No Markdown, no bullet points.

`summary`
:   Shown in `tcpkg list` output. Keep it to one line.

`readme`
:   The file the GUI renders on the Feature Release Notes tab. Beckhoff names it `<Id>.ReleaseNote.md`. See [the release notes file](#the-release-notes-file) below for what it may contain. The file must also be listed under `<files>` so it ends up in the package.

`tags`
:   Space separated words. The Package Manager reads some of them:

    | Tag | Meaning |
    |---|---|
    | `Beckhoff TwinCAT` | by convention every TwinCAT package starts with these two |
    | `XAE` / `XAR` | which side the package is for; informational on a component package |
    | `Workload`, `Category<Name>`, `VariantXAE` / `VariantXAR` | make a package a workload card, see [workloads](workloads.md) |
    | `AllowMultipleVersions` | more than one version of this package may be installed side by side (PLC libraries use it) |
    | `SystemServiceRestart` | the install restarts the TwinCAT System Service, so the GUI can warn the user |
    | `InstallationLevel1` | the package installs without administrator rights (Beckhoff uses it on PLC libraries and HMI packages) |

`dependencies`
:   What has to be installed first. A dependency on a Beckhoff package takes version `0.0.0`, which means any version, because you do not want your library package to refuse a newer TwinCAT. A dependency on one of your own packages usually pins the exact version, so that upgrading a workload upgrades its parts.

`requireLicenseAcceptance` and `license`
:   When `true`, the GUI shows the license file named by `<license type="file">LICENSE.txt</license>` and asks for acceptance. Community packages set it; packages for your own feed usually do not.

`files`
:   What goes into the package and where. `src` is relative to the nuspec, `target` is the path inside the package. `**` takes a folder tree. A `src` can reach outside the package folder, which is how this repository keeps a certificate and its password out of git: they live in the git-ignored `packaging\secrets\<Id>\` and the nuspec pulls them into `tools\` at pack time.

A workload adds `<packageTypes>` and the GUI icon fields; those are on the [workload page](workloads.md).

## The scripts

The Package Manager runs the scripts with Windows PowerShell 5.1, as administrator, from the folder the package was unpacked to. Three facts follow from that:

* Write the scripts for PowerShell 5.1. Nothing that only exists in PowerShell 7, no `??`, no `-SkipHttpErrorCheck`, no ternary operator. The build scripts in this repository are PowerShell 7; the package scripts are not.
* The package is unpacked under `C:\ProgramData\Beckhoff\TcPkg\lib\<Id>.<version>\`. `tools\` is in there, so a script finds its payload with `Split-Path -Parent $MyInvocation.MyCommand.Definition`. Do not hard code the path.
* The package manager is itself running while your script runs, and it is single instance. A script that calls `tcpkg` for anything but a read with `--ignore-mutex` fails with "TcPkg is already running".

The three scripts:

`chocolateyinstall.ps1`
:   Runs on install and again on upgrade. Set `$ErrorActionPreference = 'Stop'` at the top and `throw` on anything that means the install did not happen. A thrown error fails the install and the GUI shows the message, which is exactly what you want. Write it so running it twice is harmless; an upgrade is a second run over an installed package.

`chocolateyuninstall.ps1`
:   Runs on uninstall. Set `$ErrorActionPreference = 'Continue'` and never throw: a failing uninstall leaves the package half removed and the Package Manager still thinks it is installed. Remove what you put there, report what you could not, and finish. Remove only what is yours. The route file example checks for a marker comment before deleting a file, so a hand written file with the same name survives.

`chocolateybeforemodify.ps1`
:   Runs before an upgrade or an uninstall changes files. Stop a service here, or clear a cache, so the files are free when the next step touches them. Most packages leave a comment in it saying there is nothing to do, so that the next person knows it was considered.

Package parameters are the one way to hand a value in at install time: `tcpkg install <Id> --params "NoRestart=true"` or `--params-sensitive "CertPassword=..."`. Beckhoff documents them for the HMI Server package only and does not say how a package reads them. The packages here look for an environment variable whose name contains `PackageParameters` and parse `Name=Value` pairs out of it; that worked, but treat it as best effort and always have a default.

## The release notes file

The GUI renders the file named by `<readme>` on the Feature Release Notes tab with a very small Markdown parser. It understands `#`, `##` and `###` headings and plain paragraph lines. Bullet lists come out as `[object Object]`, bold and backticks print their markers, links print their brackets. So the file looks like this:

```markdown
## What this package installs
An ADS-over-MQTT route file that connects the TwinCAT ADS router to the broker.
The file is copied to the kernel-mode route folder and to the route folder of every Usermode Runtime.

## Requires
TwinCAT.XAE.Base (TwinCAT 3.1 Build 4026 engineering).

## Release history
### 1.2.0
The install scripts are now shared with the Runtime (XAR) package.
### 1.1.0
Refuses to install while the Runtime (XAR) variant is on the PC.
### 1.0.0
First release.
```

Headings and sentences. It reads fine, and it renders fine.

On a feed hosted on a NuGet server the GUI cannot read this file out of the package; it needs a copy hosted next to the feed. The [publishing page](../nexus/publishing.md) explains how that copy gets there. On Beckhoff's feed and on a local folder feed it just works.

## What the GUI shows, and where it comes from

| In the GUI | From |
|---|---|
| Card title | `<title>` |
| Text on the card and the Description tab | `<description>`, printed raw |
| Feature Release Notes tab | the file named by `<readme>` |
| Dependencies tab | `<dependencies>`, with their version ranges |
| Icon on the card | the stock icon for a package; a custom one only for a workload |
| Version list | every version of that id on the feed |

Verified against the Package Manager GUI 2.2 and 2.6 and Beckhoff's own packages. When in doubt, look at what Beckhoff's packages do and copy that.
