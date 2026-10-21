# What makes up a workload

Open the TwinCAT Package Manager and what you see are cards: TwinCAT Standard, TwinCAT HMI, the vendor categories with their products. Each card is a workload. A workload is a package like any other, with one difference in its metadata and one in its purpose: it is marked as a workload, and it installs nothing itself. Its only job is to depend on the packages that do the work, so that one click installs all of them in the right order.

Beckhoff's own feed shows the pattern. TwinCAT Standard is two workloads, `TwinCAT.Standard.XAE` and `TwinCAT.Standard.XAR`, drawn as one card with an Engineering row and a Runtime row. The engineering one depends on the engineering packages, the runtime one on the runtime packages.

## The folder

Same as a package, plus an icon:

```
BAUS.AdsOverMqtt.Engineering.XAE\
  BAUS.AdsOverMqtt.Engineering.XAE.nuspec
  BAUS.AdsOverMqtt.Engineering.XAE.ReleaseNote.md
  icon.png                                           64 by 64 pixels
  tools\
    chocolateyinstall.ps1                            one Write-Host line
    chocolateyuninstall.ps1                          one Write-Host line
    chocolateybeforemodify.ps1                       a comment
```

The scripts exist because the Package Manager expects them. The install script of a workload in this repository is a single line:

```powershell
# The workload itself installs nothing. The packages listed as dependencies in the nuspec
# carry the content; the package manager installs them together with this workload.
Write-Host 'Advanced Applications | Developer workload installed.'
```

## The nuspec

This is the Engineering workload of the ADS-over-MQTT card:

```xml
<?xml version="1.0" encoding="utf-8"?>
<package xmlns="http://schemas.microsoft.com/packaging/2013/05/nuspec.xsd">
  <metadata>
    <id>BAUS.AdsOverMqtt.Engineering.XAE</id>
    <version>1.1.0</version>
    <title>ADS-over-MQTT Routes</title>
    <authors>Beckhoff Automation LLC</authors>
    <projectUrl>https://github.com/0w8States/Iris-Manor-Custom-Nexus-Packages</projectUrl>
    <copyright>(c) Beckhoff Automation LLC</copyright>
    <requireLicenseAcceptance>false</requireLicenseAcceptance>
    <icon>icon.png</icon>
    <iconUrl>https://nexus.example.com/repository/tcpkg-raw/icons/baus.adsovermqtt.engineering.xae.png</iconUrl>
    <readme>BAUS.AdsOverMqtt.Engineering.XAE.ReleaseNote.md</readme>
    <summary>ADS-over-MQTT connection to the MQTT broker</summary>
    <description>ADS-over-MQTT connection to the MQTT broker</description>
    <tags>Beckhoff TwinCAT Workload CategoryAdvanced&#160;Applications&#160;Group VariantXAE ADS MQTT</tags>
    <packageTypes>
      <packageType name="Workload" />
    </packageTypes>
    <dependencies>
      <dependency id="TwinCAT.XAE.Base" version="0.0.0" />
      <dependency id="BAUS.AdsOverMqtt.XAE" version="1.2.0" />
    </dependencies>
  </metadata>
  <files>
    <file src="tools\**" target="tools" />
    <file src="BAUS.AdsOverMqtt.Engineering.XAE.ReleaseNote.md" target="BAUS.AdsOverMqtt.Engineering.XAE.ReleaseNote.md" />
    <file src="icon.png" target="icon.png" />
  </files>
</package>
```

What is different from a package:

`<packageType name="Workload" />`
:   This is what makes it a workload. Without it the GUI lists the package under the plain packages, with it the GUI draws a card.

The three tags
:   `Workload` again, `Category<Name>` for the group the card sits in, and `VariantXAE` or `VariantXAR` for the row on the card. All three are required; the GUI does not draw the card without them. A multi word category keeps its spaces as `&#160;` (a non breaking space) so that it stays one tag: `CategoryAdvanced&#160;Applications&#160;Group` shows up as the category *Advanced Applications Group*. That is how the community workloads do it, and it works.

`<icon>` and `<iconUrl>`
:   A workload is the only kind of package the GUI draws a custom icon for. `icon.png` is 64 by 64 pixels, and `<icon>` names it inside the package. On a feed hosted on a NuGet server the GUI never reads a file out of the package, so `<iconUrl>` points at a copy of the same image hosted next to the feed; the [publishing page](../nexus/publishing.md) covers that copy. On a local folder feed `<icon>` alone is enough. A component package carries neither; it always gets the stock icon, and the build lint in this repository refuses an `<iconUrl>` on a package for that reason.

`<title>` shared between two workloads
:   An XAE and an XAR workload with the same title are drawn as one card with an Engineering and a Runtime row. That is the TwinCAT Standard pattern, and it is how the two ADS-over-MQTT workloads become one card.

`<description>`
:   The card prints it in full, so it is one short sentence. The release notes file is the place for the long version.

`<dependencies>`
:   Your component packages at exact versions, and the Beckhoff base package the side needs at `0.0.0`. Pinning your own packages is what makes an upgrade work: upgrade the workload and the Package Manager upgrades its components to the versions the new workload names. Leaving them open would let the workload install while its parts stay old.

## Naming

Workload ids in this repository are `BAUS.<Product>.<Engineering|Runtime>.<XAE|XAR>` when a product has both rows and `BAUS.<Product>.<XAE|XAR>` when it has one. The community repositories use `<Vendor>.<Product>.<XAE|XAR>`. Either way the id ends in the side, and the title is the product name the user knows.

## Upgrading and uninstalling

Upgrading a workload upgrades its pinned components:

```powershell
tcpkg upgrade BAUS.AdsOverMqtt.Engineering.XAE -n MyFeed
```

Uninstalling a workload removes only the workload itself. The Package Manager leaves the dependencies to its own rules, which in practice means they stay. To take the components along, name them:

```powershell
tcpkg uninstall BAUS.AdsOverMqtt.Engineering.XAE BAUS.AdsOverMqtt.XAE
```

Do not reach for `--include-dependencies` here. The workload also depends on `TwinCAT.XAE.Base`, and that flag takes the base package with it.

## What a nuspec cannot say

A nuspec can say what a package needs. It cannot say what a package must not be installed next to. The two ADS-over-MQTT route packages carry different broker credentials and must never be on the same PC, and there is no field for that. The install script has to check: it looks for the other package's folder under `C:\ProgramData\Beckhoff\TcPkg\lib\` and for a route file carrying the other package's marker, and throws before it copies anything. The [plain files example](../examples/files.md) shows the check. When you need a rule the metadata cannot express, the install script is the only place left.
