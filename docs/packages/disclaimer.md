# A disclaimer for your feed

Add the Beckhoff USA Community feed to a PC and, before the feed is registered, the Package Manager shows a page of text and asks you to accept it: what the feed is, that its packages are unsigned, that they come as they are. Beckhoff's own testing feed does the same. The text comes from a package on the feed, a **disclaimer package**, and this page shows how to build one for your feed.

A disclaimer package is a close cousin of a [workload](workloads.md): it installs nothing and exists for its metadata. The difference is its package type. When a feed is added, the Package Manager searches it for a package of type `Disclaimer` before it does anything else. If it finds one, the feed is added only after the user has accepted the text.

## What the user sees

In the GUI, adding the feed opens the text, rendered as Markdown, with a button to accept it. In a terminal, `tcpkg source add` prints "You need to accept the disclaimer to use this feed." and then shows the text one line at a time: Enter shows the next line, Y accepts, N declines. Declined, the feed is not added.

A script can accept the disclaimer ahead of time with the accept string the package carries:

```powershell
tcpkg source add -n MyFeed -s https://nexus.example.com/repository/tcpkg-raw/feeds/myfeed/index.json --accept-string AcceptMyCompany1.0.0
```

`tcpkg source add --help` does not list `--accept-string` in 2.6.24, but the command takes it, and it is what the GUI itself runs once the user has clicked accept. The next section shows where the string comes from.

## The folder

The disclaimer package of the Beckhoff USA Community feed, as it is kept in the public [BAUS_Community_General_Assets](https://github.com/Beckhoff-USA-Community/BAUS_Community_General_Assets/tree/main/disclaimer-package) repository:

```
Beckhoff-USA-Community.Disclaimer\
  Beckhoff-USA-Community.Disclaimer.nuspec
  disclaimer\
    disclaimer.txt
```

No `tools` folder and no scripts: the package is never installed, so nothing runs. `disclaimer.txt` is the text the user reads.

## The nuspec

```xml
<?xml version="1.0" encoding="utf-8"?>
<package xmlns="http://schemas.microsoft.com/packaging/2013/05/nuspec.xsd">
  <metadata>
    <id>Beckhoff-USA-Community.Disclaimer</id>
    <version>1.0.3</version>
    <title>Beckhoff USA Community Feed Disclaimer</title>
    <authors>Beckhoff USA Community</authors>
    <requireLicenseAcceptance>true</requireLicenseAcceptance>
    <licenseUrl>https://raw.githubusercontent.com/Beckhoff-USA-Community/BAUS_Community_General_Assets/main/disclaimer-package/disclaimer/disclaimer.txt</licenseUrl>
    <projectUrl>https://github.com/Beckhoff-USA-Community</projectUrl>
    <description>Disclaimer for the Beckhoff USA Community package feed</description>
    <copyright>(c) Beckhoff Automation LLC</copyright>
    <tags>disclaimer AS:AcceptCommunity1.0.3:AS</tags>
    <packageTypes>
      <packageType name="disclaimer" />
    </packageTypes>
  </metadata>
</package>
```

What matters in it:

`<packageType name="disclaimer" />`
:   This is what makes the Package Manager look at the package when the feed is added. Case does not matter: the package writes `disclaimer`, the Package Manager searches for `Disclaimer`, and the package is found. Put one disclaimer package on a feed, not two.

`<tags>disclaimer AS:AcceptCommunity1.0.3:AS</tags>`
:   The accept string sits between `AS:` and `:AS`. `--accept-string AcceptCommunity1.0.3` on `tcpkg source add` is what the GUI runs after the user clicks accept, and what a script passes to add the feed without the prompt. The version is part of the string on purpose: a new disclaimer is a new acceptance. Beckhoff's testing feed follows the same pattern, `Beckhoff.Disclaimer.BetaFeed` 2.0.1 with the accept string `AcceptBeta2.0.1`. Changing the string breaks every script that pins the old one, so change it only with a new version of the text, and tell the people who run such scripts.

`requireLicenseAcceptance` and `licenseUrl`
:   Where the text comes from when the feed is on a server. A NuGet server hands the Package Manager the nuspec metadata only and never a file out of the package, the same as for [icons and release notes](../hosting/index.md#inside-the-package-or-hosted-next-to-the-feed). So `licenseUrl` points at a copy of `disclaimer.txt` at a plain web address, here the raw address of the file in the GitHub repository. The address has to answer without a login; for a GitHub repository that means the repository stays public, and renaming it breaks the address. On Nexus, the raw repository next to the icons is the natural place; that was not tried for this guide.

`disclaimer\disclaimer.txt` inside the package
:   Where the text comes from when the feed is a folder on disk. The Package Manager reads the file out of the package, the way it reads a workload's icon from a folder feed. Keep the file in the package and the hosted copy both, with the same text, and the disclaimer shows from either kind of feed. Keep the folder and file name as they are.

No `<files>` section
:   Without one, `tcpkg pack` puts every file in the folder into the package, so `disclaimer\disclaimer.txt` goes in at that path. Checked with `tcpkg` 2.6.24 on a copy of this nuspec.

The package is listed with the others. `tcpkg list -n MyFeed` shows it as `MyCompany.Disclaimer 1.0.0` among the regular packages, and `-t Disclaimer` is not a filter `tcpkg list` knows in 2.6.24. Nobody installs it; it has no scripts to run.

## The text

The GUI renders the text as Markdown, and the terminal prints it line by line. Plain paragraphs, headings underlined with `=` or `-`, and short lines read well in both. No lists or tables.

What belongs in it is what a user has to know before the feed is on their PC:

* who runs the feed, and that it is not Beckhoff's own;
* that the packages are unsigned, so [signature verification](../hosting/signatures.md) has to be off, with the command;
* that the packages come as they are, without warranty;
* that every package has its own license, and where to find it;
* where to ask for help.

The [community disclaimer](https://github.com/Beckhoff-USA-Community/BAUS_Community_General_Assets/blob/main/disclaimer-package/disclaimer/disclaimer.txt) covers those in about sixty lines.

## Publishing and changing

Pack it and push it like any other package; the [packing](packing.md) and [hosting](../hosting/index.md) pages apply. The community repository packs and pushes its disclaimer from a GitHub Actions workflow whenever the package folder changes on `main`, with `--skip-duplicate` so that a version that is already on the feed is left alone.

A version on a feed is [never reused](versioning.md#a-version-is-never-reused), and that holds here too. To change the text, raise `<version>`, change the accept string to match, put the new text at the hosted address, and push the new package. PCs that already have the feed are not asked again; the disclaimer is checked when a feed is added.

## Before you share a disclaimer package

* `<packageTypes>` has `<packageType name="disclaimer" />`, and it is the only disclaimer package on the feed.
* The tags carry one accept string, `AS:...:AS`, and the version in it matches `<version>`.
* `requireLicenseAcceptance` is `true` and `licenseUrl` answers without a login, with the same text as the file in the package.
* `disclaimer\disclaimer.txt` is in the package, at that path.
* The text names the feed's owner, the signature setting, the warranty terms, the licenses, and where to get help.
* Whoever adds the feed from a script has the accept string.

Next: the examples, starting with a [PLC library](../examples/plc-library.md).
