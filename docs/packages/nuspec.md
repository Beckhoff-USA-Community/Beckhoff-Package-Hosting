# The folder and the nuspec

A TwinCAT package is a NuGet package: a `.nupkg` file, which is a zip. Rename one to `.zip` and open it, and everything this section describes is in there. The Package Manager adds two conventions on top of NuGet. The metadata file describes the package the way the GUI shows it, and a `tools` folder holds PowerShell scripts that do the actual install. Those scripts follow the Chocolatey naming, which is why they are called `chocolateyinstall.ps1` and not something Beckhoff picked.

This page and the three after it build one package, from an empty folder to an installed package. The steps are numbered across the pages:

* Steps 1 to 3, the folder, the nuspec and the release notes: this page
* Steps 4 to 6, the three scripts: [install, upgrade and uninstall](scripts.md)
* The version number and what moves with it: [versioning](versioning.md)
* Steps 7 to 10, the pack command and a test install: [packing and testing](packing.md)

The package is `MyCompany.XAE.Docs.MyMachine`. It copies a folder of machine documentation onto engineering PCs. It is small on purpose, so the structure shows. The [examples](../examples/plc-library.md) use the same structure for a PLC library, a Windows service, a certificate and a route file.

## Step 1: the folder

One folder per package, named after the package id:

```
MyCompany.XAE.Docs.MyMachine\
  MyCompany.XAE.Docs.MyMachine.nuspec           the metadata
  MyCompany.XAE.Docs.MyMachine.ReleaseNote.md   the GUI's Feature Release Notes tab
  docs\                                          the files the package installs
    MyMachine Manual.pdf
    MyMachine Wiring.pdf
  tools\
    chocolateyinstall.ps1                        runs on install and on upgrade
    chocolateyuninstall.ps1                      runs on uninstall
    chocolateybeforemodify.ps1                   runs before an upgrade or uninstall touches files
```

The files the package installs are the *payload*. Here that is `docs\`. The payload can sit anywhere in the source folder. What matters is where the nuspec puts it inside the package, because the install script finds it relative to itself. This package puts it in `tools\docs`, next to the scripts.

## Step 2: the nuspec

The nuspec is an XML file. Every field the GUI shows comes from here. Start from this one and change the values:

```xml
<?xml version="1.0" encoding="utf-8"?>
<package xmlns="http://schemas.microsoft.com/packaging/2013/05/nuspec.xsd">
  <metadata>
    <id>MyCompany.XAE.Docs.MyMachine</id>
    <version>1.0.0</version>
    <title>MyMachine Documentation</title>
    <authors>MyCompany</authors>
    <requireLicenseAcceptance>false</requireLicenseAcceptance>
    <readme>MyCompany.XAE.Docs.MyMachine.ReleaseNote.md</readme>
    <summary>Manuals and drawings for MyMachine</summary>
    <description>Installs the MyMachine manuals and drawings under C:\Program Files\MyCompany\MyMachine\Documentation.</description>
    <tags>Beckhoff TwinCAT XAE</tags>
    <dependencies>
      <dependency id="TwinCAT.XAE.Base" version="0.0.0" />
    </dependencies>
  </metadata>
  <files>
    <file src="tools\**" target="tools" />
    <file src="docs\**" target="tools\docs" />
    <file src="MyCompany.XAE.Docs.MyMachine.ReleaseNote.md" target="MyCompany.XAE.Docs.MyMachine.ReleaseNote.md" />
  </files>
