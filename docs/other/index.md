# Not the primary focus

The rest of this guide is about the TwinCAT Package Manager on Windows. Nexus serves more than NuGet, though, and two Beckhoff controller platforms do not use the Package Manager at all. This section shows, more briefly than the rest, how the same Nexus also hosts:

* [Debian packages for Beckhoff RT Linux](debian.md), installed with `apt`;
* [FreeBSD packages for TwinCAT/BSD](freebsd.md), installed with `pkg`;
* [Docker images for TwinCAT containers](docker.md) on Beckhoff RT Linux.

It assumes you have read the [Nexus page](../hosting/nexus.md): anonymous read access, the `Authorization` header for uploads, and the raw repository `tcpkg-raw` are used here without being explained again.

## What was tested

The Debian and FreeBSD pages were done and tested in October 2026 against Nexus 3.97 Community, a CX2043 running Beckhoff RT Linux (Debian 13), and a CX running TwinCAT/BSD, first on 14.4 and then re-imaged to 15.1:

* **apt:** `dpkg -i` and `apt remove` of the package file, then `apt update` and `apt install` from the signed apt repository on Nexus.
* **pkg:** `pkg add` and `pkg delete` with the TwinCAT restart over ADS on 14 and 15, the `pre-install` check refusing the engineering route file, and `pkg update` and `pkg install` from the signed repository for each release.

What was not tested:

* The RUN branch of the TwinCAT restart on TwinCAT/BSD. No test box had a boot project. The [FreeBSD page](freebsd.md#do-not-restart-the-service-from-a-script) says what that means.
* Building the packages with `dpkg-deb` and `pkg create`. The tested packages were written by a small script on Windows that produces the same archives. Both tools are the standard way to build by hand, and each page shows how to check the result before you ship it.

The [Docker page](docker.md) is shorter still: it shows that Nexus can hold your container images too, with one small example.

## One example: the ADS-over-MQTT route file

Every page uses the same example: the runtime route file from [plain files](../examples/files.md#a-route-file), the one that connects a controller to an MQTT broker. The job is the same on every platform. What differs is the package format and who does which part of the work:

| Job | TwinCAT Package Manager | Beckhoff RT Linux (apt) | TwinCAT/BSD (pkg) |
|---|---|---|---|
| Put the route file in place | install script copies it into every route folder | package file at `/etc/TwinCAT/3.1/Target/Routes/` | package file at `/usr/local/etc/TwinCAT/3.1/Target/Routes/` |
| Make the router read it | restart `TcSysSrv` | `postinst` restarts `TcSystemServiceUm` | `post-install` asks TwinCAT to restart over ADS |
| Take it out again | uninstall script deletes it | dpkg deletes it | pkg deletes it |
| Keep the other variant out | install script checks packages and route files | `Conflicts:` in `control` | `pre-install` checks the route folder |
| Depend on the TwinCAT runtime | `TwinCAT.XAR.Base` | `tc31-xar-um` | `TC31-XAR` and `TcAdsTool` |

In a container, the route file is not a package at all. It is part of the image; see [the Docker page](docker.md#twincat-containers).

## The repositories on Nexus

The Nexus page created the first two. This section adds the other two, and puts more files on the raw repository:

| Name | Recipe | Holds | Read by |
|---|---|---|---|
| `nuget-hosted` | nuget (hosted) | the `.nupkg` files | `tcpkg` |
| `tcpkg-raw` | raw (hosted) | icons, release notes, the service index; also the public key of the apt repository and the FreeBSD package repositories | the Package Manager GUI, `curl`, `pkg` |
| `apt-hosted` | apt (hosted) | the `.deb` files, signed by Nexus | `apt` |
| `docker-hosted` | docker (hosted) | container images | `docker` |

Nexus has no FreeBSD format. A `pkg` repository is a folder of static files, and a raw repository serves those as they are.

## What the clients need that a Package Manager PC does not

On Windows, a feed is one `tcpkg source add` and signature verification turned off. The other clients each need a little more, and each needs it on every controller:

| | apt (Beckhoff RT Linux) | pkg (TwinCAT/BSD) | docker (Beckhoff RT Linux) |
|---|---|---|---|
| **Registering the source** | a line in `/etc/apt/sources.list.d/` | a file in `/usr/local/etc/pkg/repos/` | nothing; every image name carries the server's name |
| **Trusting it** | the repository's public key in `/etc/apt/keyrings/`, named with `signed-by`. Nexus does not serve that key; you put it on the raw repository. | a fingerprint file of your signing key. You sign the repository yourself, on a TwinCAT/BSD box. | the server's HTTPS certificate only; there is no signing key to install |
| **Logging in to read** | none with anonymous read. Otherwise a login file in `/etc/apt/auth.conf.d/`, the way Beckhoff's own repository uses `bhf.conf`. | none with anonymous read | none when the repository allows anonymous pulls; otherwise `docker login` on every controller |
| **Picking the right build** | `Architecture: all` for a package without programs | one build and one repository folder per TwinCAT/BSD release | an image is built for one processor architecture |
| **Root rights** | `sudo` | `doas` | `sudo` |

**The server certificate.** All three clients check the HTTPS certificate of the reverse proxy in front of Nexus, each against its own list of trusted authorities. The test server used a certificate from a public authority, and every client trusted it with no change. A certificate from an in-house authority needs that authority's certificate on every controller, in three different places. The commands below are the documented ones for each platform; they were not tried for this guide:

* apt: copy it to `/usr/local/share/ca-certificates/` with the extension `.crt`, then `sudo update-ca-certificates`.
* pkg: copy it to `/usr/local/etc/ssl/certs/` with the extension `.pem`, then `certctl rehash` as root.
* docker: copy it to `/etc/docker/certs.d/nexus.example.com/ca.crt`. Docker reads that folder for this one server only.

**Versions are immutable here too.** apt clients cache by version, `pkg` compares versions, and a Docker tag that changes under the same name fools every controller that already pulled it. Every change is a new version, as on Windows.
