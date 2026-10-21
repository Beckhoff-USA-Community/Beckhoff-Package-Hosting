# From tcpkg to .deb

Beckhoff RT Linux is Debian with the TwinCAT Usermode Runtime on top. Debian installs software with `apt`, and `apt` installs Debian packages, `.deb` files. The TwinCAT Package Manager does not run there, so a package that should land on a Linux controller has to become a Debian package. This page takes the ADS-over-MQTT route package from this repository through that, and the result installs with one `apt install` from the same Nexus the Windows feed lives on.

The good news is that a Debian package is simpler than a TwinCAT package. dpkg, the tool under apt, does more on its own: it tracks every file it installed and removes them on uninstall, it refuses to install a package next to one it conflicts with, and it checks dependencies before anything runs. Three of the four jobs the Windows install script does by hand come for free.

## What the Windows package does

| Job | On Windows | On Linux |
|---|---|---|
| Put the route file in the router's route folder | the install script copies it into every route folder | dpkg unpacks it to `/etc/TwinCAT/3.1/Target/Routes/AdsOverMqtt.xml` |
| Make the router read it | the script restarts `TcSysSrv` | `postinst` restarts `TcSystemServiceUm` |
| Refuse the other variant | the script checks the package folders and route files | `Conflicts: baus-adsovermqtt-xae` in `control` |
| Require the runtime | `<dependency id="TwinCAT.XAR.Base">` | `Depends: tc31-xar-um` |
| Remove the file on uninstall | the script deletes files carrying its marker | dpkg removes the files it installed |
| Make the router drop it | the script restarts the service | `postrm` restarts `TcSystemServiceUm` |

The Linux side of each of those:

* The TwinCAT Usermode Runtime comes from the apt package `tc31-xar-um`. It owns `/usr/bin/TcSystemServiceUm` and the systemd unit `TcSystemServiceUm.service`, which is the Linux twin of the Windows system service. The route folder is `/etc/TwinCAT/3.1/Target/Routes/`; it does not exist on a fresh system, and the package creates it.
* The router reads route files when it starts, same as on Windows, so `systemctl restart TcSystemServiceUm` is the activation. Verified on a CX2043: the broker connection was up within seconds.

## The folder

```
packaging\deb\baus-adsovermqtt-xar\
  control          metadata; the nuspec's counterpart
  postinst         runs after the files are in place
  postrm           runs after the files are removed
```

That is the whole source. The route file itself comes from the same git-ignored `packaging\secrets\BAUS.AdsOverMqtt.XAR\AdsOverMqtt.xml` the Windows package uses, with its "installed by" comment swapped for the apt wording at build time.

## control

```
Package: baus-adsovermqtt-xar
Version: 1.0.0
Section: admin
Priority: optional
Architecture: all
Depends: tc31-xar-um
Conflicts: baus-adsovermqtt-xae
Maintainer: Beckhoff Automation LLC <you@example.com>
Homepage: https://github.com/0w8States/Iris-Manor-Custom-Nexus-Packages
Description: ADS-over-MQTT connection to the MQTT broker
 Installs the ADS-over-MQTT route file AdsOverMqtt.xml with the runtime
 credentials into /etc/TwinCAT/3.1/Target/Routes and restarts
 TcSystemServiceUm so the TwinCAT ADS router joins the network at once.
 The Beckhoff RT Linux counterpart of the TwinCAT Package Manager package
 BAUS.AdsOverMqtt.XAR.
```

Field by field, against the nuspec:

`Package`
:   The id. Lower case, digits, hyphens, dots and plus signs only. `baus-adsovermqtt-xar` is `BAUS.AdsOverMqtt.XAR` in Debian spelling.

`Version`
:   `x.y.z`, or `x.y.z-n` when the packaging changes and the content does not. Every change needs a new version, because apt caches by version; the [apt page](apt.md) says why that matters more than on Windows.

`Architecture`
:   `all` for a package with no compiled code, which a route file is. A package with binaries says `amd64` for the x86 controllers or `arm64` for the ARM ones, and ships once per architecture.

`Depends`
:   What has to be installed first. apt installs it when missing. Version constraints look like `tc31-xar-um (>= 3.1)`; a bare name means any version, which is the `0.0.0` of nuspecs.

`Conflicts`
:   The field a nuspec does not have. dpkg refuses to install this package while the named one is installed, and refuses the named one while this is. The whole variant check from the Windows install script is this one line.

`Description`
:   First line is the short description, the lines after it, each starting with one space, are the long one. apt prints the short one in lists and the long one in `apt show`.

`Section`, `Priority`, `Maintainer`, `Homepage`
:   Required or conventional; the values above are fine for anything in-house.

## The maintainer scripts

Four hooks exist: `preinst`, `postinst`, `prerm`, `postrm`. They are `/bin/sh` scripts (not bash), run as root, and get an argument that says why they run. The route package needs two of them, both for the service restart:

