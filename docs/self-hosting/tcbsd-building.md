# Package building for TwinCAT/BSD

TwinCAT/BSD is FreeBSD with TwinCAT on top. Software on it comes from `pkg`, and `pkg` installs FreeBSD packages, `.pkg` files. This page builds the ADS-over-MQTT route package as a `.pkg`, tests it, and publishes a signed package repository on Nexus. [Client package usage on TwinCAT/BSD](tcbsd-clients.md) then installs it from there.

Nexus has no FreeBSD format, and it does not need one. A `pkg` repository is a folder of plain files: the packages, plus an index that `pkg repo` writes next to them. The raw repository serves that folder as it is. `pkg repo` runs only on FreeBSD, so a TwinCAT/BSD box does the building here, not only the testing.

**One build per release.** `pkg` refuses a package built for another FreeBSD major version, so a TwinCAT/BSD 14 box never sees a package built for 15. Build the package once per release, and publish each build to its own folder:

| TwinCAT/BSD | ABI in the manifest | `TC31-XAR` as shipped | `TcAdsTool` as shipped | Repository folder |
|---|---|---|---|---|
| 14 (FreeBSD 14.4) | `FreeBSD:14:*` | 4026.29.3_20 | 0.113.0.30 | `tcpkg-raw/pkg/tcbsd14` |
| 15 (FreeBSD 15.1) | `FreeBSD:15:*` | 4026.28.21_21 | 0.113.0.30 | `tcpkg-raw/pkg/tcbsd15` |

Beckhoff lays out its own repository the same way, one folder per release. The steps below build for release 15; for 14, use the values from the other row.

The steps:

* Steps 1 to 6 build the package and test it on the box.
* Steps 7 and 8 make the signing key and script. You do them once.
* Steps 9 and 10 build the repository and publish it, and are what you repeat for every new version.

## What you need

