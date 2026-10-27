# Signature verification

Beckhoff signs its TwinCAT packages, so that their integrity can be checked wherever a copy is kept ([TwinCAT Package Manager on Infosys](https://infosys.beckhoff.com/content/1033/tc3_installation/15698617995.html)). The Package Manager checks that signature when it downloads a package for an install. The setting is called `VerifySignatures`, and it is on by default.

A package you pack yourself carries no signature, and with the setting on it cannot be installed: the download stops with "Package integrity verification failed", exit code 574. There are two ways to deal with that, and the choice is made once per feed:

* **[Sign your packages](signing.md).** Package Manager 2.6 and later can trust a certificate of your own. The packer signs every package, every PC runs one `tcpkg certificate add`, and verification stays on. This is the better way wherever every PC runs Package Manager 2.6 or later.
* **Turn verification off** on every PC that uses the feed. The only way for older Package Managers, and the quicker one for a test PC.

## Turning it off

From an administrator PowerShell:

```powershell
tcpkg config unset -n VerifySignatures
tcpkg config list
```

`tcpkg config list` then shows `VerifySignatures: False`. Three things to know about it:

* **For the whole PC.** It is a Package Manager setting, not a property of one feed; `tcpkg source add` has no option for it. It stays off until someone turns it on again, and while it is off, no package from any feed is checked, Beckhoff's included.
* **Only installs are affected.** Adding a feed and listing it work with verification on, unsigned packages or not: tested with `tcpkg` 2.6.24 and a folder of unsigned packages. What fails is the install, at the download of the package. So the line can come before or after `tcpkg source add`; it has to come before the first install.
* **Tell your users.** Whoever installs your packages has to run that line once. Put it in your installation instructions next to the `tcpkg source add` line, with a word on what it switches off.

## Turning it back on

```powershell
tcpkg config set -n verifySignatures -y
tcpkg config list
```

"Successfully enabled signature verification.", and `config list` reads `VerifySignatures: True`. From then on every unsigned package on every feed of the PC is refused, so switch it on only after [every package on your feeds is signed](signing.md) and the certificate is trusted on that PC.

Next: [signing your packages](signing.md), or straight on to a feed, starting with [a folder on disk](folder.md).
