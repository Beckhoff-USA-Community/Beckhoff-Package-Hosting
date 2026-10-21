# Serving it with apt

A `.deb` file copied by hand installs fine. An apt repository is what makes it one command on every controller, with upgrades through `apt upgrade` like everything else on the box. Nexus hosts apt repositories, so the Windows feed's server serves the Linux packages too. This page sets the repository up once, publishes to it, and adds it to a Beckhoff RT Linux controller.

## How apt trusts a repository

apt downloads an index of the repository and checks a signature on it before it trusts a single package. The repository owner signs the index with a private key; every client holds the public half and refuses an index that does not verify. Nexus does the signing: you give the repository the private key once, and Nexus signs the `Release` file every time the content changes. The clients need the public key, and that is the one part Nexus does not take care of.

## The signing key

Generate a key pair once, on any machine with GnuPG. A batch file keeps it unattended; the key has no passphrase, because Nexus has to use it without anyone typing one:

```
%no-protection
Key-Type: RSA
Key-Length: 4096
Name-Real: MyFeed apt repository
Name-Email: you@example.com
Expire-Date: 0
%commit
```

```sh
gpg --batch --gen-key keyparams.txt
gpg --armor --export-secret-keys "MyFeed apt repository" > apt-signing.private.asc
gpg --armor --export "MyFeed apt repository" > nexus-apt.asc
```

The private half goes to `packaging\secrets\apt-signing\` (git-ignored) and from there into the Nexus repository settings. The public half is committed as `packaging\deb\keyrings\nexus-apt.asc`; it is public by nature and the controllers will download it.

## The repository on Nexus

*Create repository*, *apt (hosted)*, with:

* **Distribution**: the Debian release name the clients ask for. Beckhoff RT Linux is Debian 13, so `trixie`.
* **Signing key**: the contents of the private key file. **Passphrase** empty.
* **Deployment policy**: *Disable redeploy*. apt clients cache by version, so a changed file under the same version would never reach a client that has already seen it. Every change bumps `Version:` in `control`.

`Setup-NexusApt.ps1` does this through the REST API with the key from the secrets folder and the distribution from `feed.psd1`:

```powershell
cd .\packaging\scripts
.\Setup-NexusApt.ps1
```

Then it does the one thing Nexus will not: it uploads the public key to the raw repository, at `https://nexus.example.com/repository/tcpkg-raw/apt/nexus-apt.asc`, and checks that an anonymous download works. Nexus Community 3.97 signs the repository but answers `repository-key.gpg` with 404, so the public key has to be served from somewhere, and the raw repository already serves files anonymously.

## Publishing

A `.deb` is uploaded through Nexus's components API, a multipart `POST` with the file in a field named `apt.asset`:

```powershell
$auth = @{ Authorization = 'Basic ' + [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("admin:$env:IRIS_NEXUS_PASSWORD")) }
Invoke-RestMethod -Uri 'https://nexus.example.com/service/rest/v1/components?repository=apt-hosted' -Method Post -Headers $auth `
    -Form @{ 'apt.asset' = Get-Item .\packaging\feed-deb\baus-adsovermqtt-xar_1.0.0_all.deb }
```

Nexus rebuilds the `Packages` and `Release` files and signs them. `Publish-Apt.ps1` uploads every file in `packaging\feed-deb`, skips a name and version that the repository already lists, and afterwards fetches `dists/trixie/InRelease` and the `Packages` list without a login, which is exactly what apt on a controller will do:

```powershell
.\Build-Deb.ps1 -Clean
.\Publish-Apt.ps1
```

The script ends by printing the three lines for a controller, with the real hostnames filled in from `feed.psd1`.

## Adding the repository on a controller

On the Beckhoff RT Linux box, as a user with sudo:

```sh
sudo curl -fsSL https://nexus.example.com/repository/tcpkg-raw/apt/nexus-apt.asc -o /etc/apt/keyrings/nexus-apt.asc
echo "deb [signed-by=/etc/apt/keyrings/nexus-apt.asc] https://nexus.example.com/repository/apt-hosted trixie main" | sudo tee /etc/apt/sources.list.d/nexus.list
sudo apt update
sudo apt install baus-adsovermqtt-xar
```

Line by line:

* The public key goes into `/etc/apt/keyrings/`, the folder Debian reserves for keys of third party repositories. The old `apt-key add` is gone from current Debian; this is the replacement.
* The source line names the key with `signed-by`, so this key is trusted for this repository only and not for every repository on the box. `trixie` is the distribution the repository was created with, `main` is the component Nexus puts everything under.
* `apt update` fetches `InRelease`, verifies the signature with that key, and reads the package list. A signature error here means the key file and the key in the repository settings do not match. A 404 means the distribution name does not match.
* `apt install` fetches the package, checks `Depends` and `Conflicts`, unpacks the file, runs `postinst`, and the router restarts.

Afterwards:

```sh
apt policy baus-adsovermqtt-xar        # installed and candidate versions, and which repository they come from
systemctl status TcSystemServiceUm
```

## Upgrading and removing

A new version is built with a bumped `Version:`, published, and every controller sees it on its next `apt update`:

```sh
sudo apt update
sudo apt upgrade                        # or: sudo apt install baus-adsovermqtt-xar
```

The upgrade unpacks the new file over the old one and runs the new `postinst`, so the router restarts once with the new route. Removing is `sudo apt remove baus-adsovermqtt-xar`: dpkg deletes the route file, `postrm` restarts the service, the route is gone.

The HTTPS certificate on the Nexus proxy has to be one the controller trusts, the same as for any website. A certificate from a public authority works out of the box; one from an in-house authority needs that authority's certificate under `/usr/local/share/ca-certificates/` followed by `sudo update-ca-certificates` before `apt update` will talk to the server.