```sh
#!/bin/sh
# postinst: dpkg has put the route file in place; the TwinCAT ADS router only reads
# the route folder when it initialises, so restart the Usermode system service.
set -e

case "$1" in
    configure)
        if [ -d /run/systemd/system ] && systemctl list-unit-files TcSystemServiceUm.service >/dev/null 2>&1; then
            echo "Restarting TcSystemServiceUm so the ADS router picks up the ADS-over-MQTT route ..."
            if systemctl restart TcSystemServiceUm.service; then
                echo "TcSystemServiceUm restarted; the route comes up within a few seconds."
            else
                echo "Could not restart TcSystemServiceUm; restart it (or reboot) yourself." >&2
            fi
        else
            echo "TcSystemServiceUm.service not found; restart TwinCAT so the router picks up the route." >&2
        fi
        ;;
esac

exit 0
```

```sh
#!/bin/sh
# postrm: dpkg has removed the route file; restart the service so the router drops the route.
# On an upgrade the new package's postinst restarts, so only remove and purge act here.
set -e

case "$1" in
    remove|purge)
        if [ -d /run/systemd/system ] && systemctl is-active --quiet TcSystemServiceUm.service; then
            echo "Restarting TcSystemServiceUm so the ADS router drops the ADS-over-MQTT route ..."
            systemctl restart TcSystemServiceUm.service || echo "Could not restart TcSystemServiceUm; restart it (or reboot) yourself." >&2
        fi
        ;;
esac

exit 0
```

The habits in there:

* `set -e` and `exit 0`, so that an unexpected error stops the script and a handled one does not fail the install. A `postinst` that exits non zero leaves the package half configured, the Linux version of a throwing uninstall script.
* Test `$1`. `postinst` runs with `configure` on install and upgrade; `postrm` runs with `remove`, `purge`, and also `upgrade`, and you do not want to restart the service twice on an upgrade.
* A failed service restart is reported, not fatal. The file is in place either way; the user can restart.

What is not in there is a conffiles entry. dpkg treats a file listed in `conffiles` as configuration: it keeps it on `remove`, merges it on upgrade, and deletes it on `purge` only. The route file is deliberately not one, so that `apt remove` takes it away and the router drops the route. The gamepad service package does the opposite for its `appsettings.json`: it is created by `postinst` on the first install rather than shipped as a package file, so no upgrade ever touches it. Decide per file.

## Building the .deb

A `.deb` is an `ar` archive with three members: `debian-binary` holding the text `2.0`, `control.tar.gz` with `control`, the maintainer scripts and an `md5sums` list, and `data.tar.gz` with the files to install, owned by root. On Linux `dpkg-deb` builds it from a staging folder:

```sh
STAGE=$(mktemp -d)
mkdir -p "$STAGE/etc/TwinCAT/3.1/Target/Routes" "$STAGE/DEBIAN"
cp AdsOverMqtt.xml "$STAGE/etc/TwinCAT/3.1/Target/Routes/"
cp control postinst postrm "$STAGE/DEBIAN/"
chmod 0755 "$STAGE/DEBIAN/postinst" "$STAGE/DEBIAN/postrm"
dpkg-deb --build --root-owner-group "$STAGE" baus-adsovermqtt-xar_1.0.0_all.deb
```

`--root-owner-group` is the flag people forget: without it the files in the package are owned by whoever ran the build, and dpkg installs them that way.

The packages in this repository are built on Windows, where there is no `dpkg-deb`. `packaging\scripts\deb_build.py` writes the same archive with Python's standard library: a thirty line `ar` writer and `tarfile` for the two tarballs, with every entry owned by root and the scripts marked executable. `Build-Deb.ps1` calls it with the control folder, the files to install and their destinations, and the comment swap:

```powershell
cd .\packaging\scripts
.\Build-Deb.ps1 -Clean
```

The output lands in `packaging\feed-deb\baus-adsovermqtt-xar_1.0.0_all.deb`. Nothing in that file knows it was built on Windows; dpkg reads it like any other package. Verified with `dpkg-deb --info`, `dpkg-deb --contents` and a real install on a CX2043.

## Testing it on a controller

Copy the file over and install it with dpkg directly, before any apt repository is involved:

```sh
dpkg-deb --info baus-adsovermqtt-xar_1.0.0_all.deb        # control fields and scripts
dpkg-deb --contents baus-adsovermqtt-xar_1.0.0_all.deb    # files, owners, modes
sudo dpkg -i baus-adsovermqtt-xar_1.0.0_all.deb
systemctl status TcSystemServiceUm
ls -l /etc/TwinCAT/3.1/Target/Routes/
sudo apt remove baus-adsovermqtt-xar
```

`dpkg -i` complains when a dependency is missing instead of installing it; `apt install ./baus-adsovermqtt-xar_1.0.0_all.deb` (note the `./`) resolves dependencies from the configured repositories and is the better test once the runtime is there. After the install the file is in the route folder, the service has restarted, and the broker connection shows up from the engineering side a few seconds later. After the remove, the file is gone and the route with it.

## A package with a program in it

The route package has one file. The gamepad project's Debian package is the fuller example: a published .NET application under `/opt/ads-gamepad-service/`, a systemd unit under `/lib/systemd/system/`, a udev rule under `/lib/udev/rules.d/`, a `preinst` that stops the running service, a `postinst` that creates a service account, puts the first configuration file in place, reloads udev and systemd and starts the unit, a `prerm` that stops and disables it, and a `postrm` that deletes the configuration on `purge` only. It is `linux\build-deb.sh` and `linux\deb\` in the ADS-Gamepad-Service repository, and it is the same ideas as the [.NET service example](../examples/dotnet-service.md) with dpkg doing the file tracking.
