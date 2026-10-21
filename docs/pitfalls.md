# Pitfalls

Everything on this page happened. Most entries are one line; the page they belong to has the long version.

## The Package Manager

* **`tcpkg` is single instance.** A second `TcPkg.exe` fails with "TcPkg is already running". Close the GUI before building or publishing, and check `Get-Process TcPkg` when something fails anyway: the GUI keeps a child process that holds the lock and sometimes leaves it behind on close.
* **Inside a package script you are already inside `tcpkg`.** Do not call `tcpkg` from an install script for anything but a read with `--ignore-mutex`. Read the Package Manager's settings from `%ProgramData%\Beckhoff\TcPkg\appsettings.json` instead of `tcpkg config list`, and check what is installed by looking at the folders under `%ProgramData%\Beckhoff\TcPkg\lib` instead of `tcpkg list`.
* **Scripts run under the Package Manager's own PowerShell.** That is PowerShell 7.4 in `tcpkg` 2.6.24, and was Windows PowerShell 5.1 in earlier versions, where PowerShell 7 syntax fails with a parse error that names a line you did not think twice about. Write for 5.1 and the script runs in both.
* **The install script runs from a temporary folder.** `%TEMP%\TcPkg\cache\...` is cleaned up afterwards. Copy the payload to where it belongs; never point a service or a shortcut at the script's folder.
* **`tcpkg install --dry-run` needs an elevated shell** although it changes nothing.
* **Close TwinCAT XAE Shell and Visual Studio before installing.** `tcpkg` refuses with exit code 1051 while one of them runs, even for a package that has nothing to do with the engineering.
* **`tcpkg` installs the lowest version a dependency allows.** `version="1.0.0"` installs 1.0.0 even when 1.1.0 is on the feed. That is what makes the plain form work as a pin.
* **Turn signature verification off before adding an unsigned feed**, with `tcpkg config unset -n VerifySignatures`. The order matters: with verification on, adding the feed fails, and so does every install from it.
* **Feed metadata is cached for about 30 minutes.** A package you just published is not there yet. `--no-cache` on the command, *Reload Information* in the GUI, or delete the feed's folder under `%LOCALAPPDATA%\NuGet\v3-cache`.
* **Uninstalling a workload with `--include-dependencies` takes `TwinCAT.XAE.Base` with it.** Name the component packages instead.
* **Elevated tests and network drives do not mix.** Copy packages to a local folder before `tcpkg source add`, and start an elevated shell with `-WorkingDirectory C:\`.

## Metadata

* **A published version is immutable.** Every change, however small, is a new version. On a feed you control you can delete a version and publish it again; do that for a fix you made five minutes ago and never for a version anyone has installed, because their PC will not see a difference.
* **Three part versions.** `tcpkg pack` names a `1.2.0.0` package `...1.2.0.0.nupkg` while the nuspec inside says `1.2.0`, and from there file names, folder names and feed listings disagree.
* **`tcpkg pack -o` needs a folder that exists.** It does not create one. Without `-o` the package lands next to the nuspec, among your sources.
* **The description is printed raw.** One or two plain sentences. Line breaks, bullets and Markdown show up as typed.
* **The release notes parser knows headings and sentences.** Bullet lists render as `[object Object]`. Bold, backticks and links print their markers.
* **A `<file src>` that matches nothing is skipped without a word.** The package builds, ships, and fails on the target. List what is in the package before you ship it.
* **Icons are for workloads only.** A package with `<iconUrl>` still gets the stock icon.
* **A NuGet server hands the GUI metadata only.** Icons and release notes inside the package are never read from a feed served by a NuGet server such as Nexus; they need hosted copies next to the feed. A local folder feed has no such problem.
* **Ids starting with `TcPkg` are hidden** by the GUI.
* **A nuspec cannot say "not together with".** When two packages exclude each other, the install script is the only guard.
* **Pin your own component versions in a workload.** Otherwise upgrading the workload leaves the components where they were.
* **Dependencies on Beckhoff packages take `0.0.0`.** A pinned TwinCAT version refuses the next TwinCAT update for no reason of yours.

## Scripts

* **Never throw from an uninstall script.** A failing uninstall leaves a package half removed and still listed as installed. `$ErrorActionPreference = 'Continue'`, report, finish.
* **Remove only what you put there.** Mark your files (a comment line, a known folder) and check the mark before deleting.
* **An upgrade does not run the uninstall script.** It runs the old version's before-modify script and then the new version's install script over the old files. Make the install script safe to run over an older install, and have it remove what a newer version no longer ships.
* **The before-modify script of the old version is the one that runs on an upgrade.** A fix to it takes effect one upgrade later.
* **Package parameters are not remembered.** An upgrade runs the install script without the parameters given at install time.
* **Find the payload relative to the script**, with `Split-Path -Parent $MyInvocation.MyCommand.Definition`. The unpack location is the Package Manager's business.
* **Stop a service before you overwrite its files.** The before-modify script exists for that.
* **Check exit codes you understand.** `TcSignTool` returns 1 on success and on failure; `robocopy` returns up to 7 on success; `RepTool`'s codes are undocumented. Read the output or check the result on disk.
* **A tool can accept a wrong input and say nothing.** `TcSignTool grant` stores a wrong password without complaint. Tell the user where the failure will show up instead.
* **Environment variables set by a script are seen by new processes only.** Tell the user to sign out or reboot.
* **The router reads route files only when it starts.** Copying the file is not enough; restart `TcSysSrv`. It is enough in UM mode too, no reboot.
* **Package parameters are best effort.** Beckhoff documents `--params` for the HMI Server only. Separate them with spaces, not semicolons, and always have a default.

## Secrets

* **Whatever is in the package is on the feed in plain text.** A bundled password is readable by anyone who can read the feed. Acceptable on a LAN feed, nowhere else.
* **Keep secrets out of source control with an ignored folder and a relative `<file src>`.** A copy of your sources without that folder packs a package without the secret, without a word. Check the package contents, and have the install script throw when the file is missing.
* **Delete a bundled password from the unpacked package folder after use.** It is still in the `.nupkg`, but not in the first place someone looks.
