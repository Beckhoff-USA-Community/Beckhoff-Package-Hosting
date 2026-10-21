# Beckhoff Package Hosting

The TwinCAT Packaging Guide: how to build TwinCAT Package Manager packages and workloads by hand, and how to
host them. The nuspec, the install, uninstall and before-modify scripts, versioning, the pack command, a test
install from a local folder feed, and feeds on a folder, GitHub Packages or Nexus. The guide is written to be handed to customers as it is, so it stands alone: it shows the
manual steps and leaves automating them to the reader.

It is a static site built with [Zensical](https://zensical.org/) from the Markdown pages under `docs\`, in the
same setup as the ADS-Gamepad-Service documentation: `zensical.toml` holds the navigation and the theme,
`docs\stylesheets\extra.css` the Beckhoff USA Community look.

```
zensical.toml        navigation, theme, Markdown extensions
docs\
  index.md           home page: who it is for, what you need, the words, the placeholders
  packages\          building a package, in numbered steps:
    nuspec.md          steps 1-3: the folder, the nuspec, the release notes file
    scripts.md         steps 4-6: install, uninstall and before-modify scripts; the tcpkg commands
    versioning.md      version numbers, dependency versions, prereleases, upgrades
    packing.md         steps 7-10: tcpkg pack, looking inside, a local folder feed, a test install
    workloads.md       turning packages into a card in the GUI
  examples\          PLC library, .NET service, plain files (certificate, route file)
  hosting\           feeds for other PCs:
    index.md           the three kinds of feed; icons and release notes inside the package or hosted
    signatures.md      turning off signature verification
    folder.md          a folder on disk
    github.md          GitHub Packages: classic token, --take 100, what the GUI shows
    nexus.md           Nexus: repositories, push, hosted icons, hosted service index for release notes
  images\            Package Manager screenshots (feed-*.png)
  pitfalls.md
  stylesheets\       extra.css
site\                build output (git-ignored)
```

The running example in the `packages\` pages, `MyCompany.XAE.Docs.MyMachine`, was packed with `tcpkg` 2.6.24
exactly as the pages show it. The examples quote real packages: the public
[ADS-Gamepad-Service](https://github.com/Beckhoff-USA-Community/ADS-Gamepad-Service) project (the PLC library and
the Windows service) and packages built for Beckhoff USA's own feed (the certificate, route and workload
packages, ids starting with `BAUS`). When one of those packages changes, check the page that quotes it.
Company names, hostnames and feed names in the pages are placeholders on purpose.

Beckhoff RT Linux packages are not in the guide for now; the earlier pages on apt are in commit `50cf836`
(`docs\linux\` and the Linux section of `docs\pitfalls.md`). In the Nexus screenshot,
`docs\images\feed-nexus.png`, the broker name is blanked out.

## Reading it

Zensical is a Python package (`python -m pip install zensical`). From this folder:

```powershell
python -m zensical serve
```

opens the site at http://127.0.0.1:8000 and rebuilds it whenever a page changes. To build it once into
`site\` and read it straight from disk:

```powershell
python -m zensical build --clean --strict
```

then open `site\index.html`. The pages are written as plain `.html` files (`use_directory_urls = false`),
so the links work from the file system without a web server. `--strict` turns every warning, such as a
link to a page that does not exist, into a failed build; keep it on.

## Editing it

Pages are Markdown under `docs\`. A new page shows up once it is added to the `nav` list in `zensical.toml`.
The writing rules the pages follow: plain English, short sentences, every claim grounded in a real package or a
test that was run, with the TwinCAT or Package Manager version named when it matters. The Feature Release
Notes parser, the GUI and `tcpkg` all have habits worth checking before a statement goes in; the
[pitfalls page](docs/pitfalls.md) collects the ones found so far.
