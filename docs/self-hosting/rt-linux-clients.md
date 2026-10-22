# Client package usage on Beckhoff RT Linux

This page sets up a Beckhoff RT Linux controller to install from your apt repository on Nexus, and installs the package that [package building for Beckhoff RT Linux](rt-linux-building.md) published. Steps 1 to 3 are done once per controller. After that, your packages install and upgrade with `apt` like any other.

The shape is the same as Beckhoff's own repository entry on the controller: a key file, and a source line that names it. Run the commands as a user with `sudo`.

## Step 1: the repository's public key

`apt` trusts a repository only when it can check the signature with the repository's public key. Download it from the raw repository into the folder Debian keeps for such keys:

```sh
sudo curl -fsSL https://nexus.example.com/repository/tcpkg-raw/apt/myfeed-apt.asc -o /etc/apt/keyrings/myfeed-apt.asc
```

## Step 2: the repository

Add one source line. `signed-by` trusts the key for this repository only, `trixie` is the distribution the repository was created with, and `main` is the component Nexus puts every package in:

```sh
echo "deb [signed-by=/etc/apt/keyrings/myfeed-apt.asc] https://nexus.example.com/repository/apt-hosted trixie main" | sudo tee /etc/apt/sources.list.d/myfeed.list
```

## Step 3: read the repository

Fetch the package lists, yours included:

```sh
sudo apt update
```

If it fails with "Sub-process /usr/bin/sqv returned an error code" and "The repository ... is not signed", the key file from step 1 is missing or does not match the key in the repository. The repository itself is fine.

## Step 4: install the package

```sh
sudo apt install baus-adsovermqtt-xar
```

`apt` installs `tc31-xar-um` first if it is missing. If the engineering variant is installed, `apt` offers to remove it, because the two packages conflict; it asks before it does, and `-y` answers yes for you. The package's script restarts `TcSystemServiceUm`, and the route comes up within a few seconds.

## Step 5: check it

The route file is in the route folder:

```sh
ls -l /etc/TwinCAT/3.1/Target/Routes/
```

The installed version, and the repository it came from:

```sh
apt policy baus-adsovermqtt-xar
```

## Upgrading and removing

A new version arrives the same way as the first one. Read the repository again:

```sh
sudo apt update
```

Then install the package again; `apt` upgrades it to the newest version:

```sh
sudo apt install baus-adsovermqtt-xar
```

A plain `sudo apt upgrade` picks up your new version as well, but it also upgrades TwinCAT from Beckhoff's repository. That is a decision of its own.

To remove the package, and with it the route:

```sh
sudo apt remove baus-adsovermqtt-xar
```

## When your server needs more

Two things the steps above do not cover, because the [Nexus page](../hosting/nexus.md) sets the server up without them.

**A certificate from an in-house authority.** `apt` checks the server's HTTPS certificate. One from a public authority needs nothing. For an in-house one, copy the authority's certificate to the controller, with the extension `.crt`:

```sh
sudo cp myca.crt /usr/local/share/ca-certificates/
```

Then add it to the controller's trusted certificates:

```sh
sudo update-ca-certificates
```

**A login for reading.** With anonymous access off, `apt` needs a login for the server. It reads logins from `/etc/apt/auth.conf.d/`, the way Beckhoff's own repository uses `bhf.conf`. Create the file:

```sh
printf 'machine nexus.example.com\nlogin <user>\npassword <password>\n' | sudo tee /etc/apt/auth.conf.d/myfeed.conf > /dev/null
```

Make it readable by root only:

```sh
sudo chmod 600 /etc/apt/auth.conf.d/myfeed.conf
```

Give the controllers a Nexus user that can only read.
