# Pitfalls

Everything on this page happened. Most entries are one line; the page they belong to has the long version.

## The Package Manager

* **`tcpkg` is single instance.** A second `TcPkg.exe` fails with "TcPkg is already running". Close the GUI before building or publishing, and check `Get-Process TcPkg` when something fails anyway: the GUI keeps a child process that holds the lock and sometimes leaves it behind on close.
* **Inside a package script you are already inside `tcpkg`.** Do not call `tcpkg` from an install script for anything but a read with `--ignore-mutex`. Read the Package Manager's settings from `%ProgramData%\Beckhoff\TcPkg\appsettings.json` instead of `tcpkg config list`, and check what is installed by looking at the folders under `%ProgramData%\Beckhoff\TcPkg\lib` instead of `tcpkg list`.
* **Scripts run under Windows PowerShell 5.1.** PowerShell 7 syntax in a package script fails on the target with a parse error that names a line you did not think twice about.
* **`tcpkg install --dry-run` needs an elevated shell** although it changes nothing.
* **Turn signature verification off before adding an unsigned feed**, with `tcpkg config unset -n VerifySignatures`. The order matters: with verification on, adding the feed fails, and so does every install from it.
* **GitHub package feeds need `--take 100`** on `tcpkg source add`, or the add fails with "Failed to retrieve metadata from source". GitHub refuses pages larger than 100 and the Package Manager asks for 500.
* **Feed metadata is cached for about 30 minutes.** A package you just pushed is not there yet. `--no-cache` on the command, *Reload Information* in the GUI, or delete the feed's folder under `%LOCALAPPDATA%\NuGet\v3-cache`.
* **Uninstalling a workload with `--include-dependencies` takes `TwinCAT.XAE.Base` with it.** Name the component packages instead.
* **Elevated tests and network drives do not mix.** Copy packages to a local folder before `tcpkg source add`, and start an elevated shell with `-WorkingDirectory C:\`.
* **`tcpkg push` writes its command line, API key included, to `C:\ProgramData\Beckhoff\TcPkg\logs\TcPkg.log`.** Push with `dotnet nuget push` instead.

## Metadata

* **A published version is immutable.** Every change, however small, is a new version. On your own Nexus you can delete and re-push the same version; do that for a fix you made five minutes ago and never for a version anyone has installed, because their PC will not see a difference.
* **Three part versions.** `1.2.0.0` becomes `1.2.0` on the way through NuGet and the folder names stop matching.
* **The description is printed raw.** One or two plain sentences. Line breaks, bullets and Markdown show up as typed.
* **The release notes parser knows headings and sentences.** Bullet lists render as `[object Object]`. Bold, backticks and links print their markers.
* **A `<file src>` that matches nothing is skipped without a word.** The package builds, ships, and fails on the target. Lint for it.
* **Icons are for workloads only.** A package with `<iconUrl>` still gets the stock icon.
* **A remote feed hands the GUI metadata only.** Icons and release notes inside the package are never read from a NuGet server; they need hosted copies. See [publishing](nexus/publishing.md).
* **Ids starting with `TcPkg` are hidden** by the GUI.
* **A nuspec cannot say "not together with".** When two packages exclude each other, the install script is the only guard.
* **Pin your own component versions in a workload.** Otherwise upgrading the workload leaves the components where they were.
* **Dependencies on Beckhoff packages take `0.0.0`.** A pinned TwinCAT version refuses the next TwinCAT update for no reason of yours.

## Scripts

* **Never throw from an uninstall script.** A failing uninstall leaves a package half removed and still listed as installed. `$ErrorActionPreference = 'Continue'`, report, finish.
* **Remove only what you put there.** Mark your files (a comment line, a known folder) and check the mark before deleting.
* **Make the install script safe to run twice.** An upgrade is a second run.
* **Find the payload relative to the script**, with `Split-Path -Parent $MyInvocation.MyCommand.Definition`. The unpack location is the Package Manager's business.
* **Stop a service before you overwrite its files.** The before-modify script exists for that.
* **Check exit codes you understand.** `TcSignTool` returns 1 on success and on failure; `robocopy` returns up to 7 on success; `RepTool`'s codes are undocumented. Read the output or check the result on disk.
* **A tool can accept a wrong input and say nothing.** `TcSignTool grant` stores a wrong password without complaint. Tell the user where the failure will show up instead.
* **Environment variables set by a script are seen by new processes only.** Tell the user to sign out or reboot.
* **The router reads route files only when it starts.** Copying the file is not enough; restart `TcSysSrv`. It is enough in UM mode too, no reboot.
* **Package parameters are best effort.** Beckhoff documents `--params` for the HMI Server only. Always have a default.

## Secrets

* **Whatever is in the package is on the feed in plain text.** A bundled password is readable by anyone who can read the feed. Acceptable on a LAN feed, nowhere else.
* **Keep secrets out of git with a git-ignored folder and a relative `<file src>`.** Have the build fail when the file is missing, so a clone cannot build a hollow package.
* **Delete a bundled password from the unpacked package folder after use.** It is still in the `.nupkg`, but not in the first place someone looks.

## Nexus

* **Nexus answers an unauthenticated write with 403, not 401, when anonymous access is on.** PowerShell's `-Credential` waits for a 401 challenge that never comes. Send the `Authorization: Basic` header up front.
* **Invoke-RestMethod hands a JSON array back as one object** in PowerShell 7.6, so `Where-Object` over the repository list matched the whole list. Use `Invoke-WebRequest` and `ConvertFrom-Json` on the text.
* **Nexus does not advertise a readme template in its NuGet index**, so a feed registered straight on Nexus's `index.json` shows no Feature Release Notes. Register the hosted copy of the index.
* **The NuGet API-Key realm has to be active** (Security, Realms) or the API key from the user menu does nothing.
* **The raw repository needs the write policy *Allow* and strict content validation off**, or re-uploading `index.json` fails and `.md` files are refused.
* **Anonymous read is what the GUI uses.** When an icon or release note shows up for you and not for a colleague, check the URL without a login.

## Beckhoff RT Linux and apt

* **The apt repository is write once per version.** apt caches by version, so a re-upload of a changed file with the same version would be ignored by every client that already saw it. Bump `Version:` in `control`.
* **Nexus signs the repository but does not serve the public key.** `repository-key.gpg` returns 404 on the Community edition. Host the public half somewhere apt clients can fetch it (the raw repository works).
* **A file you want removed on uninstall must not be a conffile.** dpkg keeps conffiles on `remove` and deletes them on `purge` only.
* **Maintainer scripts are `/bin/sh`**, not bash, and `postrm` runs on upgrade too (with `upgrade` as its argument), so test `$1`.
* **Package names are lower case** with hyphens; `Version:` is `x.y.z` or `x.y.z-n`.
* **Restart `TcSystemServiceUm`, not `TcSysSrv`**, on Linux. Same reason as on Windows: the router reads routes when it starts.
