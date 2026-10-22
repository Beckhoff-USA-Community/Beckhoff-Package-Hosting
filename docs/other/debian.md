# Debian packages for Beckhoff RT Linux

Beckhoff RT Linux is Debian 13 with the TwinCAT Usermode Runtime on top. Software on it comes from `apt`, and `apt` installs Debian packages, `.deb` files. This page builds the ADS-over-MQTT route package as a Debian package, puts it on an apt repository on Nexus, and installs it on a controller with one `apt install`.

Of the two controller platforms this is the easier one. Nexus has a native apt format and signs the repository itself, and dpkg, the tool under apt, does more of the work than the Package Manager does: it tracks the files it installed, refuses a package that conflicts with an installed one, and checks dependencies before anything runs.

## The package

A Debian package is built from a folder that looks like the target file system, with the metadata in a `DEBIAN` subfolder:

```
baus-adsovermqtt-xar/
  DEBIAN/control
  DEBIAN/postinst
  DEBIAN/postrm
  etc/TwinCAT/3.1/Target/Routes/AdsOverMqtt.xml
```

`AdsOverMqtt.xml` is the [route file](../examples/files.md#a-route-file), with the comment line on top changed to name this package. The route folder does not exist on a fresh controller; dpkg creates it.

### control

The nuspec's counterpart:

```
Package: baus-adsovermqtt-xar
Version: 1.1.0
Section: admin
Priority: optional
Architecture: all
Depends: tc31-xar-um
Conflicts: baus-adsovermqtt-xae
Maintainer: MyCompany <you@example.com>
Description: ADS-over-MQTT connection to the MQTT broker
 Installs the ADS-over-MQTT route file AdsOverMqtt.xml into
 /etc/TwinCAT/3.1/Target/Routes and restarts TcSystemServiceUm so the
 TwinCAT ADS router joins the network at once.
```

`Package`
:   The id. Lower case letters, digits, hyphens, dots and plus signs only. `baus-adsovermqtt-xar` is `BAUS.AdsOverMqtt.XAR` in Debian spelling.

`Version`
:   Bump it for every change. apt caches by version, and the repository refuses to replace one.

`Architecture`
:   `all` for a package without compiled programs, which a route file is.

`Depends`
:   `tc31-xar-um` is the TwinCAT Usermode Runtime. It owns `/usr/bin/TcSystemServiceUm` and its systemd unit. Find the TwinCAT package names on a controller with `dpkg -l | grep -i tc`.

`Conflicts`
:   The field a nuspec does not have. dpkg refuses this package while the engineering variant is installed, and the other way round. The whole variant check of the Windows install script is this one line.

`Description`
:   The first line is the short description. The lines after it, each starting with one space, are the long one.

### The maintainer scripts

The router reads route files only when it starts, the same as on Windows. `postinst` restarts the Usermode system service after dpkg has put the file in place:

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

`postrm` restarts it again after dpkg has removed the file, so the router drops the route:

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

* The scripts are `/bin/sh`, not bash, and run as root.
* `$1` says why a script runs. `postinst` runs with `configure` on install and upgrade. `postrm` runs with `remove` and `purge`, and also with `upgrade`, and an upgrade should restart the service once, not twice.
* A failed restart is reported, not fatal. A `postinst` that exits with an error leaves the package half configured.
* systemd starts the restarted service outside the script's process tree, so restarting from a script just works here. On TwinCAT/BSD it does not; see the [FreeBSD page](freebsd.md#do-not-restart-the-service-from-a-script).

What is not there is a `conffiles` entry. dpkg keeps a file listed in `conffiles` on `apt remove` and deletes it only on `purge`. The route file is deliberately not one, so `apt remove` takes it away and an upgrade replaces it.

Save the scripts with LF line endings. A shell script saved with Windows line endings fails on the controller. If the folder is in Git on Windows, a line such as `baus-adsovermqtt-xar/** text eol=lf` in `.gitattributes` keeps Git from converting them.

## Building it

dpkg-deb builds the package from the folder. It runs on any Debian system, the controller included:

```sh
chmod 0755 baus-adsovermqtt-xar/DEBIAN/postinst baus-adsovermqtt-xar/DEBIAN/postrm
dpkg-deb --build --root-owner-group baus-adsovermqtt-xar baus-adsovermqtt-xar_1.1.0_all.deb
```

`--root-owner-group` is the flag people forget. Without it, the files in the package belong to whoever ran the build, and dpkg installs them that way. The file name is the Debian convention: name, version and architecture.

Look inside before you go on:

```sh
dpkg-deb --info baus-adsovermqtt-xar_1.1.0_all.deb        # the control fields and the scripts
dpkg-deb --contents baus-adsovermqtt-xar_1.1.0_all.deb    # the files, their owners and modes
```

The tested package was built on Windows by a script that writes the same archive directly. A `.deb` is an `ar` archive with three members: `debian-binary`, `control.tar.gz` and `data.tar.gz`. These two checks are the ones it passed.

## Testing it on a controller

Copy the file over and install it with dpkg, before any repository is involved:

```sh
sudo dpkg -i baus-adsovermqtt-xar_1.1.0_all.deb
ls -l /etc/TwinCAT/3.1/Target/Routes/
systemctl status TcSystemServiceUm
sudo apt remove baus-adsovermqtt-xar
ls -l /etc/TwinCAT/3.1/Target/Routes/
```

After the install, the file is in the route folder, the service has restarted, and the broker connection shows up a few seconds later. After the remove, the file is gone and the route with it. `dpkg -i` complains about a missing dependency instead of installing it; `sudo apt install ./baus-adsovermqtt-xar_1.1.0_all.deb`, with the `./`, resolves it from the configured repositories.

## The apt repository on Nexus

apt checks a signature on the repository's index before it trusts a single package. Nexus does the signing: you give the repository a private key once, and Nexus signs the index every time the content changes. The clients need the public half.

### The signing key

Make a key pair once, with GnuPG on any Linux system. No passphrase, because Nexus uses the key without anyone typing one:

```sh
gpg --batch --gen-key <<'EOF'
%no-protection
Key-Type: RSA
Key-Length: 4096
Key-Usage: sign
Name-Real: MyFeed apt repository
Name-Email: you@example.com
Expire-Date: 0
%commit
EOF
gpg --armor --export-secret-keys "MyFeed apt repository" > apt-signing.private.asc
gpg --armor --export "MyFeed apt repository" > myfeed-apt.asc
```

Keep the private half out of source control. The public half is public by nature; every controller downloads it.

### The repository

In the Nexus administration, *Repositories*, *Create repository*, **apt (hosted)**:

* **Name:** `apt-hosted`.
* **Distribution:** the Debian release name the clients ask for. Beckhoff RT Linux is Debian 13, so `trixie`.
* **Signing key:** the contents of `apt-signing.private.asc`. **Passphrase** empty.
* **Deployment policy:** *Disable redeploy*. A changed package gets a new version, never a re-upload.

### The public key, on the raw repository

Nexus signs the repository but does not serve the public key. The `repository-key.gpg` address that some guides mention returns 404 on 3.97. Put the key on the raw repository, which every client reads without a login, with the `$auth` header from [uploading to the raw repository](../hosting/nexus.md#uploading-to-the-raw-repository):

```powershell
Invoke-WebRequest -Uri 'https://nexus.example.com/repository/tcpkg-raw/apt/myfeed-apt.asc' -Method Put -InFile .\myfeed-apt.asc -Headers $auth
```

Then open the address in a browser that is not logged in to Nexus. That is the request every controller will make.

## Publishing

A `.deb` goes up through Nexus's components API, one multipart `POST` per file, with the file in a field named `apt.asset`:

```powershell
Invoke-RestMethod -Uri 'https://nexus.example.com/service/rest/v1/components?repository=apt-hosted' `
    -Method Post -Headers $auth -Form @{ 'apt.asset' = Get-Item .\baus-adsovermqtt-xar_1.1.0_all.deb }
```

`-Form` needs PowerShell 7; Windows PowerShell 5.1 does not have it. Send the `Authorization` header with the request, for the [same reason as on the raw repository](../hosting/nexus.md#uploading-to-the-raw-repository).

Nexus then rebuilds the package lists and signs `dists/trixie/InRelease`. An `Architecture: all` package lands in `main/binary-all/Packages`, and apt on an amd64 controller finds it there. To check, open `https://nexus.example.com/repository/apt-hosted/dists/trixie/InRelease` without a login.

## Setting up a controller

On the controller, as a user with `sudo`. The same shape as Beckhoff's own repository entry, with an armored key named by `signed-by`:

```sh
sudo curl -fsSL https://nexus.example.com/repository/tcpkg-raw/apt/myfeed-apt.asc -o /etc/apt/keyrings/myfeed-apt.asc
echo "deb [signed-by=/etc/apt/keyrings/myfeed-apt.asc] https://nexus.example.com/repository/apt-hosted trixie main" | sudo tee /etc/apt/sources.list.d/myfeed.list
sudo apt update && sudo apt install baus-adsovermqtt-xar
```

* The key goes to `/etc/apt/keyrings/`, the folder Debian keeps for the keys of other repositories. `signed-by` trusts it for this one repository, not for every repository on the box.
* `trixie` is the distribution the repository was created with. `main` is the component Nexus puts everything under.
* If `apt update` fails with "Sub-process /usr/bin/sqv returned an error code" and "The repository ... is not signed", the key file is missing or does not match the key in the repository settings. The repository itself is fine.

A new version reaches a controller with the same `sudo apt update && sudo apt install baus-adsovermqtt-xar`. A plain `sudo apt upgrade` picks it up as well, but it also upgrades TwinCAT from Beckhoff's repository, which is a decision of its own. The upgrade unpacks the new file over the old one and runs the new `postinst`, so the router restarts once with the new route. `sudo apt remove baus-adsovermqtt-xar` deletes the file, and `postrm` makes the router drop the route.
