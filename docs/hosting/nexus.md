# Nexus

Sonatype Nexus Repository is a server that hosts package repositories of many formats, NuGet and plain files among them. The free Community edition does everything this page needs. One Nexus serves your packages and the files the Package Manager GUI wants next to them, so that icons and release notes show from a server the way they do from a folder.

What is described here was done on Nexus 3.97 Community behind a reverse proxy that provides HTTPS. Nexus itself listens on plain HTTP, port 8081; put a proxy with a certificate in front of it.

The page sets the server up once, publishes a package, and then registers the feed on a PC. The names are placeholders: `https://nexus.example.com` for the server, `nuget-hosted` and `tcpkg-raw` for the two repositories, and *MyFeed* for the feed.

## Security settings

Two settings under *Administration*, *Security*:

**Anonymous Access** on. The Package Manager and its GUI read the feed without a login, and the default anonymous role already grants read access to every repository. Writing still needs a login. On a company network that nobody outside reaches, that is the right split: the network is the gate.

**Realms**: add the *NuGet API-Key Realm* to the active realms. Without it, the API key a user generates is not accepted for a push.

Then, from your own user menu, *NuGet API Key*, generate the key. That key publishes packages. A Nexus login with write access to the raw repository uploads everything else.

## Two repositories

Create both in the Nexus administration under *Repositories*, *Create repository*:

| Name | Recipe | Holds |
|---|---|---|
| `nuget-hosted` | nuget (hosted) | the `.nupkg` files |
| `tcpkg-raw` | raw (hosted) | workload icons, release notes, and the feed's own copy of the service index |

**nuget-hosted.** Leave the defaults. The deployment policy *Disable redeploy* is what keeps a published version from being replaced, which is what you want. The service index of the repository is `https://nexus.example.com/repository/nuget-hosted/index.json`.

**tcpkg-raw.** Three settings differ from the defaults:

* **Deployment policy: Allow redeploy.** The copy of the service index is uploaded again when it changes.
* **Strict content type validation: off.** Otherwise Nexus refuses an upload whose content does not match what it expects from the file extension, and `.md` and `.json` files trip it.
* **Content disposition: inline.** A browser then shows a file instead of downloading it, which helps when you check an address by hand.

## Publishing a package

A package goes to `nuget-hosted` with the NuGet push protocol. `dotnet nuget push` speaks it:

```powershell
dotnet nuget push .\MyCompany.XAE.Docs.MyMachine.1.0.0.nupkg `
    --source https://nexus.example.com/repository/nuget-hosted/ `
    --api-key <NuGet API key> --skip-duplicate
```

* Push the component packages before the workloads, so that a workload is never on the feed before its parts.
* `--skip-duplicate` reports a version that is already there instead of failing. Nexus refuses to replace it anyway. You can delete a version in the Nexus web UI and push it again; do that only for a mistake nobody has installed yet.
* Use `dotnet nuget push`, not `tcpkg push`: `tcpkg` writes every command line it runs, the API key included, into `C:\ProgramData\Beckhoff\TcPkg\logs\TcPkg.log`.

That is enough for the packages to install. What it does not give you are icons and release notes in the GUI, because a NuGet server hands the GUI only the nuspec metadata; see [inside the package, or hosted next to the feed](index.md#inside-the-package-or-hosted-next-to-the-feed). The next three sections put them on the raw repository.

## Uploading to the raw repository

A file goes into the raw repository with an HTTP `PUT` to the address it should have. Nexus answers a write without a login with 403 when anonymous access is on, not with the 401 challenge that PowerShell's `-Credential` waits for. So send the login with the first request, as a header:

```powershell
$cred = Get-Credential    # a Nexus login with write access to tcpkg-raw
$pair = "$($cred.UserName):$($cred.GetNetworkCredential().Password)"
$auth = @{ Authorization = 'Basic ' + [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($pair)) }

