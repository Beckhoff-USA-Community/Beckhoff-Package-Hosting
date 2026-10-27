# A folder on disk

The simplest feed there is: a folder with `.nupkg` files in it. No server, no login, no upload. It is the feed the [packing and testing](../packages/packing.md) steps use, and it is enough for one PC, for a test, or for handing a set of packages over on a USB stick.

## Setting it up

Make the folder and put the packages in it. Pack straight into it, or copy the `.nupkg` files in:

```powershell
New-Item -ItemType Directory -Force C:\MyPackages | Out-Null
Copy-Item .\MyCompany.XAE.Docs.MyMachine.1.0.0.nupkg C:\MyPackages\
```

Then, on each PC that should use it, from an administrator PowerShell:

```powershell
tcpkg config unset -n VerifySignatures
tcpkg source add -n MyPackages -s C:\MyPackages --priority 2
tcpkg list -n MyPackages --no-cache
```

The first line is [signature verification](signatures.md), and it is the line for unsigned packages. For [signed packages](signing.md) it is `tcpkg certificate add C:\MyPackages\MyCompany-Signing.cer -n MyCompany` instead, with the certificate file kept in the folder next to the packages, and verification stays on. `-n` is the name the GUI shows in its feed selector, `-s` the folder. Beckhoff's own feed has priority 1; the number only orders the list.

## Publishing and changing

Publishing a new version is copying its `.nupkg` into the folder. A PC that has the older version sees the new one as an update. Leave the older versions in the folder, so that a PC can still install them or go back to one.

You can delete a file from the folder, and nothing stops you from putting a different file under the same version back. Do not, for a version anyone has installed: their PC will not see the difference. See [versioning](../packages/versioning.md#a-version-is-never-reused).

The Package Manager caches what it read from the feed. A package you just copied in may not show up for a while; `--no-cache` on `tcpkg list` and `tcpkg install`, or *Reload Information* in the GUI, reads the folder again.

## What works

Everything the GUI shows comes out of the package itself: the workload icon from `<icon>`, the release notes from `<readme>`, title, description and dependencies from the nuspec. Nothing has to be hosted anywhere else. Workloads with a shared title show as one card with an Engineering and a Runtime row.

The catch is speed. Beckhoff notes on [Infosys](https://infosys.beckhoff.com/content/1033/tc3_installation/15698617995.html) that a folder feed is slower than a NuGet server: the Package Manager has to read every package in the folder to learn its name and version, where a server hands that metadata over separately. The more packages and versions in the folder, the more there is to read.

## A PC without a feed: the offline bundle

A folder feed on a USB stick is one way to reach a PC that has no network. The Package Manager has a second one that needs no feed on the target at all: `tcpkg download` bundles a package into one file on a connected PC, and `tcpkg import` installs it from that file. Both were checked with `tcpkg` 2.6.24.

On the connected PC, from any shell, with a folder that already exists (`tcpkg download` does not create it, like `pack`):

```powershell
New-Item -ItemType Directory -Force C:\bundle | Out-Null
tcpkg download MyCompany.XAE.Docs.MyMachine=1.0.0 -n MyPackages --package-only -o C:\bundle --zip -y
```

"Creating download.config... done. Creating download.tcpkgzip... done. Archive is ready for import." The folder holds one file, `download.tcpkgzip`: a zip with the `.nupkg` and a `download.config` that lists the id and version. Without `--package-only` the dependencies come along, Beckhoff's included, which is the point when the target has no TwinCAT yet; that case was not tried for this guide. `-i packages.config` takes a list of packages instead of naming them.

On the target PC, from an administrator PowerShell:

```powershell
tcpkg import -i C:\bundle\download.tcpkgzip -y
```

The package installs as if it came from a feed, scripts included, and the target needs no feed registered: tested with the feed removed. Importing again says "This or a newer version is already installed"; `--dry-run` prints what would be installed. `tcpkg import --help` lists the same `--params` options as `install`.

`tcpkg download` has a third form, `--v3`, which writes the packages unzipped into the NuGet v3 folder layout (`<lower id>\<version>\` with the `.nupkg`, its hash, the nuspec and the unpacked `tools\` folder). That is the layout a NuGet server serves, not the layout a folder feed reads; it was looked at and not used for this guide.

## A shared folder

`tcpkg source add` also takes a file share as the source, according to its help. That turns the folder into a feed for every PC on the network that can read the share. It was not tested for this guide. What was seen is that an elevated `tcpkg` does not always find a mapped network drive, and on one PC starting an elevated shell with a network drive as its working directory made the elevation itself fail. A drive letter is mapped per user session, so if you try a share, register it by its full path, `\\server\share\MyPackages`, not by a drive letter.

Next: [GitHub Packages](github.md), or [Nexus](nexus.md) for a server on your own network.
