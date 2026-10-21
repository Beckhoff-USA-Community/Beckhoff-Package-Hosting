# Beckhoff Package Hosting

The TwinCAT Packaging Guide: how to build TwinCAT Package Manager packages and workloads by hand. The nuspec,
the install, uninstall and before-modify scripts, versioning, the pack command, and a test install from a
local folder feed. The guide is written to be handed to customers as it is, so it stands alone: it shows the
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

Hosting a feed (a shared folder, GitHub, Nexus) and Beckhoff RT Linux packages are not in the guide for now.
Earlier pages on Nexus and on apt for Beckhoff RT Linux are in commit `50cf836` (`docs\nexus\`, `docs\linux\`,
and the Nexus and Linux sections of `docs\pitfalls.md`) for a later hosting section.

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
