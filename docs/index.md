# TwinCAT Packaging Guide

This guide is for the engineer who has something to hand out, such as a PLC library, a Windows service, a certificate, a route file or a folder of documents, and wants it to install with one click in the TwinCAT Package Manager instead of a zip file and a page of instructions.

It shows how to build such a package by hand: the folder, the nuspec, the install scripts, the version number, the pack command, and a test install on your own PC. Then how several packages become one card in the Package Manager, and how to host them so that other PCs can install them: from a folder, from GitHub Packages, or from a Nexus server. Everything is done with the tools the Package Manager brings and a text editor. Once you have built a package by hand, automating the steps is up to you.

Everything here was learned by doing it. The examples quote real packages: the public [ADS-Gamepad-Service](https://github.com/Beckhoff-USA-Community/ADS-Gamepad-Service) project, and packages built for Beckhoff USA's own feed. Where something was verified against a particular TwinCAT or Package Manager version, the page says so.

## What you need

* TwinCAT 3.1 Build 4026 with the **TwinCAT Package Manager**. It brings the command line tool `tcpkg`. The pages were checked with Package Manager 2.6 (`tcpkg` 2.6.24).
* Administrator rights on the PC you test on. Installing and uninstalling packages needs them, because the install scripts run as administrator. The Package Manager GUI itself runs without them and asks for elevation only when it needs it.
* A text editor. The package scripts are PowerShell, and the Package Manager brings the PowerShell it runs them with.

## The words

TwinCAT 3.1 Build 4026 installs through the **TwinCAT Package Manager**. It has a GUI and a command line tool, `tcpkg`, and both do the same thing. Under the hood it is NuGet: Beckhoff [describes it](https://infosys.beckhoff.com/content/1033/tc3_installation/15698617995.html) as built on NuGet technology, the NuGet package format and NuGet communication. The install scripts follow Chocolatey's conventions. That matters, because it explains most of what follows.

* A **package** is one unit of install: a `.nupkg` file (a zip) with a metadata file, the files to install, and a PowerShell script that installs them. Beckhoff's own PLC libraries, drivers and the engineering itself are packages.
* The **nuspec** is the package's metadata file: id, version, title, description, dependencies, and which files go in.
* A **workload** is a package that installs nothing itself and exists to pull in other packages. It is what the GUI shows as a card. You pick a workload; the Package Manager installs the packages it depends on.
* **XAE** is the engineering side (Visual Studio, the PLC compiler, libraries). **XAR** is the runtime side (the thing that runs the PLC). A package is usually for one or the other, and the id says which.
* A **feed** is where packages come from: Beckhoff's own server, a folder on disk, or a NuGet server you host. The [hosting section](hosting/index.md) covers all three.
* **KM** and **UM** are the two ways the TwinCAT runtime can run on Windows: in kernel mode as always, or as the Usermode Runtime. A package that touches the runtime's files has to know both.

## The way through

1. **Building a package.** Read these in order. They build one small package from an empty folder to a test install, in numbered steps: [the folder and the nuspec](packages/nuspec.md), [install, upgrade and uninstall](packages/scripts.md), [versioning](packages/versioning.md), [packing and testing](packages/packing.md). Then [workloads](packages/workloads.md) put packages on a card.
2. **Examples.** The same steps on the packages people most often build: a [PLC library](examples/plc-library.md), a [.NET service](examples/dotnet-service.md), and [plain files](examples/files.md) such as a certificate or a route file.
3. **[Hosting](hosting/index.md).** Put the packages on a feed other PCs can reach: [a folder on disk](hosting/folder.md), [GitHub Packages](hosting/github.md) or [Nexus](hosting/nexus.md). Read the first page of the section whichever you pick; it explains why icons and release notes need extra care on a server.
4. **[Pitfalls](pitfalls.md).** Read it before you share anything. Most of it is one line each, and every line cost somebody an afternoon.
5. **[Self-hosting other packages](self-hosting/index.md).** The same Nexus can also host Debian packages for Beckhoff RT Linux, FreeBSD packages for TwinCAT/BSD, and Docker images. Each platform has a page for building and publishing a package and a page for setting up a controller, with the ADS-over-MQTT route file as the example.

## Names used in this guide

Company names, hostnames and feed names in this guide are placeholders. Swap them for your own:

| Placeholder | Meaning |
|---|---|
| `MyCompany` | your company; the first part of every package id |
| `MyMachine` | your product |
| `MyFeed` | the name of a feed as the Package Manager shows it |
| `https://nexus.example.com` | your Nexus server |
| `nuget-hosted`, `tcpkg-raw` | the two repositories on Nexus: the packages, and the files the GUI needs next to them |
| `apt-hosted`, `docker-hosted` | the apt and Docker repositories on Nexus, in [self-hosting other packages](self-hosting/index.md) |
| `mqtt.example.com` | the MQTT broker the ADS-over-MQTT example connects to |

Package ids that start with `BAUS` are Beckhoff USA's own, and ids that start with `Beckhoff-USA-Community` come from the community projects. They are quoted as they are.
