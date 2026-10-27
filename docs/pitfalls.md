# Pitfalls

Everything on this page happened. Most entries are one line; the page they belong to has the long version.

## The Package Manager

* **`tcpkg` is single instance.** A second `TcPkg.exe` fails with "TcPkg is already running". Close the GUI before building or publishing, and check `Get-Process TcPkg` when something fails anyway: the GUI keeps a child process that holds the lock and sometimes leaves it behind on close.
* **Inside a package script you are already inside `tcpkg`.** Do not call `tcpkg` from an install script for anything but a read with `--ignore-mutex`. Read the Package Manager's settings from `%ProgramData%\Beckhoff\TcPkg\appsettings.json` instead of `tcpkg config list`, and check what is installed by looking at the folders under `%ProgramData%\Beckhoff\TcPkg\lib` instead of `tcpkg list`.
* **Scripts run under the Package Manager's own PowerShell,** not the shell you start `tcpkg` from. In `tcpkg` 2.6.24 that is PowerShell 7.4, even when `tcpkg` is called from Windows PowerShell 5.1. Older Package Manager versions were not checked; to be safe with them, keep to what 5.1 also understands.
* **The install script runs from a temporary folder.** `%TEMP%\TcPkg\cache\...` is cleaned up afterwards. Copy the payload to where it belongs; never point a service or a shortcut at the script's folder.
* **`tcpkg install --dry-run` needs an elevated shell** although it changes nothing.
* **Close TwinCAT XAE Shell and Visual Studio before installing.** `tcpkg` refuses with exit code 1051 while one of them runs, even for a package that has nothing to do with the engineering. The tag `NoVisualStudioRestart` exempts a package; workloads are never checked.
* **`tcpkg` does not read answers from a pipe.** Without `-y` in a script it stops with "Input is not possible from a non-interactive shell", exit code 1223. A package with an [upgrade disclaimer](packages/lifecycle.md) needs `--accept-string` on top; `-y` does not accept a disclaimer.
* **`tcpkg` installs the lowest version a dependency allows.** `version="1.0.0"` installs 1.0.0 even when 1.1.0 is on the feed. That is what makes the plain form work as a pin.
* **An unsigned package does not install while signature verification is on.** The download stops with "Package integrity verification failed", exit code 574. Adding and listing the feed still work. Either [sign your packages](hosting/signing.md) and trust the certificate on each PC, or turn verification off with `tcpkg config unset -n VerifySignatures`.
* **Switching verification back on refuses every unsigned package on every feed of that PC**, including versions you published before you started signing. Sign them all first.
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
* **`requireLicenseAcceptance` shows nothing for a plain package.** Not in the terminal, not in the GUI, not from a folder feed or Nexus, with a license file or a URL. Put the license in the package anyway; the text a user accepts is the [feed disclaimer](packages/disclaimer.md).
* **Changing an id needs two packages.** The new id with `DP:<old id>:DP` in its tags, and a last version of the old id that depends on the new one. Without the second, `tcpkg upgrade` never finds the new id. See [retiring an id](packages/lifecycle.md#retiring-an-id).
* **A NuGet server hands the GUI metadata only.** Icons and release notes inside the package are never read from a feed served by a NuGet server such as Nexus or GitHub; they need hosted copies next to the feed. A local folder feed has no such problem. See [hosting](hosting/index.md#inside-the-package-or-hosted-next-to-the-feed).
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
* **Package parameters start with a slash.** `Get-PackageParameters`, the helper every script gets, reads `/Color:Blue`, `/Size=2` and `/Quiet`; a `Color=Blue` without the slash is dropped. Separate them with spaces, and always have a default.
* **Never print a sensitive parameter.** `tcpkg` masks `--params-sensitive` in its own log line, but every line a script prints lands in `TcPkg.log` as it is.
* **No uninstall script means your files stay.** The Package Manager removes its record of the package and nothing else. Beckhoff can skip the script because Windows Installer removes its MSIs; a package that copies files cannot.
* **The install script knows what it replaces.** `previousInstalledPackageVersion` is set on an upgrade and on a downgrade, in the incoming install script only. Use it to remove what the old version shipped and the new one does not.

## Secrets

* **Whatever is in the package is on the feed in plain text.** A bundled password is readable by anyone who can read the feed. Acceptable on a LAN feed, nowhere else.
* **Keep secrets out of source control with an ignored folder and a relative `<file src>`.** A copy of your sources without that folder packs a package without the secret, without a word. Check the package contents, and have the install script throw when the file is missing.
* **Delete a bundled password from the unpacked package folder after use.** It is still in the `.nupkg`, but not in the first place someone looks.

## Hosting

* **GitHub feeds need `--take 100`** on `tcpkg source add`, or the add fails with "Failed to retrieve metadata from source". GitHub refuses pages larger than 100, and the Package Manager asks for 500.
* **GitHub feeds need a login, even for public packages:** your GitHub user name and a *classic* personal access token with `read:packages`.
* **From GitHub, the card shows the package id instead of the title,** and the Engineering and Runtime workloads of one product are two cards.
* **`tcpkg push` writes its command line, API key included, to `C:\ProgramData\Beckhoff\TcPkg\logs\TcPkg.log`.** Push with `dotnet nuget push` instead.
* **Nexus answers an unauthenticated write with 403, not 401, when anonymous access is on.** PowerShell's `-Credential` waits for a 401 challenge that never comes. Send the `Authorization: Basic` header with the first request.
* **Nexus does not advertise a readme template in its NuGet service index,** so a feed registered straight on Nexus's `index.json` shows no Feature Release Notes. Register a hosted copy of the index with the template added.
* **The NuGet API-Key realm has to be active** in Nexus (*Security*, *Realms*), or the API key from the user menu does nothing.
* **The raw repository needs *Allow redeploy* and strict content type validation off,** or uploading `index.json` a second time fails and `.md` files are refused.
* **Anonymous read is what the GUI uses.** When an icon or release note shows up for you and not for a colleague, open the address without a login.
* **`tcpkg download -o` needs a folder that exists**, like `pack`. The bundle it writes with `--zip` installs with `tcpkg import` on a PC that has no feed at all; see [the offline bundle](hosting/folder.md#a-pc-without-a-feed-the-offline-bundle).
* **Push to the hosted NuGet repository, not to the raw copy of the index.** A feed registered on `tcpkg-raw/feeds/myfeed/index.json` is still published to `nuget-hosted`; a push to the raw address answers 400.

## Beckhoff RT Linux and TwinCAT/BSD

From [self-hosting other packages](self-hosting/index.md).

* **Nexus signs an apt repository but does not serve its public key.** `repository-key.gpg` returns 404 on 3.97. Put the key on the raw repository. See [the public key](self-hosting/rt-linux-building.md#step-8-the-public-key-on-the-raw-repository-once).
* **On TwinCAT/BSD, never restart `TcSystemService` from a package script.** `pkg` kills every process the script started, the new service included. Ask TwinCAT for a restart over ADS instead: `ads 127.0.0.1.1.1 state 16`, or `2` in RUN. See [the post-install script](self-hosting/tcbsd-building.md#step-4-the-post-install-script).
* **`pkg repo <folder> rsa:<key>` leaves fingerprint clients with "No signature found".** That form does not embed the public key. Sign with `signing_command:` and the handbook script. See [the signing script](self-hosting/tcbsd-building.md#step-8-the-signing-script-once).
* **A `conflicts` entry in a `pkg` manifest breaks `pkg add`** with a sqlite NOT NULL error when the named package is absent. Check for the other variant in `pre-install`. See [the manifest](self-hosting/tcbsd-building.md#step-2-the-manifest).
* **A `pkg` package is invisible to another FreeBSD major version.** Build once per TwinCAT/BSD release, into a repository folder per release. See [package building for TwinCAT/BSD](self-hosting/tcbsd-building.md).
