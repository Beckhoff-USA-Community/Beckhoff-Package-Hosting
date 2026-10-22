# FreeBSD packages for TwinCAT/BSD

TwinCAT/BSD is FreeBSD with TwinCAT on top, and software on it comes from `pkg`. This page builds the ADS-over-MQTT route package as a FreeBSD package, puts a signed `pkg` repository on Nexus, and installs it on a controller.

Three things make it more work than the [Debian package](debian.md):

* **Nexus has no `pkg` format.** It does not need one. A `pkg` repository is a folder of static files, and the raw repository serves them as they are.
* **The repository index can only be built on FreeBSD.** `pkg repo` makes it, and `pkg` does not run on Windows. A TwinCAT/BSD box is part of publishing, not only of testing.
* **Two traps** cost hours if you do not know them: the [restart](#do-not-restart-the-service-from-a-script) and the [conflicts field](#no-conflicts-in-the-manifest).

Everything below runs as root on the TwinCAT/BSD box: `doas sh` first.

## One build per TwinCAT/BSD release

`pkg` refuses a package built for another FreeBSD major version. A package built with `abi: "FreeBSD:14:*"` is invisible to a TwinCAT/BSD 15 box, and the other way round. The content of the route package is the same for both, so build it once per release, and give each build its own repository folder, because the file names are the same:

| TwinCAT/BSD | FreeBSD | ABI | Repository folder | Dependencies as shipped |
|---|---|---|---|---|
| 14 | 14.4 | `FreeBSD:14:*` | `tcpkg-raw/pkg/tcbsd14` | `TC31-XAR` 4026.29.3_20, `TcAdsTool` 0.113.0.30 |
| 15 | 15.1 | `FreeBSD:15:*` | `tcpkg-raw/pkg/tcbsd15` | `TC31-XAR` 4026.28.21_21, `TcAdsTool` 0.113.0.30 |

That is how Beckhoff lays out its own repository, with `TCBSD/14/stable/packages` and `TCBSD/15/stable/packages`. `pkg` resolves dependencies by name, so the versions in the manifest are information only, but record the ones the release ships so `pkg info` does not mislead anyone. `pkg info | grep -i TC31` on a box shows them.

The rest of this page uses release 15. For 14, change the ABI, the dependency versions and the folder.

## The package

Four files and the route file:

```
baus-adsovermqtt-xar/
  manifest.json
  pre-install
  post-install
  post-deinstall
  AdsOverMqtt.xml
```

`AdsOverMqtt.xml` is the [route file](../examples/files.md#a-route-file), with the comment line on top changed to name this package. It installs to `/usr/local/etc/TwinCAT/3.1/Target/Routes/`. That folder does not exist until a route is placed there.

### The manifest

`pkg` reads UCL, and JSON is valid UCL:

```json
{
  "name": "baus-adsovermqtt-xar",
  "version": "1.1.1",
  "origin": "baus/baus-adsovermqtt-xar",
  "comment": "ADS-over-MQTT connection to the MQTT broker",
  "desc": "Installs the ADS-over-MQTT route file AdsOverMqtt.xml into /usr/local/etc/TwinCAT/3.1/Target/Routes and asks TwinCAT for a restart over ADS, so the ADS router joins the network at once.",
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

`TC31-XAR` owns the TwinCAT system service, `/usr/local/bin/TcSystemService`. `TcAdsTool` brings the `ads` command line tool, which the scripts need. There is no `conflicts` entry, on purpose.

### No `conflicts` in the manifest

A `conflicts` entry naming a package that is not installed makes `pkg add` fail with `sqlite error ... NOT NULL constraint failed: pkg_conflicts.conflict_id`. Conflicts in `pkg` are meant to be computed by `pkg repo` from overlapping files, not declared by hand.

So the variant check goes into `pre-install`. It looks for a route file of the other variant and stops the install when it finds one:

```sh
#!/bin/sh
# pre-install: the engineering and runtime variants carry different credentials and exclude each other.
# pkg cannot be asked what is installed while it holds its own database, so check the route folder.
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

`pkg` stops with "PRE-INSTALL script failed", which is the behaviour wanted. It finds the other variant by its comment line, the same marker the Windows scripts use.

### Do not restart the service from a script

This is the trap that cost the most time. `pkg` kills every process a maintainer script started when the script ends. A `service TcSystemService restart` in `post-install` starts the service, the script finishes, and `pkg` kills the new service together with the rest of the script's process tree. The box is left with no TwinCAT system service, the log shows no shutdown message, and `daemon -f` does not escape. The same `service` command typed over ssh works, which is what makes it confusing.

Ask the running service for a TwinCAT restart over ADS instead. That is what the *Restart TwinCAT* button in the engineering sends, and the `ads` tool can send it. `post-install`:

```sh
#!/bin/sh
# post-install: the router reads the route folder only when TwinCAT initialises. Ask the running
# system service for a restart over ADS; a service restart from here dies with this script.
ads=/usr/local/bin/ads
if [ -x "$ads" ]; then
    state=$("$ads" 127.0.0.1.1.1 state 2>/dev/null)          # 5 = RUN, 15 = CONFIG
    if [ "$state" = "5" ]; then cmd=2; else cmd=16; fi        # 2 = RESET, 16 = RECONFIG
    echo "ADS state $state: requesting a TwinCAT restart so the router picks up the ADS-over-MQTT route ..."
    "$ads" 127.0.0.1.1.1 state "$cmd" || echo "The ADS request failed; run: doas service TcSystemService restart" >&2
else
    echo "$ads not found (package TcAdsTool); run: doas service TcSystemService restart" >&2
fi
exit 0
```

RESET (2) restarts TwinCAT into RUN, so a boot project comes back up. RECONFIG (16) restarts it into CONFIG. Either way the ADS router reads the route folder again and connects to the broker within seconds, and the `TcSystemService` process stays the same one. Verified on TwinCAT/BSD 14 and 15 in CONFIG, with the route added and with it removed. The RUN branch was not tried, because no test box had a boot project; if it ever misbehaves, `doas service TcSystemService restart` typed by hand is the fallback the script prints.

`post-deinstall` is the same script with "drops" for "picks up": `pkg` has deleted the file, and the router has to forget the route.

### Tell the person at the terminal what is left

People install from a shell and read what it prints. The tested `post-install` ends by printing a note with the two things a person may still need: how to check the connection, and what to do about the firewall.

```
ADS-over-MQTT route installed: broker mqtt.example.com, port 1883 (8883 with TLS).

Check the connection (a TcSystemService line means the router is connected):
    doas sockstat -4 -c | grep 1883

Firewall (pf): this controller connects OUT to the broker. The TwinCAT/BSD default /etc/pf.conf
allows all outgoing traffic, so nothing needs to change. If pf.conf has been tightened, or this
controller must also ACCEPT connections on the MQTT ports, add these lines to /etc/pf.conf
(before the "anchor bhf" line) and reload pf:
    pass out quick proto tcp to port { 1883, 8883 }
    pass in  quick proto tcp to port { 1883, 8883 }
    doas service pf reload
```

The default `/etc/pf.conf` on TwinCAT/BSD has `block return in all` and `pass out quick all`. The controller is the MQTT client, so the route needs no firewall change unless someone tightened the rules.

Save all scripts with LF line endings. A shell script saved with Windows line endings fails on the controller.

## Building it

Copy the folder to the box, for example with `scp -r`. `pkg create` builds the package from three things: a metadata folder with the manifest as `+MANIFEST` and the scripts under their `pkg` names, a staging folder that looks like the target file system, and a list of the files:

```sh
cd baus-adsovermqtt-xar
mkdir -p meta stage/usr/local/etc/TwinCAT/3.1/Target/Routes out
cp manifest.json meta/+MANIFEST
cp pre-install meta/+PRE_INSTALL
cp post-install meta/+POST_INSTALL
cp post-deinstall meta/+POST_DEINSTALL
cp AdsOverMqtt.xml stage/usr/local/etc/TwinCAT/3.1/Target/Routes/
echo 'etc/TwinCAT/3.1/Target/Routes/AdsOverMqtt.xml' > plist
pkg create -m meta -r stage -p plist -o out
```

The paths in `plist` are relative to the manifest's `prefix`, `/usr/local`. The result is `out/baus-adsovermqtt-xar-1.1.1.pkg`. Look inside before you go on:

```sh
pkg info -F out/baus-adsovermqtt-xar-1.1.1.pkg       # name, version, ABI, dependencies
pkg info -l -F out/baus-adsovermqtt-xar-1.1.1.pkg    # the files
```

`pkg create` is FreeBSD's own tool for this and the one the ports use, but this sequence was not run for the guide. The tested package was written on Windows by a script that produces the same archive: a tar file whose first two members are `+COMPACT_MANIFEST` and `+MANIFEST`, with the scripts embedded in the manifest, followed by the files with their full paths. Check the result with `pkg info -F`, and with a test install, before you publish it.

## Testing it on the box

```sh
pgrep -l TcSystemService                         # note the process ID
pkg add out/baus-adsovermqtt-xar-1.1.1.pkg
ls -l /usr/local/etc/TwinCAT/3.1/Target/Routes/
sockstat -4 -c | grep 1883                        # the router's connection to the broker
pgrep -l TcSystemService                         # the same process ID as before
pkg delete -y baus-adsovermqtt-xar
```

The process ID is the check that matters on TwinCAT/BSD: the same number before and after means TwinCAT restarted over ADS and the system service survived. `service TcSystemService status` says "not running" to a user who is not root, because it cannot read the pidfile; `ps` and `pgrep` are the honest check.

## Signing the repository

Beckhoff's own repository uses `signature_type: "fingerprints"`, and so should yours. `pkg repo` can sign in two ways, and only one of them works for a fingerprint client:

* `pkg repo <folder> rsa:<keyfile>` signs the index but embeds only the signature. A fingerprint client answers `pkg: No signature found` and refuses the repository. This form is for clients configured with `signature_type: "pubkey"` and a local copy of the public key.
* `pkg repo <folder> signing_command: sh sign.sh` runs a script that gets the index's checksum on stdin and prints the signature and the public key. The index then carries both, and fingerprint verification works.

Make an RSA key once, on the box or on any system with OpenSSL:

```sh
openssl genrsa -out repo.key 2048
openssl rsa -in repo.key -pubout -out repo.pub
sha256 -q repo.pub
```

The last line prints the fingerprint the controllers will check against. `sha256 -q` is FreeBSD's command; `sha256sum repo.pub` on Linux prints the same value. Keep `repo.key` somewhere safe and out of source control. It needs to be on the box only while `pkg repo` runs.

The signing script, from the FreeBSD handbook. It expects the key pair in `/tmp`; copy `repo.key` and `repo.pub` there for the run, next to the script as `/tmp/sign.sh`:

```sh
#!/bin/sh
read -t 2 sum
[ -z "$sum" ] && exit 1
echo SIGNATURE
echo -n "$sum" | openssl dgst -sign /tmp/repo.key -sha256 -binary
echo
echo CERT
cat /tmp/repo.pub
echo END
```

Two `pkg` 2.x details: a bare key path is no longer accepted, so the `rsa:` prefix or `signing_command:` is required, and `pkg help repo` opens a pager, which hangs a session over ssh that has no terminal.

## Building the repository

On the box, one folder per release, with the packages under `All`:

```sh
mkdir -p repo/tcbsd15/All
cp out/baus-adsovermqtt-xar-1.1.1.pkg repo/tcbsd15/All/
chmod 600 /tmp/repo.key
pkg repo -o repo/tcbsd15 repo/tcbsd15 signing_command: sh /tmp/sign.sh
rm /tmp/repo.key /tmp/repo.pub /tmp/sign.sh
```

`pkg repo` writes the index next to the packages: `meta.conf`, `meta`, `packagesite.pkg` and `data.pkg`. A box of either release can build the index for both folders; it reads only the manifests.

## Publishing

Copy the folder back to the PC, for example with `scp -r`, and upload every file to the raw repository at the same relative path, with the `$auth` header from [uploading to the raw repository](../hosting/nexus.md#uploading-to-the-raw-repository):

```powershell
$root = (Resolve-Path .\repo\tcbsd15).Path
Get-ChildItem $root -Recurse -File | ForEach-Object {
    $rel = $_.FullName.Substring($root.Length + 1) -replace '\\', '/'
    Invoke-WebRequest -Uri "https://nexus.example.com/repository/tcpkg-raw/pkg/tcbsd15/$rel" -Method Put -InFile $_.FullName -Headers $auth
}
```

The index files are replaced on every release, which the raw repository's *Allow redeploy* permits. The package files carry their version in the name, so a new version never replaces an old one. Then open `https://nexus.example.com/repository/tcpkg-raw/pkg/tcbsd15/packagesite.pkg` without a login.

## Setting up a controller

As root (`doas sh`) on a TwinCAT/BSD 15 box; a 14 box uses the `tcbsd14` folder:

```sh
mkdir -p /usr/local/etc/pkg/fingerprints/myfeed/trusted /usr/local/etc/pkg/repos
printf '%s\n' 'function: sha256' 'fingerprint: <the output of sha256 -q repo.pub>' > /usr/local/etc/pkg/fingerprints/myfeed/trusted/myfeed
printf '%s\n' 'MyFeed: {' '  url: "https://nexus.example.com/repository/tcpkg-raw/pkg/tcbsd15",' '  enabled: yes,' '  signature_type: "fingerprints",' '  fingerprints: "/usr/local/etc/pkg/fingerprints/myfeed"' '}' > /usr/local/etc/pkg/repos/MyFeed.conf
pkg update -r MyFeed && pkg install -y -r MyFeed baus-adsovermqtt-xar
```

* The fingerprint file says which key this repository's index must be signed with. It goes into a `trusted` folder of its own, so it trusts nothing else.
* The box's own repository configuration is `/etc/pkg/TCBSD.conf`. Yours goes into `/usr/local/etc/pkg/repos/` with its own fingerprint folder, so the two never touch.
* `pkg search -r MyFeed baus` shows what the repository offers for this box's ABI. If a box has fetched an older index, `pkg update -f -r MyFeed` forces a fresh one.

A new version reaches the controller with the same two commands, `pkg update -r MyFeed` and `pkg install -r MyFeed baus-adsovermqtt-xar`. A plain `pkg upgrade` would also upgrade TwinCAT from Beckhoff's repository, which is a decision of its own. `pkg delete baus-adsovermqtt-xar` removes the package again.