* **A TwinCAT/BSD box**, with ssh access. Either release can build the repository for both.
* **Root on the box.** Run every command on the box in a root shell. Open it with `doas sh` in your home folder; it stays there, so the files land where `scp` finds them in step 10.
* **The route file**, `AdsOverMqtt.xml`, copied to the box, for example with `scp AdsOverMqtt.xml <user>@<box>:`. Change the comment line on top so it names this package.
* **PowerShell** on the PC you publish from, with the `$auth` header from [uploading to the raw repository](../hosting/nexus.md#uploading-to-the-raw-repository).

## Step 1: the package folder

`pkg create` builds a package from a folder of metadata, a staging folder that looks like the target file system, and a list of the files. When the steps are done, it looks like this:

```
baus-adsovermqtt-xar/
  meta/+MANIFEST          the metadata
  meta/+PRE_INSTALL       runs before the files are placed
  meta/+POST_INSTALL      runs after the files are placed
  meta/+POST_DEINSTALL    runs after the files are removed
  plist                   the list of files
  stage/usr/local/etc/TwinCAT/3.1/Target/Routes/AdsOverMqtt.xml
```

Create the folders:

```sh
mkdir -p baus-adsovermqtt-xar/meta baus-adsovermqtt-xar/stage/usr/local/etc/TwinCAT/3.1/Target/Routes
```

Copy the route file into place:

```sh
cp AdsOverMqtt.xml baus-adsovermqtt-xar/stage/usr/local/etc/TwinCAT/3.1/Target/Routes/
```

Write the list of files. The path is relative to `/usr/local`, the package's prefix:

```sh
echo 'etc/TwinCAT/3.1/Target/Routes/AdsOverMqtt.xml' > baus-adsovermqtt-xar/plist
```

## Step 2: the manifest

`meta/+MANIFEST` holds the metadata. `pkg` reads UCL, and JSON is valid UCL. Create it with this content:

```json
{
  "name": "baus-adsovermqtt-xar",
  "version": "1.1.1",
  "origin": "baus/baus-adsovermqtt-xar",
  "comment": "ADS-over-MQTT connection to the MQTT broker",
  "desc": "Installs the ADS-over-MQTT route file AdsOverMqtt.xml and asks TwinCAT for a restart over ADS, so the ADS router joins the network at once.",
  "maintainer": "you@example.com",
  "www": "https://www.example.com",
  "prefix": "/usr/local",
  "abi": "FreeBSD:15:*",
  "arch": "freebsd:15:*",
  "deps": {
    "TC31-XAR":  { "origin": "bhf/TC31-XAR",  "version": "4026.28.21_21" },
    "TcAdsTool": { "origin": "bhf/TcAdsTool", "version": "0.113.0.30" }
  }
}
```

`TC31-XAR` is the TwinCAT runtime. `TcAdsTool` brings the `ads` command the scripts use. `pkg` resolves dependencies by name; the versions are the ones the release ships, so `pkg info` tells the truth.

There is no `conflicts` entry, on purpose. One that names a package that is not installed makes `pkg add` fail with a sqlite NOT NULL error. Step 3 keeps the engineering variant out instead.

## Step 3: the pre-install script

`meta/+PRE_INSTALL` runs before `pkg` places the files. It looks for a route file of the engineering variant, by its comment line, and stops the install if it finds one:

```sh
#!/bin/sh
# pre-install: the engineering and runtime variants carry different credentials and exclude each other.
routes=/usr/local/etc/TwinCAT/3.1/Target/Routes
for f in "$routes"/*.xml; do
    [ -f "$f" ] || continue
    if grep -q -E "BAUS.AdsOverMqtt.XAE|baus-adsovermqtt-xae" "$f"; then
        echo "$f belongs to the engineering variant of the ADS-over-MQTT route. Remove it first." >&2
        exit 1
    fi
done
exit 0
```

`pkg` then stops with "PRE-INSTALL script failed". The script reads the folder because `pkg` cannot be asked what is installed while it is installing.

## Step 4: the post-install script

The router reads route files only when TwinCAT starts. Do not restart the service from the script: `pkg` kills every process a script started when the script ends, and that includes the restarted `TcSystemService`. Ask the running service for a TwinCAT restart over ADS instead, as the *Restart TwinCAT* button in the engineering does. `meta/+POST_INSTALL`:

```sh
#!/bin/sh
# Ask TwinCAT for a restart over ADS, so the ADS router reads the route folder again.
# A service restart from here would not survive: pkg ends every process this script starts.
ads=/usr/local/bin/ads
if [ -x "$ads" ]; then
    state=$("$ads" 127.0.0.1.1.1 state 2>/dev/null)          # 5 = RUN, 15 = CONFIG
    if [ "$state" = "5" ]; then cmd=2; else cmd=16; fi        # 2 = RESET, 16 = RECONFIG
    echo "ADS state $state: requesting a TwinCAT restart so the router reads the ADS-over-MQTT route ..."
    "$ads" 127.0.0.1.1.1 state "$cmd" || echo "The ADS request failed; run: doas service TcSystemService restart" >&2
else
    echo "$ads not found (package TcAdsTool); run: doas service TcSystemService restart" >&2
fi
exit 0
```

RESET (2) restarts TwinCAT into RUN, so a boot project comes back up. RECONFIG (16) restarts it into CONFIG. Either way the router reconnects to the broker within seconds.

## Step 5: the post-deinstall script

After a removal, the router has to forget the route, which takes the same restart. Use the same script:

```sh
cp baus-adsovermqtt-xar/meta/+POST_INSTALL baus-adsovermqtt-xar/meta/+POST_DEINSTALL
```

If you write the scripts on Windows, save them with LF line endings. A shell script with Windows line endings fails on the box.

## Step 6: build and test

Build the package. It lands in the current folder as `baus-adsovermqtt-xar-1.1.1.pkg`:

```sh
pkg create -m baus-adsovermqtt-xar/meta -r baus-adsovermqtt-xar/stage -p baus-adsovermqtt-xar/plist
```

Check the metadata: name, version, ABI and dependencies:

```sh
pkg info -F baus-adsovermqtt-xar-1.1.1.pkg
```

Check the files:

```sh
pkg info -l -F baus-adsovermqtt-xar-1.1.1.pkg
```

Note the process ID of the TwinCAT system service:

```sh
pgrep -l TcSystemService
```

Install the package from the file:

```sh
pkg add baus-adsovermqtt-xar-1.1.1.pkg
```

The route file is now in the route folder:

```sh
ls -l /usr/local/etc/TwinCAT/3.1/Target/Routes/
```

The process ID is the same as before. That means TwinCAT restarted over ADS and the system service survived:

```sh
pgrep -l TcSystemService
```

Remove the package again. The route file is gone afterwards:

```sh
pkg delete -y baus-adsovermqtt-xar
```

## Step 7: a signing key (once)

Clients trust your repository by the fingerprint of the key that signed it. Make a folder for the key:

```sh
mkdir -p /root/pkg-signing
```

Make the private key:

```sh
openssl genrsa -out /root/pkg-signing/repo.key 2048
```

Make it readable by root only:

```sh
chmod 600 /root/pkg-signing/repo.key
```

Make the public key from it:

```sh
openssl rsa -in /root/pkg-signing/repo.key -pubout -out /root/pkg-signing/repo.pub
```

Print the fingerprint. Every controller needs this value:

```sh
sha256 -q /root/pkg-signing/repo.pub
```

Keep a copy of `repo.key` somewhere safe, out of source control. Whoever has it can sign packages your controllers trust.

## Step 8: the signing script (once)

`pkg repo` can sign in two ways, and only one of them works for clients that check a fingerprint, as Beckhoff's own repository and yours do. `rsa:<keyfile>` leaves out the public key, and those clients answer "No signature found". The other way runs a script that returns the signature and the public key together. Create `/root/pkg-signing/sign.sh`, from the FreeBSD handbook, with this content:

```sh
#!/bin/sh
read -t 2 sum
[ -z "$sum" ] && exit 1
echo SIGNATURE
echo -n "$sum" | openssl dgst -sign /root/pkg-signing/repo.key -sha256 -binary
echo
echo CERT
cat /root/pkg-signing/repo.pub
echo END
```

## Step 9: build the repository

Make the repository folder for this release, with the packages under `All`:

```sh
mkdir -p repo/tcbsd15/All
```

Copy the package into it:

```sh
cp baus-adsovermqtt-xar-1.1.1.pkg repo/tcbsd15/All/
```

Write the signed index:

```sh
pkg repo -o repo/tcbsd15 repo/tcbsd15 signing_command: sh /root/pkg-signing/sign.sh
```

The folder now holds `meta.conf`, `meta`, `packagesite.pkg`, `data.pkg` and `All/baus-adsovermqtt-xar-1.1.1.pkg`. Every package of this release goes into the same `All` folder, and the index covers them all.

## Step 10: publish the repository

Copy the folder from the box to the PC, in PowerShell on the PC:

```powershell
scp -r <user>@<box>:repo/tcbsd15 .
```

Remember the full path of the copied folder:

```powershell
$root = (Resolve-Path .\tcbsd15).Path
```

Upload every file in it to the raw repository, at the same path below `pkg/tcbsd15`:

```powershell
Get-ChildItem $root -Recurse -File | ForEach-Object {
    $rel = $_.FullName.Substring($root.Length + 1) -replace '\\', '/'
    Invoke-WebRequest -Uri "https://nexus.example.com/repository/tcpkg-raw/pkg/tcbsd15/$rel" -Method Put -InFile $_.FullName -Headers $auth
}
```

The index files are replaced with every release, which the raw repository's *Allow redeploy* permits. To check, open `https://nexus.example.com/repository/tcpkg-raw/pkg/tcbsd15/packagesite.pkg` without a login; the browser downloads it.

## A new version

Raise `version` in `meta/+MANIFEST`, then repeat step 6 with the new file name, and steps 9 and 10. Controllers pick it up as described on the [client page](tcbsd-clients.md#upgrading-and-removing).

Next: [client package usage on TwinCAT/BSD](tcbsd-clients.md).
