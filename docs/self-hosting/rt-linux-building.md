# Package building for Beckhoff RT Linux

Beckhoff RT Linux is Debian 13 with the TwinCAT Usermode Runtime on top. Software on it comes from `apt`, and `apt` installs Debian packages, `.deb` files. This page builds the ADS-over-MQTT route package as a `.deb`, tests it on a controller, and publishes it to an apt repository on Nexus. [Client package usage on Beckhoff RT Linux](rt-linux-clients.md) then installs it from there.

A Debian package is simpler than a TwinCAT package. dpkg, the tool under `apt`, keeps track of the files it installed and removes them again, refuses a package that conflicts with an installed one, and checks dependencies before anything runs. Nexus has a native apt format and signs the repository for you.

The steps:

* Steps 1 to 5 build the package and test it on a controller.
* Steps 6 to 8 set up the apt repository on Nexus. You do them once.
* Step 9 publishes the package, and is what you repeat for every new version.

## What you need

* **A Debian system to build on.** The controller itself works: it has `dpkg-deb`. Run the commands of steps 1 to 5 there, from your home folder.
* **The route file**, `AdsOverMqtt.xml`, copied to that folder. Change the comment line on top so it names this package.
* **GnuPG** on any Linux system, once, for the signing key in step 6.
* **PowerShell 7** on the PC you publish from, with the `$auth` header from [uploading to the raw repository](../hosting/nexus.md#uploading-to-the-raw-repository).

## Step 1: the package folder

A Debian package is built from a folder that looks like the target file system, with the metadata in a `DEBIAN` subfolder. When the steps are done, it looks like this:

```
baus-adsovermqtt-xar/
  DEBIAN/control                                   the metadata
  DEBIAN/postinst                                  runs after the files are in place
  DEBIAN/postrm                                    runs after the files are removed
  etc/TwinCAT/3.1/Target/Routes/AdsOverMqtt.xml    the route file
```

Create the folders:

```sh
mkdir -p baus-adsovermqtt-xar/DEBIAN baus-adsovermqtt-xar/etc/TwinCAT/3.1/Target/Routes
```

Copy the route file into place:

```sh
cp AdsOverMqtt.xml baus-adsovermqtt-xar/etc/TwinCAT/3.1/Target/Routes/
```

## Step 2: the control file

`DEBIAN/control` is the nuspec's counterpart. Create it with this content:

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

* `Package` is the id: lower case, digits, hyphens and dots.
* `Version` goes up with every change. The repository refuses to replace a version.
* `Architecture: all` is for a package without compiled programs.
* `Depends: tc31-xar-um` is the TwinCAT Usermode Runtime.
* `Conflicts` keeps the engineering variant out. It is the field a nuspec does not have.
* `Description`: the first line is the short text; the lines after it start with one space.

## Step 3: the install script

The router reads route files only when it starts. `DEBIAN/postinst` runs after dpkg has put the file in place, and restarts the Usermode system service:

```sh
#!/bin/sh
# postinst: restart the Usermode system service so the ADS router reads the new route.
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

`$1` says why the script runs; `configure` means install or upgrade. A failed restart is reported, not fatal: a `postinst` that fails leaves the package half installed.

## Step 4: the remove script

`DEBIAN/postrm` runs after dpkg has removed the file, and restarts the service so the router drops the route:

```sh
#!/bin/sh
# postrm: restart the Usermode system service so the ADS router drops the route.
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

Both scripts must be executable:

```sh
chmod 0755 baus-adsovermqtt-xar/DEBIAN/postinst baus-adsovermqtt-xar/DEBIAN/postrm
```

If you write the scripts on Windows, save them with LF line endings. A shell script with Windows line endings fails on the controller.

## Step 5: build and test

Build the package. `--root-owner-group` makes root the owner of every file in it; without it, the files belong to whoever ran the build:

```sh
dpkg-deb --build --root-owner-group baus-adsovermqtt-xar baus-adsovermqtt-xar_1.1.0_all.deb
```

Check the metadata and the scripts:

```sh
dpkg-deb --info baus-adsovermqtt-xar_1.1.0_all.deb
```

Check the files, their owners and their paths:

```sh
dpkg-deb --contents baus-adsovermqtt-xar_1.1.0_all.deb
```

Install it on the controller, straight from the file:

```sh
sudo dpkg -i baus-adsovermqtt-xar_1.1.0_all.deb
```

The route file is now in the route folder, and the service has restarted:

```sh
ls -l /etc/TwinCAT/3.1/Target/Routes/
```

Remove it again. The file is gone afterwards, and the router drops the route:

```sh
sudo apt remove baus-adsovermqtt-xar
```

The route file is deliberately not listed as a configuration file (`conffiles`). dpkg keeps those on `remove`; this way `remove` deletes it, and an upgrade replaces it.

## Step 6: a signing key (once)

`apt` checks a signature on the repository before it trusts a single package. Nexus does the signing with a key you give it. Make one with GnuPG, without a passphrase, because Nexus uses it without anyone typing one:

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
```

Export the private half, for Nexus. Keep it out of source control:

```sh
gpg --armor --export-secret-keys "MyFeed apt repository" > apt-signing.private.asc
```

Export the public half, for the controllers:

```sh
gpg --armor --export "MyFeed apt repository" > myfeed-apt.asc
```

## Step 7: the apt repository (once)

In the Nexus administration, *Repositories*, *Create repository*, **apt (hosted)**:

* **Name:** `apt-hosted`.
* **Distribution:** `trixie`, the name of Debian 13. The controllers ask for it by that name.
* **Signing key:** the contents of `apt-signing.private.asc`. Leave **Passphrase** empty.
* **Deployment policy:** *Disable redeploy*, so a published version never changes.

## Step 8: the public key on the raw repository (once)

Nexus signs the repository but does not serve the public key; the `repository-key.gpg` address some guides mention returns 404 on 3.97. Upload the key to the raw repository, from the PC:

```powershell
Invoke-WebRequest -Uri 'https://nexus.example.com/repository/tcpkg-raw/apt/myfeed-apt.asc' -Method Put -InFile .\myfeed-apt.asc -Headers $auth
```

Then open `https://nexus.example.com/repository/tcpkg-raw/apt/myfeed-apt.asc` in a browser that is not logged in to Nexus. Every controller makes that request.

## Step 9: publish the package

Copy the `.deb` to the PC and upload it through the Nexus components API. `-Form` needs PowerShell 7:

```powershell
Invoke-RestMethod -Uri 'https://nexus.example.com/service/rest/v1/components?repository=apt-hosted' -Method Post -Headers $auth -Form @{ 'apt.asset' = Get-Item .\baus-adsovermqtt-xar_1.1.0_all.deb }
```

Nexus adds the package to the repository's index and signs the index again. To check, open `https://nexus.example.com/repository/apt-hosted/dists/trixie/main/binary-all/Packages` without a login; the package is listed there.

## A new version

Raise `Version` in `control`, then repeat step 5 with the new file name and step 9. Controllers pick it up as described on the [client page](rt-linux-clients.md#upgrading-and-removing).

Next: [client package usage on Beckhoff RT Linux](rt-linux-clients.md).
