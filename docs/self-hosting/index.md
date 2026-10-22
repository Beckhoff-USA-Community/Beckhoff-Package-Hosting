# Self-hosting other packages

The rest of this guide is about the TwinCAT Package Manager on Windows. Two Beckhoff controller platforms do not use it: Beckhoff RT Linux installs software with `apt`, and TwinCAT/BSD with `pkg`. The Nexus server that hosts your NuGet feed can host their packages too, and your container images as well. This section shows how, with one example throughout.

It assumes you have set up Nexus as on the [Nexus page](../hosting/nexus.md). Anonymous read access, the `$auth` header for uploads, and the raw repository `tcpkg-raw` are used here as they are set up there.

## The pages

Each platform has two pages. The first builds a package and publishes it to Nexus. The second sets up a controller to install from there, and is what you repeat on every controller.

| Platform | Building and publishing | On the controller |
|---|---|---|
| Beckhoff RT Linux (`apt`) | [Package building for Beckhoff RT Linux](rt-linux-building.md) | [Client package usage on Beckhoff RT Linux](rt-linux-clients.md) |
| TwinCAT/BSD (`pkg`) | [Package building for TwinCAT/BSD](tcbsd-building.md) | [Client package usage on TwinCAT/BSD](tcbsd-clients.md) |
| Containers (`docker`) | [Docker images](docker.md) | the same page |

## The example: the ADS-over-MQTT route file

Both package pages build the same package: the runtime route file from [plain files](../examples/files.md#a-route-file), which connects a controller to an MQTT broker. Its Debian and FreeBSD name is `baus-adsovermqtt-xar`, the counterpart of `BAUS.AdsOverMqtt.XAR`.

The package has the same jobs on every platform. Who does which part differs:

| Job | TwinCAT Package Manager | Beckhoff RT Linux (apt) | TwinCAT/BSD (pkg) |
|---|---|---|---|
| Put the route file in place | the install script copies it | dpkg unpacks it to `/etc/TwinCAT/3.1/Target/Routes/` | pkg unpacks it to `/usr/local/etc/TwinCAT/3.1/Target/Routes/` |
| Make the router read it | the script restarts `TcSysSrv` | `postinst` restarts `TcSystemServiceUm` | `post-install` asks TwinCAT to restart over ADS |
| Take it out again | the uninstall script deletes it | dpkg deletes it | pkg deletes it |
| Keep the engineering variant out | the install script checks | `Conflicts:` in `control` | `pre-install` checks the route folder |
| Require the TwinCAT runtime | `TwinCAT.XAR.Base` | `tc31-xar-um` | `TC31-XAR` and `TcAdsTool` |

## The repositories on Nexus

The Nexus page created the first two. This section adds two more, and puts more files on the raw repository:

| Name | Recipe | Holds |
|---|---|---|
| `nuget-hosted` | nuget (hosted) | the `.nupkg` files for the Package Manager |
| `tcpkg-raw` | raw (hosted) | as before, plus the public key of the apt repository and the TwinCAT/BSD package repositories |
| `apt-hosted` | apt (hosted) | the `.deb` files for Beckhoff RT Linux |
| `docker-hosted` | docker (hosted) | container images |

Nexus has no FreeBSD format. It does not need one: a `pkg` repository is a folder of plain files, and the raw repository serves them as they are.

## What a controller needs that a Package Manager PC does not

On Windows, a feed is one `tcpkg source add` with signature checking turned off. The other clients need a little more, once per controller. The client pages walk through it step by step:

| | Beckhoff RT Linux (apt) | TwinCAT/BSD (pkg) | Docker |
|---|---|---|---|
| **Register the server** | a line in `/etc/apt/sources.list.d/` | a file in `/usr/local/etc/pkg/repos/` | nothing; the server is part of every image name |
| **Trust it** | the repository's public key, which you host on the raw repository | the fingerprint of the key you signed the repository with | the server's HTTPS certificate only |
| **Pick the right build** | nothing for a package without programs | the folder for the box's TwinCAT/BSD release | an image for the controller's processor |
| **Become root** | `sudo` | `doas` | `sudo` |

All three check the HTTPS certificate of your server. A certificate from a public authority needs nothing more. One from an in-house authority has to be added on every controller; each client page shows where.
