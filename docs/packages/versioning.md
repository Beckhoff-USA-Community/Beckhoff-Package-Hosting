# Versioning

The version is how the Package Manager tells two builds of a package apart. It decides what a PC sees as an update, which build an install picks, and which versions of a dependency are good enough. Get it right once and the *Updates* tab does the rest.

## The version number

Three numbers: `1.0.0`. The nuspec carries it:

```xml
<version>1.0.0</version>
```

Stay with three. A four part version goes wrong on the way through NuGet. `tcpkg pack` (2.6.24) writes `<version>1.2.0.0</version>` as a file called `MyCompany.XAE.Docs.MyMachine.1.2.0.0.nupkg`, while the nuspec inside it says `1.2.0`. From then on file names, folder names and feed listings disagree.

Which number to raise is a convention, and the usual one is [semantic versioning](https://semver.org/):

| Raise | When | Example |
|---|---|---|
| the third, `1.0.0` → `1.0.1` | a fix; nothing the user relies on changes | a corrected page in the manual |
| the second, `1.0.1` → `1.1.0` | something new; what was there still works | an added spare parts list |
| the first, `1.1.0` → `2.0.0` | a change that can break what the user has | a PLC library whose function blocks changed their interface |

## A version is never reused

A feed treats a published version as fixed. Every change, however small, gets a new version: a new file, a fixed script, a corrected description.

This is not only a feed rule. The Package Manager will not pick up a changed package under a version it already knows, and a PC that installed `1.0.0` sees nothing new in a second, different `1.0.0`. On a local folder feed you can delete a file and re-pack the same version, and on a NuGet server you run yourself you can usually delete a version and push it again. Do that only for a mistake nobody has installed yet. Once you do it from habit, you will one day do it to a version that is out there.

## Where the version lives

The nuspec `<version>` is the one that counts, but it is rarely the only place:

* **The release notes file.** Add a heading for the new version under *Release history*, with a sentence on what changed. See [step 3](nuspec.md#step-3-the-release-notes-file).
* **The scripts, when they name the version.** A PLC library package removes the library by its exact version, so the library version is in the uninstall script and in the install script's check. The [PLC library example](../examples/plc-library.md#uninstalling) moves them together with the nuspec.
* **A workload that depends on the package.** See [below](#workloads-move-with-their-parts).

Before you pack, search the package folder for the old version number.

`tcpkg pack` can also set the version from the command line, which leaves the nuspec alone. The version inside the package and the file name follow the command line:

```powershell
tcpkg pack .\MyCompany.XAE.Docs.MyMachine\MyCompany.XAE.Docs.MyMachine.nuspec -o C:\feed -v 1.1.0
```

That is handy when something else decides the version, a build server for example. When you build by hand, edit the nuspec, so the source says what was built.

## Prereleases

A version with a suffix is a prerelease: `1.1.0-beta.1`. `tcpkg pack` accepts it and writes `MyCompany.XAE.Docs.MyMachine.1.1.0-beta.1.nupkg`. `tcpkg list`, `install` and `upgrade` leave prereleases out unless you add `--pre`: with 1.1.0 and 1.2.0-beta.1 of a test package on the feed, `tcpkg upgrade` picked 1.1.0. Every feed also has an *Include Prereleases* setting, which `tcpkg source list` shows.

## Dependency versions

The `version` attribute of a `<dependency>` is a NuGet version range, not a single version. [NuGet's notation](https://learn.microsoft.com/en-us/nuget/concepts/package-versioning#version-ranges):

| Written | Means |
|---|---|
| `version="0.0.0"` | any version |
| `version="1.2.0"` | 1.2.0 or newer |
| `version="[1.2.0]"` | exactly 1.2.0 |
| `version="[1.2.0,2.0.0)"` | 1.2.0 or newer, but below 2.0.0 |

**A dependency on a Beckhoff package takes `0.0.0`.** Your package needs TwinCAT, not one particular build of it. A pinned TwinCAT version would make your package refuse the next TwinCAT update for no reason of yours.

**A dependency on one of your own packages names the version it was built and tested with**, in the plain form: `version="1.2.0"`. That works as a pin because `tcpkg` installs the *lowest* version the range allows, not the newest. With a test package in versions 1.0.0 and 1.1.0 on the feed:

| The dependency says | `tcpkg` installed |
|---|---|
| `version="1.0.0"` | 1.0.0, although 1.1.0 was there |
| `version="[1.0.0]"` | 1.0.0 |
| `version="1.1.0"` | 1.1.0 |

Verified with `tcpkg install --dry-run` on 2.6.24. So a dependency installs exactly the version it names as long as that version is on the feed, and moving it to a new version is what brings the new version along. The workloads quoted in this guide all use the plain form.

## Workloads move with their parts

A [workload](workloads.md) depends on your component packages at the versions it was tested with. Upgrading the workload upgrades the components to the versions the new workload names.

So when a component changes, the workload changes too. The component gets a new version, the workload's dependency moves to it, and the workload gets a new version of its own. Publish both, the component first, so the workload is never on the feed before its parts.

## More than one version at once

Normally one version of a package is installed, and an upgrade replaces it. PLC libraries are the exception. A project that references library 2.2.1 has to keep compiling after 2.3.0 arrives, so both have to be there. The tag `AllowMultipleVersions` in the nuspec tells the Package Manager that versions of this package may be installed side by side. Beckhoff's own library packages carry it; see the [PLC library example](../examples/plc-library.md#the-nuspec). Two narrower forms exist on Beckhoff's feed, `AllowMultipleMajorVersions` (one installed version per major version, on the HMI Server API packages) and `AllowMultipleMajorMinorVersions` (one per major.minor, on `Beckhoff.Python`); neither was tried for this guide.

## A major version, and a new id

A new major version can warn the user before the upgrade runs, with a text the user has to accept, and a package id that must change can hand its users over to the new id. Both are tags, both were measured, and both are on [upgrades and retiring an id](lifecycle.md).

## Upgrades on the user's PC

Once a newer version is on the feed, a PC that has the package sees it:

* in the GUI on the *Updates* tab;
* with `tcpkg list -o`, which lists installed packages that have a newer version;
* and installs it with `tcpkg upgrade <Id>`.

`tcpkg install` of a version that is already installed does nothing; it says so and stops. To go back to an older version, name it and allow the step down: `tcpkg install <Id>=1.0.0 --allow-downgrade`. The Package Manager treats that as an upgrade, so the same scripts run; see [which script runs when](scripts.md#how-the-scripts-run).

The Package Manager caches feed metadata for about 30 minutes, so a version you just published can take that long to show up. `--no-cache` on `tcpkg list` or `tcpkg install`, or *Reload Information* in the GUI, skips the wait.

Next: [packing and testing](packing.md).
