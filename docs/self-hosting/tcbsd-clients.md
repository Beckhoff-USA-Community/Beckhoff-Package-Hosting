# Client package usage on TwinCAT/BSD

This page sets up a TwinCAT/BSD controller to install from your package repository on Nexus, and installs the package that [package building for TwinCAT/BSD](tcbsd-building.md) published. Steps 1 to 4 are done once per controller. After that, your packages install and upgrade with `pkg`.

Your repository sits next to Beckhoff's, not in place of it. The box's own configuration is `/etc/pkg/TCBSD.conf`; yours goes into `/usr/local/etc/pkg/repos/` with a fingerprint folder of its own, so the two never touch. Run every command in a root shell: `doas sh` first.

## Step 1: find the release

The repository has one folder per TwinCAT/BSD release, and the box can only use the one for its own. Print the box's ABI:

```sh
pkg config ABI
```

`FreeBSD:15:amd64` means the `tcbsd15` folder, `FreeBSD:14:amd64` the `tcbsd14` folder. The steps below use 15.

## Step 2: trust your signing key

`pkg` checks the repository's signature against a fingerprint you give it. Make a folder for it:

```sh
mkdir -p /usr/local/etc/pkg/fingerprints/myfeed/trusted
```

Write the fingerprint file. The value is the one printed in step 7 of the building page:

```sh
cat > /usr/local/etc/pkg/fingerprints/myfeed/trusted/myfeed <<'EOF'
function: sha256
fingerprint: <the fingerprint of repo.pub>
EOF
```

## Step 3: add the repository

Make the folder for your repository configuration:

```sh
mkdir -p /usr/local/etc/pkg/repos
```

Write the configuration. The `url` is the folder for this box's release, and `fingerprints` is the folder from step 2:

```sh
cat > /usr/local/etc/pkg/repos/MyFeed.conf <<'EOF'
MyFeed: {
  url: "https://nexus.example.com/repository/tcpkg-raw/pkg/tcbsd15",
  enabled: yes,
  signature_type: "fingerprints",
  fingerprints: "/usr/local/etc/pkg/fingerprints/myfeed"
}
EOF
```

## Step 4: read the repository

Fetch the index of your repository and check its signature:

```sh
pkg update -r MyFeed
```

If `pkg` answers "No signature found", the repository was signed with `rsa:` instead of the signing script; see step 8 of the building page. To see what the repository offers this box:

```sh
pkg search -r MyFeed baus
```

## Step 5: install the package

```sh
pkg install -y -r MyFeed baus-adsovermqtt-xar
```

`pkg` installs the dependencies first if they are missing. The package's script asks TwinCAT for a restart over ADS, and the route comes up within a few seconds.

## Step 6: check it

The route file is in the route folder:

```sh
ls -l /usr/local/etc/TwinCAT/3.1/Target/Routes/
```

The router is connected to the broker when this shows a `TcSystemService` line:

```sh
sockstat -4 -c | grep 1883
```

`service TcSystemService status` says "not running" to a user who is not root, because it cannot read the pidfile. In a root shell, or with `ps`, you see the truth.

## Upgrading and removing

A new version arrives the same way as the first one. Read the repository again:

```sh
pkg update -r MyFeed
```

Then install the package again; `pkg` upgrades it to the newest version:

```sh
pkg install -y -r MyFeed baus-adsovermqtt-xar
```

A plain `pkg upgrade` would also upgrade TwinCAT from Beckhoff's repository. That is a decision of its own. If the box still shows an older version, force a fresh index:

```sh
pkg update -f -r MyFeed
```

To remove the package, and with it the route:

```sh
pkg delete -y baus-adsovermqtt-xar
```

## The firewall

The controller connects out to the MQTT broker. The default `/etc/pf.conf` on TwinCAT/BSD passes all outgoing traffic and blocks incoming, so the route needs no firewall change.

If someone has tightened `pf.conf`, or the controller must also accept connections on the MQTT ports, add these lines to `/etc/pf.conf`, before the `anchor bhf` line:

```
pass out quick proto tcp to port { 1883, 8883 }
pass in  quick proto tcp to port { 1883, 8883 }
```

Then reload the firewall:

```sh
service pf reload
```

## A certificate from an in-house authority

`pkg` checks the server's HTTPS certificate. One from a public authority needs nothing. For an in-house one, copy the authority's certificate to the box, with the extension `.pem`:

```sh
cp myca.pem /usr/local/etc/ssl/certs/
```

Then add it to the box's trusted certificates:

```sh
certctl rehash
```
