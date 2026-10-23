# GitHub Packages

GitHub runs a NuGet registry for every account and organization, GitHub Packages, and the TwinCAT Package Manager can use it as a feed. That works with the Package Manager releases since May 2026. It is a good home for packages whose source already lives on GitHub; the [Beckhoff USA Community](https://github.com/Beckhoff-USA-Community) projects publish theirs there. Two things are needed that a folder feed does not need, a token and the `--take` option, and the GUI shows less than from other feeds; [the last section](#what-works-and-what-does-not) says what.

The feed address is fixed by GitHub, with your account or organization name in it:

```
https://nuget.pkg.github.com/MyCompany/index.json
```

## A token, even to read

GitHub asks for a login on its package feeds, even for public packages; the address above answers 401 to a request without one. The login is your GitHub user name and a personal access token, and it has to be a **classic** token. Create one in your GitHub account under *Settings*, *Developer settings*, *Personal access tokens*, *Tokens (classic)*, with the scope:

* `read:packages` for every PC that installs from the feed;
* `write:packages` as well for the PC or build job that publishes.

Treat the token like a password. Whoever has it can read, or with `write:packages` publish to, your packages.

## Adding the feed

From an administrator PowerShell:

```powershell
tcpkg config unset -n VerifySignatures
tcpkg source add -n MyCompany -s https://nuget.pkg.github.com/MyCompany/index.json -u <your GitHub user name> --take 100
```

`tcpkg` asks for a password; paste the token. If the feed carries a [disclaimer](../packages/disclaimer.md), `tcpkg` shows it next and asks you to accept it; the Beckhoff USA Community feed has one. Both settings are required:

* **[Signature verification off](signatures.md)**, before the feed is added.
* **`--take 100`.** The Package Manager asks a feed for 500 results per page by default, and GitHub refuses pages larger than 100. Without the option, adding the feed fails with "Failed to retrieve metadata from source". For a feed that was added without it, `tcpkg source edit --help` offers the same option: `tcpkg source edit MyCompany --take 100`.

Then check what the feed serves:

```powershell
tcpkg list -n MyCompany --no-cache
```

## Publishing

`dotnet nuget push` uploads a package to GitHub, with a token that has `write:packages`:

```powershell
dotnet nuget push .\MyCompany.XAE.Docs.MyMachine.1.0.0.nupkg `
    --source https://nuget.pkg.github.com/MyCompany/index.json `
    --api-key <token> --skip-duplicate
```

`--skip-duplicate` reports a version that is already there instead of failing. Push the component packages before the workloads, so that a workload is never on the feed before its parts. A GitHub Actions workflow can run the same command with the workflow's own token; the community projects publish that way when a release is tagged.

Use `dotnet nuget push`, not `tcpkg push`: `tcpkg` writes every command line it runs, the token included, into `C:\ProgramData\Beckhoff\TcPkg\logs\TcPkg.log`.

## What works, and what does not

Installing, upgrading and uninstalling work as from any feed, and so do the category and the Engineering and Runtime rows that come from the [workload tags](../packages/workloads.md). The GUI shows less than it does from a folder feed or from Nexus:

![The Package Manager showing a GitHub feed: separate cards titled with package ids such as Beckhoff-USA-Community.AdsGamepad.XAE and .XAR](../images/feed-github.png)

* **The card shows the id, not the title.** The ADS Gamepad workloads have `<title>ADS Gamepad</title>` in their nuspecs; from GitHub the cards read `Beckhoff-USA-Community.AdsGamepad.XAE` and `...XAR`.
* **So the Engineering and Runtime workloads are two cards.** The GUI draws one card for workloads with the same title, and from GitHub it does not get the title. Each workload still works; they just sit side by side.
* **Stock icons and no release notes.** GitHub serves the nuspec metadata only, like any NuGet server; see [inside the package, or hosted next to the feed](index.md#inside-the-package-or-hosted-next-to-the-feed). An `<iconUrl>` was not tried with GitHub. Pointing the GUI at hosted release notes needs a service index of your own, and with GitHub's login in the way that was not tried either.

Since the id is what the user reads on GitHub, pick ids that name the product plainly.

Next: [Nexus](nexus.md), for a feed on your own network where the GUI shows everything.