Invoke-WebRequest -Uri 'https://nexus.example.com/repository/tcpkg-raw/<path>' -Method Put -InFile <file> -Headers $auth
```

Every file the GUI reads from there it reads without a login. After an upload, open the address in a browser that is not logged in to Nexus. When a file shows up for you and not for a colleague, that is the check to make.

## The icons

A workload nuspec carries the icon twice: `<icon>` for a folder feed, and `<iconUrl>` with the address of a copy on the raw repository for a server:

```xml
<icon>icon.png</icon>
<iconUrl>https://nexus.example.com/repository/tcpkg-raw/icons/mycompany.mymachine.xae.png</iconUrl>
```

Upload the same `icon.png` to exactly that address:

```powershell
Invoke-WebRequest -Uri 'https://nexus.example.com/repository/tcpkg-raw/icons/mycompany.mymachine.xae.png' -Method Put -InFile .\MyCompany.MyMachine.XAE\icon.png -Headers $auth
```

Naming the file after the id in lower case keeps one icon per workload and makes the address easy to derive. Only workloads get a custom icon; a component package always shows the stock one.

## The release notes

The GUI fetches the release notes of a package from a server only when the feed's service index tells it where they are: a resource of type `ReadmeUriTemplate/6.13.0`, an address with the id and the version left open. Beckhoff's feed has one. Nexus's service index does not, so the feed gets a copy of the index with the resource added, on the raw repository, and is registered on that copy.

**1. Fetch Nexus's service index.**

```powershell
Invoke-WebRequest -Uri 'https://nexus.example.com/repository/nuget-hosted/index.json' -OutFile .\index.json
```

**2. Add the template.** Open `index.json` in a text editor and add one entry to the `resources` list. With Nexus's own entries shortened, the result looks like this:

```json
{
  "version": "3.0.0",
  "resources": [
    { "@id": "https://nexus.example.com/repository/nuget-hosted/query", "@type": "SearchQueryService" },
    { "@id": "https://nexus.example.com/repository/nuget-hosted/v3/registration/", "@type": "RegistrationsBaseUrl" },
    { "@id": "https://nexus.example.com/repository/nuget-hosted/", "@type": "PackagePublish/2.0.0" },
    { "@id": "https://nexus.example.com/repository/nuget-hosted/v3/content/", "@type": "PackageBaseAddress/3.0.0" },
    {
      "@id": "https://nexus.example.com/repository/tcpkg-raw/feeds/myfeed/readme/{lower_id}/{lower_version}/readme.md",
      "@type": "ReadmeUriTemplate/6.13.0",
      "comment": "Feature release notes per package version, hosted next to this index"
    }
  ]
}
```

Leave Nexus's entries as they are. They still point at `nuget-hosted`, so searches and downloads go to Nexus as before; the copy only adds the one resource.

**3. Upload the copy.**

```powershell
Invoke-WebRequest -Uri 'https://nexus.example.com/repository/tcpkg-raw/feeds/myfeed/index.json' -Method Put -InFile .\index.json -Headers $auth -ContentType 'application/json'
```

It needs uploading again only when Nexus's own index changes.

**4. Upload the release notes of every package version** to the address the template names, with the id and the version in lower case:

```powershell
Invoke-WebRequest -Uri 'https://nexus.example.com/repository/tcpkg-raw/feeds/myfeed/readme/mycompany.xae.docs.mymachine/1.0.0/readme.md' -Method Put -InFile .\MyCompany.XAE.Docs.MyMachine\MyCompany.XAE.Docs.MyMachine.ReleaseNote.md -Headers $auth
```

This one is part of every release: a new version without its file has an empty Feature Release Notes tab.

The raw repository ends up like this:

```
tcpkg-raw/
  icons/<lower id>.png                                   one per workload; <iconUrl> points here
  feeds/myfeed/index.json                                Nexus's service index plus the readme template
  feeds/myfeed/readme/<lower id>/<lower version>/readme.md   the release notes of each package version
```

This arrangement was worked out with `tcpkg` 2.4 against Nexus and holds for 2.6.

## Registering the feed

On every PC that should see the feed, register the copy of the index on the raw repository, not Nexus's own:

```powershell
tcpkg config unset -n VerifySignatures
tcpkg source add -n MyFeed -s https://nexus.example.com/repository/tcpkg-raw/feeds/myfeed/index.json --priority 2
tcpkg list -n MyFeed --no-cache
```

The first line is [signature verification](signatures.md). A feed registered on `nuget-hosted/index.json` directly works too, but shows no release notes. If `tcpkg source add` fails with "Failed to retrieve metadata from source", add `--take 100`; the Package Manager asks for 500 results per page by default, and some servers refuse that.

Open the GUI, pick *MyFeed* in the feed selector, and the cards are there.

## After a release

The Package Manager caches feed metadata for about 30 minutes. A new version shows up on other PCs after that, or at once with `tcpkg list -n MyFeed --no-cache` or *Reload Information* in the GUI.

Then check, from a PC that is not the one you published from:

* `tcpkg list -n MyFeed --no-cache` lists the new version;
* the icon address and the release notes address open without a login;
* `tcpkg install <Id> -n MyFeed --dry-run -y` resolves.

When all of that holds, the release is out.
