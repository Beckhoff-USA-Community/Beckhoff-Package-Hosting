# Signing your packages

Beckhoff signs every package on its feed, and the Package Manager checks that signature before it installs anything. Since Package Manager 2.6 you can do the same for your own packages: sign them once with a certificate of your own, let every PC trust that certificate once, and leave signature verification on. Before 2.6 the only way to use a feed of your own was to [turn verification off](signatures.md) on every PC, and that is still the fallback for a PC with an older Package Manager.

Everything on this page was done with `tcpkg` 2.6.24, a self-signed certificate, a folder feed and a Nexus feed.

## How it works

`tcpkg sign` adds one file to the package, `.signature.p7s`, at the root of the zip. It holds a hash of the package and the certificate that signed it. Nothing else in the package changes, so a signed package is the same package with one more entry.

The Package Manager keeps a trust store of its own, separate from the Windows certificate stores, and `tcpkg certificate list` shows what is in it. A certificate that is only in your Windows user store is not trusted: in the test, a self-signed certificate created in the user's personal store failed verification until it was added with `tcpkg certificate add`.

With the setting `VerifySignatures` on, every package is checked when it is downloaded for an install. A package without a signature stops there with "Package integrity verification failed", exit code 574, and nothing is installed. Adding a feed, listing it and searching it are not affected: in the test, a feed of unsigned packages was added and listed with verification on, and only the install of an unsigned package failed.

## Step 1: a certificate

One certificate for the company, kept by whoever packs. A self-signed one is enough, because the Package Manager trusts whatever is in its store and never asks who issued it. From a normal PowerShell on the PC you pack on:

```powershell
$cert = New-SelfSignedCertificate -Type CodeSigningCert -Subject 'CN=MyCompany Package Signing' -CertStoreLocation Cert:\CurrentUser\My -NotAfter (Get-Date).AddYears(5)
Export-Certificate -Cert $cert -FilePath .\MyCompany-Signing.cer
$cert.Thumbprint
```

That creates the certificate with its private key in your personal certificate store, writes the public half to `MyCompany-Signing.cer`, and prints the thumbprint that `tcpkg sign` is called with. The `.cer` file holds no secret and is what your users get. The private key stays in the store of the account that created it; `Export-PfxCertificate` (a standard Windows command, not tried for this guide) writes a password protected backup of it, and `tcpkg sign --pfx` signs from such a file on a PC where the certificate is not installed.

## Step 2: sign the package

After `tcpkg pack` and before the package goes on a feed:

```powershell
tcpkg sign .\MyCompany.XAE.Docs.MyMachine.1.0.0.nupkg --thumbprint <thumbprint> --timestamp-server http://timestamp.digicert.com
```

"Package signed successfully" and the file is replaced by its signed copy; `-o <path>` writes it somewhere else instead. `--timestamp-server` is required. It names an RFC 3161 server that countersigns the time of signing, and the DigiCert one above answered in the test; `--timestamp-server none` signs without a timestamp. The option needs the PC to reach the server, which is worth knowing on a closed network. Signing does not need an administrator shell.

A package is signed once, after its last change. Any change to the file afterwards breaks the signature, so sign as the last step before publishing, and sign every version, including old versions you keep on the feed: with verification on, an unsigned version on your feed can no longer be installed.

## Step 3: check the signature

`tcpkg verify` reads the signature without installing anything:

```powershell
tcpkg verify .\MyCompany.XAE.Docs.MyMachine.1.0.0.nupkg
```

On the PC that signed it, before the certificate is in the trust store, the answer is two things at once:

```
The file '...MyCompany.XAE.Docs.MyMachine.1.0.0.nupkg' is invalid: Failure_SignerValidationFailed
Package integrity is valid.
Signature hash algorithm: SHA-512
Signer is invalid: Failure_NoSignerCertificatePathValidates.
	Subject name: CN=MyCompany Package Signing
	Issued by: CN=MyCompany Package Signing
	...
```

The package is intact and signed, and the signer is not trusted yet. After step 4 the same command says "is valid" and "Signer is valid.", exit code 0. An unsigned package answers "is invalid: Failure_MissingP7S", and both failures exit with code 4204.

## Step 4: hand out the certificate

Every PC that installs from your feed needs the `.cer` file once. Put it where the feed is:

* **A folder feed or a USB stick:** next to the packages, `C:\MyPackages\MyCompany-Signing.cer`.
* **Nexus:** on the raw repository next to the icons, the same `PUT` as for an icon on the [Nexus page](nexus.md#uploading-to-the-raw-repository), for example at `https://nexus.example.com/repository/tcpkg-raw/certificates/mycompany-signing.cer`. Anonymous read is what the users need. The upload itself was not tried with a `.cer` for this guide; it is the same mechanism as the icons.
* **GitHub:** in the repository that holds the package sources, or as a release asset, and link it from the feed instructions.

Then, on each PC, from an administrator PowerShell:

```powershell
tcpkg certificate add C:\MyPackages\MyCompany-Signing.cer -n MyCompany
tcpkg certificate list
```

"Certificate added to TcPkg store. Thumbprint: ...", and the list shows thumbprint, subject, issuer, the validity dates and whether it is self-signed. `-n` is a display name. Put these two lines in your installation instructions where "turn verification off" used to be.

## Step 5: leave verification on

On a PC that has never had a feed of its own, `VerifySignatures` is still on, and nothing more is needed. On a PC where it was turned off for an earlier, unsigned feed, turn it back on:

```powershell
tcpkg config set -n verifySignatures -y
tcpkg config list
```

"Successfully enabled signature verification.", and `config list` shows `VerifySignatures: True`. From then on every install from every feed on that PC is checked, so every package on every feed of your own has to be signed first. Beckhoff's packages are signed already.

The order of the three lines on a new PC does not matter. In the test, a feed was added with verification on before its certificate was trusted, the certificate was added afterwards, and the install worked.

## When an install is refused

| Message | Exit code | Meaning |
|---|---|---|
| "Package integrity verification failed, details have been written to the logfile." | 574 | the package is unsigned, or its signer is not in the trust store; the install stops at the download |
| `tcpkg verify`: "Failure_MissingP7S" | 4204 | no signature in the package |
| `tcpkg verify`: "Failure_NoSignerCertificatePathValidates" | 4204 | signed, but the certificate is not in this PC's trust store |

The log, `C:\ProgramData\Beckhoff\TcPkg\logs\TcPkg.log`, adds only the exception name. `tcpkg verify` on the package is the quicker diagnosis.

## Replacing the certificate

A new certificate, because the old one expires or its key was lost, is a new `.cer` for every PC. Add it with `tcpkg certificate add` before the first package signed with it reaches the feed, and leave the old one in the trust store as long as packages signed with it are on the feed. `tcpkg certificate remove <thumbprint>` takes one out again; the thumbprint is in `tcpkg certificate list`.

## Not tried for this guide

* A certificate from a company certificate authority, and whether the Package Manager also trusts the Windows certificate stores. The self-signed route above does not depend on either.
* What the timestamp changes once the certificate has expired.
* A GitHub Packages feed with verification on.
* Package Manager versions before 2.6. The `sign` and `certificate` commands are documented by Beckhoff for the current version; a PC with an older Package Manager cannot trust your certificate, and for it [turning verification off](signatures.md) stays the way.

Next: [a folder on disk](folder.md), the simplest feed.
