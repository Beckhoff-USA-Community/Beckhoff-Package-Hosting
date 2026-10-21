# Beckhoff Package Hosting

The TwinCAT Packaging Guide: how to build TwinCAT Package Manager packages and workloads, host them on a
Nexus server of your own, and ship the same thing to Beckhoff RT Linux controllers as apt packages. The guide
is a static site built with [Zensical](https://zensical.org/) from the Markdown pages under `docs\`, in the
same setup as the ADS-Gamepad-Service documentation: `zensical.toml` holds the navigation and the theme,
`docs\stylesheets\extra.css` the Beckhoff USA Community look.

```
zensical.toml        navigation, theme, Markdown extensions
docs\
  index.md           home page
  packages\          what makes up a package, what makes up a workload, building and testing
  examples\          PLC library, .NET service, plain files
  pitfalls.md
  nexus\             setting up Nexus, publishing a release
  linux\             from tcpkg to .deb, serving it with apt
  stylesheets\       extra.css
site\                build output (git-ignored)
```

The examples quote real packages from two repositories: the Iris packaging repository
(`Iris-Manor-Custom-Nexus-Packages`, the certificate, route and workload packages plus the Nexus and apt
scripts) and the ADS-Gamepad-Service project (the PLC library, the Windows service and the Linux package).
When a package there changes, check the page that quotes it. Hostnames and feed names in the pages are
placeholders on purpose, so the guide can be shared.

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
