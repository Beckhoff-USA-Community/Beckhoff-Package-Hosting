# Upgrades and retiring an id

Once 1.0.0 is on a feed, two things can happen to a package that the [versioning page](versioning.md) does not cover: a new major version that the user should read about before it lands, and an id that has to go. The Package Manager has a mechanism for each, both driven by tags, and Beckhoff's own feed uses both. This page shows them as they were measured with `tcpkg` 2.6.24 and GUI 2.7.6 on test packages.

## A disclaimer before a major upgrade

A PLC library whose function blocks changed, a service whose configuration file has a new layout: the user should know before the upgrade runs, not from the release notes afterwards. The package can carry a text that the Package Manager shows and the user has to accept before an upgrade across a major version. Beckhoff's `TwinCAT.XAE.AdvancedMotion` does it for its 3.3 to 3.4 step.

Three things in the nuspec of the new major version, here 2.0.0:

```xml
<readme>tools\UpdateInfo.md</readme>
...
<tags>Beckhoff TwinCAT XAE PackageDisclaimerMajorUpgrade AS:AcceptMyMachine2.0:AS</tags>
```

* **`PackageDisclaimerMajorUpgrade`** switches the check on.
* **`AS:AcceptMyMachine2.0:AS`** is the accept string, between `AS:` and `:AS`, the same form as on the [feed disclaimer](disclaimer.md). Put the version in it: a new text is a new acceptance.
* **`<readme>`** names the text. It is the same file the GUI shows on the Feature Release Notes tab, so for this version the file carries the warning first and the release history after it. Beckhoff keeps it in `tools\UpdateInfo.md`; the running example keeps `<Id>.ReleaseNote.md`, and either path works as long as the file is in the package. Headings and plain lines only, as on the [release notes](nuspec.md#step-3-the-release-notes-file); the disclaimer dialog showed the test file's lines as written.

What the user sees when 1.0.0 is installed and 2.0.0 is on the feed:

* **In the GUI**, the Updates tab offers 2.0.0 as usual. The upgrade opens a dialog titled *Package Disclaimer* with the package id, the version, the text of the readme file, and *Accept* and *Decline* buttons. Decline leaves 1.0.0 in place.
* **In the terminal**, `tcpkg upgrade` asks its usual "Continue? (Y/N)" and then prints "To install MyCompany.XAE.Docs.MyMachine 2.0.0 you have to accept the following disclaimer:", the text, and "Press Y to accept or N to decline the disclaimer."

When it asks, and when it does not, with the tag on every 2.x version:

| The user | Asked? |
|---|---|
| upgrades 1.0.0 to 2.0.0 | yes |
| installs 2.0.0 on a PC without the package | no |
| upgrades 2.0.0 to 2.1.0 | no |
| goes back from 2.0.0 to 1.0.0 with `--allow-downgrade` | no |

So the tag does what its name says: it asks once, at the step across the major version. Beckhoff's packages also carry a `PackageDisclaimerMinorUpgrade` tag; it was not tried for this guide.

**In a script**, the prompt cannot be answered. `tcpkg upgrade -y` stops with "You need to accept the disclaimer to install MyCompany.XAE.Docs.MyMachine 2.0.0.", exit code 1223, because `-y` answers the confirmation and not the disclaimer. The accept string does:

```powershell
tcpkg upgrade MyCompany.XAE.Docs.MyMachine -n MyFeed -y --accept-string AcceptMyMachine2.0
```

A wrong string stops with "The provided accept string does not match the required accept string for MyCompany.XAE.Docs.MyMachine 2.0.0.", exit code 4201, and nothing changes. `tcpkg upgrade --help` does not list the option in 2.6.24, like `source add --help` does not; both take it.

**On an older Package Manager** the tag means nothing and the upgrade would run without the warning. Beckhoff's packages guard against that in the install script, with the environment variable the Package Manager sets when it knows the feature:

```powershell
if (-not $env:TcPkgCliFeaturePackageDisclaimer) {
    Set-PowerShellExitCode 15603
    throw "A newer version of the TwinCAT Package Manager is required to install $env:ChocolateyPackageName $env:chocolateyPackageVersion."
}
```

See [the helper functions](scripts.md#the-helper-functions). Whether you want that guard depends on whether the warning is advice or a condition.

## Retiring an id

The [nuspec page](nuspec.md#step-2-the-nuspec) says to pick the id once and never change it, because an upgrade only works when the id matches. When it has to change anyway, because a company or a product was renamed or a package was split, the Package Manager can move a PC from the old id to the new one. Beckhoff did it when `TF5500.MC3.PTP.XAE` became `TF5500.MC3.Base.XAE`.

Two packages make it work:

**The new id carries the old id in its tags**, between `DP:` and `:DP`:

```xml
<id>MyCompany.XAE.Docs.MyMachine2</id>
<version>1.0.0</version>
...
<tags>Beckhoff TwinCAT XAE DP:MyCompany.XAE.Docs.MyMachine:DP</tags>
```

**The old id gets one last version** that installs nothing and depends on the new id. Its version is higher than any version of the old id on the feed, and its dependency names the version of the new id it should bring:

```xml
<id>MyCompany.XAE.Docs.MyMachine</id>
<version>3.0.0</version>
<description>Replaced by MyCompany.XAE.Docs.MyMachine2. Upgrading this package installs the new one.</description>
...
<dependencies>
  <dependency id="MyCompany.XAE.Docs.MyMachine2" version="1.0.0" />
</dependencies>
```

Both go on the feed. What happens on a PC that has the old id installed, measured on test packages:

* `tcpkg list -o` and the GUI's Updates tab show the old id with an update to 3.0.0, an ordinary update as far as the user can see.
* `tcpkg upgrade MyCompany.XAE.Docs.MyMachine` prints "MyCompany.XAE.Docs.MyMachine is deprecated and will be replaced by MyCompany.XAE.Docs.MyMachine2.", then "TcPkg will uninstall the following packages: MyCompany.XAE.Docs.MyMachine 2.1.0 (Deprecated)" and "TcPkg will install the following packages: MyCompany.XAE.Docs.MyMachine2 1.0.0". It runs the old package's before-modify and uninstall scripts, then the new package's install script. **The last version of the old id is never installed**; afterwards `tcpkg list -i` shows the new id only.
* `tcpkg install MyCompany.XAE.Docs.MyMachine2` on such a PC does the same without the detour: it uninstalls the old id as "(Deprecated)" and installs the new one.
* Without the last version of the old id, `tcpkg list -o` and `tcpkg upgrade` do not notice the new id at all. The tag alone only acts when the new id is installed by name.

Three things to know before you do it:

* **The scripts do not learn about the switch.** The old uninstall script runs as on any uninstall and removes the old package's files, and the new install script runs as on a first install, without `previousInstalledPackageVersion`. Anything the old package made that the new one should keep, such as an edited configuration file, has to survive the old uninstall script on its own.
* **Workloads that depend on the old id move to the new one.** A workload that still names the old id would pull its last version, and with it the new id, which works but reads oddly on the Dependencies tab.
* **On an older Package Manager** the tag means nothing; the last version of the old id installs as an empty package next to the new one, which is harmless. The Package Manager sets `TcPkgCliFeaturePackageDeprecation` when it knows the feature, for a guard like the one above.

Next: [workloads](workloads.md), which turn several packages into one card in the GUI.
