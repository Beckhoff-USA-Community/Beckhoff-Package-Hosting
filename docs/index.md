# TwinCAT Packaging Guide

This guide is for the engineer who has a thing to hand out, a PLC library, a Windows service, a certificate, a route file, and wants it to install with one click in the TwinCAT Package Manager instead of a zip file and a page of instructions. It walks through what a package is made of, how a workload groups packages into one card in the Package Manager, how to build and test both, how to host them on a Nexus server of your own, and how to take the same package to a Beckhoff RT Linux controller as an apt package.

Everything here was learned by doing it. The examples quote real packages: the ones in this repository and the ones in the [ADS-Gamepad-Service](https://github.com/Beckhoff-USA-Community/ADS-Gamepad-Service) project. Where something was verified against a particular TwinCAT or Package Manager version, the page says so.

## The words

TwinCAT 3.1 Build 4026 installs through the **TwinCAT Package Manager**. It has a GUI and a command line tool, `tcpkg`, and both do the same thing. Under the hood it is NuGet with Chocolatey style install scripts, which matters because it explains most of what follows.

* A **package** is one unit of install: a `.nupkg` file (a zip) with a metadata file, the files to install, and a PowerShell script that installs them. Beckhoff's own PLC libraries, drivers and the engineering itself are packages.
* A **workload** is a package that installs nothing itself and exists to pull in other packages. It is what the GUI shows as a card. You pick a workload; the Package Manager installs the packages it depends on.
* **XAE** is the engineering side (Visual Studio, the PLC compiler, libraries). **XAR** is the runtime side (the thing that runs the PLC). A package is usually for one or the other, and the id says which.
* A **feed** is where packages come from: Beckhoff's own server, a folder on disk, or a NuGet server you host. This guide hosts one on **Sonatype Nexus**.
* **KM** and **UM** are the two ways the TwinCAT runtime can run on Windows: in kernel mode as always, or as the Usermode Runtime. A package that touches the runtime's files has to know both.

## Where to start

If you have never opened a nuspec file, read [what makes up a package](packages/anatomy.md) and [what makes up a workload](packages/workloads.md) in that order, then [building and testing](packages/building.md). The three example pages show the same ideas on real packages: a [PLC library](examples/plc-library.md), a [.NET service](examples/dotnet-service.md), and [plain files](examples/files.md) such as a certificate or a route file.

[Pitfalls](pitfalls.md) is the page to read before you publish anything. Most of it is one line each and every line cost somebody an afternoon.

The Nexus pages cover [setting up the server](nexus/setup.md) once and [publishing a release](nexus/publishing.md) every time after that. The Linux pages take the ADS-over-MQTT route package [from tcpkg to a Debian package](linux/tcpkg-to-deb.md) and then [serve it through apt](linux/apt.md) from the same Nexus.

## Names used in this guide

Hostnames and feed names in this guide are placeholders so the pages can be shared. Swap them for your own:

| Placeholder | Meaning |
|---|---|
| `https://nexus.example.com` | your Nexus server |
| `MyFeed` | the name of the feed as the Package Manager shows it |
| `nuget-hosted`, `tcpkg-raw`, `apt-hosted` | the three repositories on Nexus: packages, the files the GUI needs next to them, and the Debian packages |
| `mqtt.example.com` | the MQTT broker the ADS-over-MQTT example connects to |

The examples come from a packaging repository that is laid out like this, and every script in it reads the real values from `packaging\feed.psd1`:

## The repository the examples come from

```
packaging\
  feed.psd1                 feed settings (Nexus host, repository names, feed name)
  packages\<Id>\            one folder per package: nuspec, release notes, icon, tools\ scripts
  deb\<package>\            one folder per Debian package: control plus maintainer scripts
  secrets\                  git-ignored: certificates, passwords, keys, Nexus credentials
  scripts\                  build, publish, setup and test scripts (PowerShell 7)
  feed\  feed-deb\  feed-index\   build output (git-ignored)
```

That is the `Iris-Manor-Custom-Nexus-Packages` repository. The PLC library, the Windows service and the Linux service package are quoted from the [ADS-Gamepad-Service](https://github.com/Beckhoff-USA-Community/ADS-Gamepad-Service) repository, under `packaging\` and `linux\` there.
