# Publishing a release

Three scripts, run in order from `packaging\scripts`, with the Package Manager GUI closed:

```powershell
.\Build-Packages.ps1 -Clean    # lint + pack into packaging\feed
.\Publish-Nexus.ps1            # push to nuget-hosted, then icons, release notes and index.json to tcpkg-raw
.\Test-Feed.ps1                # listed? icon and release notes reachable? install dry run
```

The rest of this page is what those scripts do and why, so that you can do it by hand on another server or fix it when it breaks.

## Pushing packages

A `.nupkg` goes to the NuGet repository with the NuGet push protocol. `dotnet nuget push` speaks it, and so does `tcpkg push`:

```powershell
dotnet nuget push .\packaging\feed\BAUS.Development.OEMCertificate.1.2.0.nupkg `
    --source https://nexus.example.com/repository/nuget-hosted/ `
    --api-key $env:IRIS_NEXUS_APIKEY --skip-duplicate
```

`Publish-Nexus.ps1` does this one file at a time, component packages first and workloads last, so a workload is never on the feed before its parts. Two choices in it are deliberate:

* It uses `dotnet nuget push` rather than `tcpkg push`, because `tcpkg` writes every command line it runs, including `-k <api key>`, into `C:\ProgramData\Beckhoff\TcPkg\logs\TcPkg.log`. `-UseTcpkg` switches when there is no .NET SDK on the PC.
* It pushes with `--skip-duplicate`, so a version that is already there is reported and skipped instead of failing the run. Nexus treats versions as immutable. To re-push a fixed build of the same version on your own feed, `-Replace` deletes the version first; do that only for a version nobody has installed yet.

The push target is the repository root. The script reads it from the service index (`PackagePublish/2.0.0` resource) rather than hard coding it, which also makes it work against the hosted copy of the index.

## Why the raw repository exists

This is the part that is not obvious and cost the most time. It was verified with `tcpkg` 2.4 against Nexus and holds for 2.6.

When the Package Manager reads a feed from a NuGet server, the server hands it the nuspec metadata: id, version, title, description, tags, dependencies. It does not hand over any file from inside the package. Two things the GUI shows live inside the package:

* the workload's `icon.png`
* the release notes file named by `<readme>`

On Beckhoff's own feed both show up, because Beckhoff's server does two things Nexus does not: it serves the icons at the URL in `<iconUrl>`, and its service index advertises a resource of type `ReadmeUriTemplate/6.13.0`, a URL template the GUI fills with the package id and version to fetch the release notes. Without that resource in the index, the Feature Release Notes tab is simply empty.

So the raw repository holds a hand made copy of what Beckhoff's server would serve:

```
tcpkg-raw/
  icons/<lower id>.png                          the 64 by 64 icon of each workload; <iconUrl> points here
  feeds/myfeed/index.json                       Nexus's own service index plus the readme template
  feeds/myfeed/readme/<lower id>/<version>/readme.md    the release notes file of each package version
  apt/nexus-apt.asc                             the public apt signing key, see the apt page
```

And the feed on every PC is registered on `feeds/myfeed/index.json`, the hosted copy, not on Nexus's `index.json`. Package downloads still come from `nuget-hosted`, because the hosted index lists the same resources as Nexus's; it only adds one.

The hosted `index.json` looks like this, with Nexus's resources shortened:

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

`Publish-FeedIndex.ps1` fetches Nexus's index, appends the template, writes everything into `packaging\feed-index\` with the same relative paths, and with `-Upload` puts each file into the raw repository with an HTTP `PUT`:

```powershell
$auth = @{ Authorization = 'Basic ' + [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("admin:$env:IRIS_NEXUS_PASSWORD")) }
Invoke-WebRequest -Uri 'https://nexus.example.com/repository/tcpkg-raw/feeds/myfeed/index.json' -Method Put -InFile .\feed-index\feeds\myfeed\index.json -Headers $auth -ContentType 'application/json'
```

`Publish-Nexus.ps1` calls it after every push. Afterwards the script fetches the hosted index anonymously, because that is how the GUI will fetch it.

The icon URL in a workload nuspec has to match where the file lands: `https://nexus.example.com/repository/tcpkg-raw/icons/<id in lower case>.png`. The build lint checks that against `feed.psd1`, so the two cannot drift.

## After the push

`tcpkg` caches feed metadata for about 30 minutes. `Publish-Nexus.ps1` deletes the cache folders for the feed on the publishing PC; on every other PC the new version shows up after the cache expires, after `tcpkg list -n MyFeed --no-cache`, or after *Reload Information* in the GUI.

Then run `Test-Feed.ps1`. It is read-only and checks, for every package in the repository, what a user's GUI will ask for: is the version listed, does the icon URL answer 200 without a login, does the release notes URL, does an install dry run resolve. When all four are green on a PC that is not the publishing PC, the release is out.

## Changing a published package

Bump the version, build, publish. On the feed the old version stays, and a PC that has it installed sees an update. That is the whole release model, and the Package Manager's *Updates* tab is built on it.

When a workload's component changes, the workload changes too: it pins the component's version, so the pin moves and the workload gets a new version of its own. Publish both, component first.
