# Packing and testing

The folder is ready: a nuspec, the payload, three scripts. Packing turns it into one `.nupkg` file. Testing installs that file on your own PC from a folder, before anyone else sees it.

## What you need

* **The TwinCAT Package Manager** on the PC you pack on. It puts `tcpkg` on the PATH, and `tcpkg pack` is the packer. On a PC without it, the `nuget` command line tool packs the same nuspec: `nuget pack <nuspec> -OutputDirectory <folder> -NoPackageAnalysis`.
* **The Package Manager GUI closed.** `tcpkg` is single instance, and the GUI runs `tcpkg` for every click. A pack or install that fails with "TcPkg is already running" usually means the GUI, or a `TcPkg.exe` it left behind; see [when something goes wrong](scripts.md#when-something-goes-wrong).
* **TwinCAT XAE Shell and Visual Studio closed** for the test install. `tcpkg` 2.6.24 refused to install even this guide's test packages, which have nothing to do with the engineering, while TwinCAT XAE Shell was open: "Error: Running Microsoft Visual Studio or Beckhoff TwinCAT XAE Shell (64) process found.", exit code 1051. The dry run prints its plan and then the same error. A package whose files the engineering never has open can carry the tag `NoVisualStudioRestart` and install with the Shell open; see [the tags](nuspec.md#step-2-the-nuspec).
* **An administrator PowerShell** for the test install. Packing does not need one.

## Step 7: pack

```powershell
New-Item -ItemType Directory -Force C:\feed | Out-Null
tcpkg pack .\MyCompany.XAE.Docs.MyMachine\MyCompany.XAE.Docs.MyMachine.nuspec -o C:\feed
```

That writes `C:\feed\MyCompany.XAE.Docs.MyMachine.1.0.0.nupkg`. The id and the version in the file name come from the nuspec. Two habits of `tcpkg pack` (2.6.24):

* The output folder must exist. `tcpkg pack` does not create it and stops with "The specified output directory ... does not exist".
* Without `-o` the package lands next to the nuspec, in among your sources. Always give `-o`.

`-v <version>` sets the version from the command line instead of the nuspec; see [versioning](versioning.md#where-the-version-lives).

When you build several packages that depend on each other, pack the dependencies first, so that when you test the one on top its parts are already in the folder.

## Step 8: look inside

A `.nupkg` is a zip. Before you install it, list what is in it:

```powershell
Add-Type -AssemblyName System.IO.Compression.FileSystem
$zip = [System.IO.Compression.ZipFile]::OpenRead('C:\feed\MyCompany.XAE.Docs.MyMachine.1.0.0.nupkg')
$zip.Entries | Select-Object FullName, Length
$zip.Dispose()
```

For the example package, built with two placeholder PDFs:

```
FullName                                                                          Length
--------                                                                          ------
_rels/.rels                                                                          522
MyCompany.XAE.Docs.MyMachine.nuspec                                                  773
MyCompany.XAE.Docs.MyMachine.ReleaseNote.md                                          166
tools/chocolateybeforemodify.ps1                                                     127
tools/chocolateyinstall.ps1                                                          708
tools/chocolateyuninstall.ps1                                                        580
tools/docs/MyMachine Manual.pdf                                                       42
tools/docs/MyMachine Wiring.pdf                                                       32
[Content_Types].xml                                                                  590
package/services/metadata/core-properties/1ae24b930f0a464eb93b380cd367a32a.psmdcp    735
```

`_rels`, `[Content_Types].xml` and `package\` are NuGet's own bookkeeping; the `.psmdcp` name changes with every pack. The rest is what you want to see: the nuspec, the release notes file at the path `<readme>` names, the three scripts, and the payload where the install script expects it.

What you do not want to see is a payload that should be there and is not. A `<file src>` that matches nothing is skipped without a word: `tcpkg pack` builds the package anyway, and the package fails on the target instead of on your desk. Verified with 2.6.24, with a `<file src>` pointing at a file that did not exist. This step is the only one that catches it.

## Step 9: a local folder feed

A folder of `.nupkg` files is a feed. Register `C:\feed` once, from an administrator PowerShell:

```powershell
tcpkg config unset -n VerifySignatures
tcpkg source add -n Local -s C:\feed --priority 2
tcpkg source list
```

The first line turns signature checking off. Beckhoff signs its own packages, so that their integrity can be checked wherever a copy is kept ([TwinCAT Package Manager on Infosys](https://infosys.beckhoff.com/content/1033/tc3_installation/15698617995.html)). A package you pack yourself carries no such signature, and with checking on, the install of an unsigned package fails at the download. The feed can be added either way. The setting stays off on that PC until someone turns it on again; for the feed you hand to others, [signing your packages](../hosting/signing.md) is the way to leave it on. Beckhoff's own feed is priority 1; the number only orders the list.

Keep the folder on a local drive. An elevated `tcpkg` does not always see a mapped network drive, and on one PC, starting an elevated shell with a network drive as its working directory made the elevation itself fail. If your packages are built on a share, copy them to `C:\feed` first, and start elevated shells with `-WorkingDirectory C:\`.

A folder feed is the right tool for testing, and the [hosting section](../hosting/index.md) covers it and the alternatives for sharing packages with other PCs. Beckhoff notes on the same Infosys page that it is slower than a NuGet server: the Package Manager has to read every package in the folder to learn its name and version, where a server hands that metadata over separately.

## Step 10: install, check, upgrade, uninstall

From an administrator PowerShell:

```powershell
tcpkg list -n Local --no-cache
tcpkg install MyCompany.XAE.Docs.MyMachine -n Local --dry-run -y
tcpkg install MyCompany.XAE.Docs.MyMachine -n Local -y
```

`list` shows that the feed has the package. The dry run resolves the dependencies and prints what would be installed, without touching the PC. The real install runs your install script, and its output includes every line the script printed. The full log is `C:\ProgramData\Beckhoff\TcPkg\logs\TcPkg.log`.

Then check the result, by hand:

* **On disk.** The files are where the install script put them: `C:\Program Files\MyCompany\MyMachine\Documentation`.
* **In the GUI.** Open the Package Manager, pick the feed *Local* in the feed selector, and find the package: title, description, release notes, dependencies. The GUI is what your users see, and it is pickier about the metadata than the command line.

Then the upgrade. Raise the version in the nuspec to `1.1.0`, add it to the release notes, pack again into `C:\feed`, and:

```powershell
tcpkg upgrade MyCompany.XAE.Docs.MyMachine -n Local -y
```

And the uninstall:

```powershell
tcpkg uninstall MyCompany.XAE.Docs.MyMachine -y
```

After the uninstall, the `Documentation` folder is gone, and so are the `MyMachine` and `MyCompany` folders, because nothing else was in them. This package went through exactly these steps, install, upgrade from 1.0.0 to 1.1.0 and uninstall, with `tcpkg` 2.6.24.

## Testing a change

Every test of a change is a new version: raise the version, pack, install or upgrade. The Package Manager treats a version as fixed and will not pick up a changed file under the same version; see [versioning](versioning.md#a-version-is-never-reused).

The Package Manager also caches feed metadata, for about 30 minutes, in `%LOCALAPPDATA%\NuGet\v3-cache`. A package you just packed may not be listed yet. `--no-cache` on `list` and `install` skips the cache. Deleting the feed's folder under `v3-cache` resets it for the GUI too.

## Before you share a package

A checklist of the mistakes that have already happened once:

* The version has three numbers and is higher than every version you shared before.
* The release notes file has a heading for the new version, and uses headings and plain lines only.
* The description is one or two plain sentences.
* The id does not start with `TcPkg`.
* A dependency on a Beckhoff package uses `0.0.0`.
* Every `<file src>` matched something: step 8 shows the file in the package.
* The install script ran, the upgrade from the previous version ran, and the uninstall left nothing behind.
* The scripts use nothing that needs PowerShell 7, if the package may meet an older Package Manager.
* The package is [signed](../hosting/signing.md), if your feed is used with signature verification on.
* For a workload: see the [workload page](workloads.md).

## Secrets in a package

Some packages exist to install something secret: a certificate with its password, a route file with a broker login. Those files must be in the package, and they should not be in your source control. One way to arrange it:

* Keep the secret files in a folder that source control ignores, for example `secrets\<Id>\` next to your package folders.
* Let the nuspec reach over with a relative `src`: `<file src="..\secrets\MyCompany.XAE.Cert.Signing\*.tccert" target="tools" />`. The packer resolves it relative to the nuspec, and the file lands in `tools\` inside the package.
* Because a `src` that matches nothing is skipped without a word, a copy of your sources without the secrets folder packs a package without the secret. Step 8 catches that; so does an install script that checks for the file and throws with a message that says where it should have come from.

The packed package contains the secret in plain text, and so does every feed it is on. That is acceptable on a feed only people who may have the secret can read, such as one on a closed network, and not anywhere else.

Next: [upgrades and retiring an id](lifecycle.md), what the Package Manager offers once 1.0.0 is out, and then [workloads](workloads.md), which turn several packages into one card in the GUI.
