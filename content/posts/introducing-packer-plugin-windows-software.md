---
title: "Introducing packer-plugin-windows-software: Verified Software Installs in Packer Builds"
description: "Installing common Windows software in Packer images from vendor URLs, with SHA-256 and Authenticode checks on the host and the guest, and a catalog that updates itself."
date: 2026-11-23
draft: false
tags: ["packer", "windows", "security", "automation", "devops"]
categories: ["Infrastructure Automation"]
keywords: ["packer-plugin-windows-software", "packer install software windows", "windows image software installation", "authenticode verification", "winget packer"]
---

[packer-plugin-windows-software](https://github.com/larskroghdk/packer-plugin-windows-software)
v0.1.0 is released: a Packer provisioner that installs a small catalog of
Windows software into an image. No installer runs until its SHA-256 hash and
Authenticode signer have been checked, first on the Packer host and then
again on the guest.

<!--more-->

## The problem

Most image pipelines install their software the same way: a provisioner
downloads `latest` from a vendor URL and runs it with some silent switches.
That URL is a moving target. The bytes behind it change without notice, and
nothing in the build records which version went into the image, or checks
that the file is what the vendor signed.

The alternative is to keep installers in a file share and update them by
hand, which means someone has to remember to do it.

I wanted both: current versions without manual work, and an installer that
is pinned to a hash and a signer before it gets anywhere near the image.

## What it installs

| ID | Package | Installer | Server Core |
| --- | --- | --- | --- |
| `vcredist` | Microsoft Visual C++ 2015-2022 x64 runtime | EXE | Yes |
| `powershell7` | Microsoft PowerShell 7 | MSI | Yes |
| `adobereader` | Adobe Acrobat Reader x64 | EXE | No |
| `chrome` | Google Chrome Enterprise x64 | MSI | No |
| `edge` | Microsoft Edge for Business x64 | MSI | No |

All packages are declared for Windows Server 2019, 2022 and 2025 with
Desktop Experience, Windows 10 1809 and later, and Windows 11. Those
declarations come from the vendors' published requirements. The guest must
be x64; ARM64 and x86 are rejected. Every selected package is checked
against the detected OS before anything is installed.

## How an install is verified

The plugin reads a catalog file, `catalog.json`, on the Packer host. Each
record pins one version of one package: the vendor URL, the SHA-256, the
allowed signer names, the silent arguments, how to detect the installed
version and which exit codes mean success. For each package in list order:

1. Check the installed version on the guest. If it already matches, skip it.
2. Download the installer on the Packer host over HTTPS, size-bounded, and
   verify its SHA-256.
3. Upload it through the builder's communicator.
4. On the guest, verify the SHA-256 again and require a valid Authenticode
   signature from an allowed signer.
5. Run the silent install. The plugin never reboots.
6. Read the installed version and require it to equal the catalog record.

Installed versions are read from the file version of the main executable or
a registry value, never through `Win32_Product`, which is slow and can
trigger MSI repairs.

Exit codes 0 and 3010 are success, and 3010 is reported as
`reboot_required`. Exit code 1641 means the installer started a restart on
its own, and that always fails the build. An installer that runs past its
timeout is left alone rather than killed mid-transaction, and the build
fails with the guest marked as being in an unknown state. Discard that VM.

The plugin also refuses to downgrade. Pinning a version older than what's
installed is an error, not a silent downgrade.

## A catalog that updates itself

The catalog isn't edited by hand. A scheduled GitHub Actions workflow runs
every day:

1. It looks up the newest stable version of each package in Microsoft's
   [winget-pkgs](https://github.com/microsoft/winget-pkgs) index with an x64,
   machine-scope installer, downloads it from a host the package recipe
   allows, and checks the index's SHA-256. The index is used to find new
   versions, not as a trust anchor.
2. On Windows, it requires a valid Authenticode signature from an allowed
   signer and an embedded product version that matches the candidate.
3. It publishes a release, `software/<id>/<version>`, with the catalog record
   and a checksum file, and appends the record to `packages/catalog.json`.

Records are immutable. A version already in the catalog has to keep its
hash, and a concurrent change to the catalog fails instead of being
overwritten.

Only metadata is published, not the installers. Redistributing vendor
software needs permission, and a free download isn't one. Adobe's
distribution terms in particular have to be reviewed first. So the plugin
downloads from the vendor's URL at build time, and the hash check makes
sure it gets the same bytes the workflow validated. If the vendor replaces
the file behind the URL, the build fails safely rather than installing
something else.

## Getting started

The plugin is published on GitHub, so `packer init` can install it. The
Packer host needs HTTPS access to the vendor download hosts, and the guest
needs Windows PowerShell 5.1, an administrative WinRM account, and network
access for certificate revocation checks. Use WinRM over TLS with
certificate validation.

The catalog lives in the repository rather than inside the plugin, so
fetch a copy into your template directory. Take it from a tag or commit you
have reviewed, not from `main`:

```powershell
New-Item -ItemType Directory packages -Force | Out-Null
Invoke-WebRequest `
    -Uri 'https://raw.githubusercontent.com/larskroghdk/packer-plugin-windows-software/v0.1.0/packages/catalog.json' `
    -OutFile packages/catalog.json
```

Then add the provisioner:

```hcl
packer {
  required_plugins {
    windows-software = {
      source  = "github.com/larskroghdk/windows-software"
      version = ">= 0.1.0"
    }
  }
}

build {
  sources = ["source.vsphere-iso.ws2025"]

  provisioner "windows-software" {
    packages     = ["vcredist", "powershell7", "chrome", "edge"]
    catalog_path = "packages/catalog.json"
    versions = {
      powershell7 = "7.6.10"
    }
    report_path = "reports/ws2025-software.json"
  }

  provisioner "windows-restart" {}
}
```

```powershell
packer init .
packer build .
```

Packages that aren't in `versions` get the newest validated version in the
catalog. For a reproducible build, pin every package and keep the catalog
file in your own repository. A pin of `7.6.10` matches a record of
`7.6.10.0`. The restart afterwards is your call: the plugin reports when an
installer asked for one but never restarts on its own.

Browsers update themselves after deployment. That's expected; manage it
through Edge and Chrome policy rather than by pinning the image.

The repository has a
[complete example](https://github.com/larskroghdk/packer-plugin-windows-software/tree/main/examples)
with the null builder and a PowerShell runner that builds the plugin and
runs the template against an existing disposable VM.

## The report

The report is written on the Packer host, also after a failure. For each
package it records the requested and resolved version, the version before
and after, the status, the source URL, the SHA-256, the exit code, whether a
restart is required and how long it took.

```powershell
$report = Get-Content .\reports\ws2025-software.json -Raw | ConvertFrom-Json
$report.operating_system
$report.packages |
    Select-Object id, previous_version, final_version, status, reboot_required, error
```

## Adding a package

An ordinary MSI or EXE needs no code, only a recipe in `packages/<id>.json`:
the winget ID, the exact signer names, the allowed download hosts, the
installer type, silent and no-restart arguments, how to detect the
installed version, exit codes and supported OS families. The resolver and
validation scripts can be run locally to check a candidate without
installing it. Review the signature and the vendor's terms before you add
one.

## Testing status

The five installers in the catalog were validated on 2026-11-20 for
SHA-256, Authenticode signer and embedded product version. Go tests cover
the catalog, version comparison and downloads, and the guest script has
Pester tests on both Windows PowerShell 5.1 and PowerShell 7.

That validates trust, not installation. A full install of every package on
every supported OS hasn't been recorded yet, so run the example against a
disposable VM before you put it in a production pipeline.

## Links

- [GitHub repository](https://github.com/larskroghdk/packer-plugin-windows-software)
- [Releases](https://github.com/larskroghdk/packer-plugin-windows-software/releases)
- [Vendor and SDK references](https://github.com/larskroghdk/packer-plugin-windows-software/blob/main/docs/sources.md)
- [Issues](https://github.com/larskroghdk/packer-plugin-windows-software/issues)
- [HashiCorp Packer documentation](https://developer.hashicorp.com/packer)
