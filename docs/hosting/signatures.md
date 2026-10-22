# Signature verification

Beckhoff signs its TwinCAT packages, so that their integrity can be checked wherever a copy is kept ([TwinCAT Package Manager on Infosys](https://infosys.beckhoff.com/content/1033/tc3_installation/15698617995.html)). The Package Manager checks that signature before it installs a package. The setting is called `VerifySignatures`, and it is on by default.

A package you pack yourself carries no Beckhoff signature. With checking on, adding a feed of your own fails, and so does every install from it. So every PC that uses your feed turns checking off first, from an administrator PowerShell:

```powershell
tcpkg config unset -n VerifySignatures
tcpkg config list
```

`tcpkg config list` then shows `VerifySignatures: False`.

Three things to know about it:

* **Before the feed, not after.** Turn checking off before `tcpkg source add`. With checking on, adding the feed fails.
* **For the whole PC.** It is a Package Manager setting, not a property of one feed; `tcpkg source add` has no option for it. It stays off until someone turns it on again.
* **Tell your users.** Whoever installs your packages has to run that line once. Put it at the top of your installation instructions, next to the `tcpkg source add` line.

Turning checking back on was not tried for this guide. The setting is `VerifySignatures`, and `tcpkg config set --help` shows how to set it. Try it on a test PC and check with `tcpkg config list` that it reads `VerifySignatures: True` before you rely on it.

Next: pick a feed. The simplest is [a folder on disk](folder.md).
