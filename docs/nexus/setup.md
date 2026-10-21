# Setting up Nexus

Sonatype Nexus Repository is a server that hosts package repositories of many formats: NuGet, apt, raw files, and others. The free Community edition does everything this guide needs. One Nexus serves the TwinCAT packages, the files the Package Manager GUI wants next to them, and the Debian packages for Beckhoff RT Linux, so it is the one server to run. This page is the one time setup; [publishing a release](publishing.md) is what happens every time after.

What is described here was done on Nexus 3.97 Community behind a reverse proxy that provides HTTPS. Nexus itself listens on plain HTTP on port 8081; put a proxy with a certificate in front of it, because the Package Manager and apt both want HTTPS without complaint.

## Security settings

Two settings under *Administration*, *Security*:

**Anonymous Access** on. The Package Manager, the GUI and apt all read without a login, and the default anonymous role already grants read on every repository. Writes still need a login. On a LAN that nobody outside reaches, that is the right split: the network is the gate, Nexus holds the files.

**Realms**: add the *NuGet API-Key Realm* to the active realms. Without it the API key a user generates under their user menu is not accepted on a push.

Then, from your own user menu, *NuGet API Key*, generate the key. That key pushes packages. The admin login, or a user with write on the raw repository, uploads everything else.

## Three repositories

| Name | Format | Holds | Read by |
|---|---|---|---|
| `nuget-hosted` | nuget (hosted) | the `.nupkg` files | the Package Manager |
| `tcpkg-raw` | raw (hosted) | workload icons, release notes, the feed's `index.json`, the apt signing key | the Package Manager GUI, apt clients |
| `apt-hosted` | apt (hosted) | the `.deb` files | apt on Beckhoff RT Linux |

### nuget-hosted

*Create repository*, *nuget (hosted)*. Nexus speaks NuGet V3 on a hosted repository out of the box; the service index the Package Manager reads is `https://nexus.example.com/repository/nuget-hosted/index.json`. The write policy stays *Disable redeploy*, which is what makes versions immutable, and that is what you want. Leave the defaults otherwise.

### tcpkg-raw

*Create repository*, *raw (hosted)*. Three settings differ from the defaults, and the script `Setup-NexusRaw.ps1` sets them through the REST API:

* **Deployment policy: Allow redeploy.** `index.json` and the release notes are re-uploaded on every release.
* **Strict content type validation: off.** Nexus otherwise refuses an upload whose content does not match what it guesses from the extension, and `.md` and `.json` files trip it.
* **Content disposition: inline.** A browser shows the file instead of downloading it, which is handy when you check a URL by hand.

```powershell
cd .\packaging\scripts
.\Setup-NexusRaw.ps1
```

The script creates the repository when it is missing, then uploads a tiny probe file, reads it back without a login, and deletes it. That read is the one that matters: the GUI fetches icons and release notes anonymously, and if the probe comes back with anything but 200, anonymous access is not set up the way the GUI needs.

### apt-hosted

The apt repository needs a signing key first, so it has its own section on [the apt page](../linux/apt.md). Skip it until you have a Debian package to serve.

## The credentials file

The scripts take their credentials from a git-ignored file, `packaging\secrets\Nexus-Secrets.ps1`, which `Common.ps1` loads when it exists:

```powershell
$env:IRIS_NEXUS_APIKEY   = ''        # NuGet API key: pushes and deletes packages
$env:IRIS_NEXUS_USER     = 'admin'   # a login that can write to tcpkg-raw and create repositories
$env:IRIS_NEXUS_PASSWORD = ''
```

Copy `Nexus-Secrets.example.ps1` to that name and fill it in. Nothing else in the repository needs to know the values.

One Nexus habit to know about: with anonymous access on, Nexus answers an unauthenticated write with 403 instead of a 401 challenge. PowerShell's `-Credential` parameter waits for the challenge and never sends the login, so every write in these scripts sends a Basic `Authorization` header up front. If you write your own upload, do the same.

## The feed on each engineering PC

With the repositories in place and a first release [published](publishing.md), every PC that should see the feed registers it once. The feed is registered on the hosted `index.json` in the raw repository, not on Nexus's own; the publishing page explains why, in short: only the hosted copy tells the GUI where the release notes are.

```powershell
tcpkg config unset -n VerifySignatures
tcpkg source add -n MyFeed -s https://nexus.example.com/repository/tcpkg-raw/feeds/myfeed/index.json --priority 2
tcpkg source list
tcpkg list -n MyFeed --no-cache
```

`Setup-Feed.ps1` does the same with the URL from `feed.psd1`, removes a source of the same name first, and lists what the feed serves afterwards. If `tcpkg source add` fails with "Failed to retrieve metadata from source", add `--take 100` (or `-Take 100` on the script); the Package Manager asks for 500 results per page and some servers refuse that.

Open the GUI, pick *MyFeed* in the feed selector, and the cards are there.

## Checking the server

`Test-Feed.ps1` is the read-only check to run from any PC with the feed registered. For every package folder in the repository it checks that the id and version are listed on the feed, that the icon and the release notes URLs answer 200 without a login, and that `tcpkg install --dry-run` resolves. Nothing is installed.