</package>
```

`tcpkg pack` refuses a nuspec without `id`, `version`, `authors` or `description`. It accepts one without `title` or `summary`, but the GUI and `tcpkg list` show those, so always fill them in. Verified with `tcpkg` 2.6.24.

Field by field:

`id`
:   The name the package is installed and upgraded by. Pick it once and never change it, because an upgrade only works when the id matches. No spaces: `tcpkg pack` refuses an id with a space in it. The convention the community packages follow is `<Company>.<XAE|XAR>.<Kind>.<Product>`, for example `MyCompany.XAE.Docs.MyMachine` or `MyCompany.XAE.PLC.Lib.MyLibrary`. Any scheme works if you keep to it on one feed. Ids that start with `TcPkg` are hidden by the GUI.

`version`
:   Three numbers, `1.0.0`. Every change to a package needs a new version. The [versioning page](versioning.md) has the rules.

`title`
:   What the GUI prints on the card and in the package list. Plain words, spaces allowed.

`authors`
:   Your company name.

`description`
:   The GUI prints this raw, line breaks and all, on the card and on the Description tab. Write one or two plain sentences. No Markdown, no bullet points.

`summary`
:   Shown in `tcpkg list` output. Keep it to one line.

`readme`
:   The file the GUI shows on the Feature Release Notes tab. Beckhoff names it `<Id>.ReleaseNote.md`. The file must also be listed under `<files>`; when it is not, `tcpkg pack` stops with "The readme file ... does not exist in the package". [Step 3](#step-3-the-release-notes-file) says what the file may contain.

`tags`
:   Space separated words. The Package Manager reads some of them:

    | Tag | Meaning |
    |---|---|
    | `Beckhoff TwinCAT` | by convention every TwinCAT package starts with these two |
    | `XAE` / `XAR` | which side the package is for: engineering or runtime |
    | `Workload`, `Category<Name>`, `VariantXAE` / `VariantXAR` | make a package a workload card, see [workloads](workloads.md) |
    | `AllowMultipleVersions` | more than one version of this package may be installed side by side (PLC libraries use it) |
    | `SystemServiceRestart` | the install restarts the TwinCAT System Service, so the GUI can warn the user |
    | `InstallationLevel1` | the package installs without administrator rights (Beckhoff uses it on PLC libraries and HMI packages) |

`dependencies`
:   What has to be installed first. The Package Manager installs a missing dependency before your package. A dependency on a Beckhoff package takes version `0.0.0`, which means any version, so your package does not refuse a newer TwinCAT. `TwinCAT.XAE.Base` is the engineering base package and `TwinCAT.XAR.Base` the runtime one; depend on the one for the side your package is for. The [versioning page](versioning.md#dependency-versions) covers dependencies on your own packages.

`requireLicenseAcceptance` and `license`
:   When `requireLicenseAcceptance` is `true`, the GUI shows the license file named by `<license type="file">LICENSE.txt</license>` and asks the user to accept it. List the license file under `<files>` like any other file. Leave it `false` when there is no license to accept.

`projectUrl`, `copyright`
:   Optional. The example has neither.

`files`
:   What goes into the package, and where. `src` is relative to the nuspec, `target` is the path inside the package, and `**` takes a whole folder tree. A `src` that matches nothing is skipped without a word, and the package fails on the target PC instead of on your desk. [Step 8](packing.md#step-8-look-inside) is how you catch that.

A workload adds `<packageTypes>` and the icon fields; those are on the [workload page](workloads.md).

## Step 3: the release notes file

The GUI shows the file named by `<readme>` on the Feature Release Notes tab. Its Markdown parser is very small. It understands `#`, `##` and `###` headings and plain lines of text. Bullet lists come out as `[object Object]`. Bold and backticks print their markers, and links print their brackets. So the file uses headings and sentences only:

```markdown
## What this package installs
The MyMachine manuals and drawings, in C:\Program Files\MyCompany\MyMachine\Documentation.

## Release history
### 1.1.0
Adds the spare parts list.
### 1.0.0
First release.
```

Add a line under *Release history* with every new version. The user reads this tab to decide whether to upgrade.

On Beckhoff's feed and on a local folder feed the tab works as it is. A feed served by a NuGet server of your own, such as Nexus, hands the GUI the nuspec metadata but no file from inside the package, so there the release notes need a copy hosted next to the feed. Verified with `tcpkg` 2.4 against Nexus, and still so with 2.6.

## What the GUI shows, and where it comes from

| In the GUI | From |
|---|---|
| Card title | `<title>` |
| Text on the card and the Description tab | `<description>`, printed raw |
| Feature Release Notes tab | the file named by `<readme>` |
| Dependencies tab | `<dependencies>`, with their version ranges |
| Icon on the card | the stock icon for a package; a custom one only for a [workload](workloads.md) |
| Version list | every version of that id on the feed |

Verified against the Package Manager GUI 2.2 and 2.6 and Beckhoff's own packages. When in doubt, look at what Beckhoff's packages do and copy that.

Next: [step 4, the install script](scripts.md).
