---
title: "Introducing packer-hyperv-windows-server: A Windows Server Image Factory for Hyper-V"
description: "Packer templates and a PowerShell wrapper that build generalized Windows Server 2019, 2022 and 2025 VHDX images on Hyper-V, with reports, a manifest and a checksum."
date: 2026-11-24
draft: false
tags: ["packer", "windows", "hyper-v", "automation", "devops"]
categories: ["Infrastructure Automation"]
keywords: ["packer-hyperv-windows-server", "packer hyper-v windows server", "windows server image factory", "generalized vhdx packer", "windows server 2025 packer template"]
---

[packer-hyperv-windows-server](https://github.com/larskroghdk/packer-hyperv-windows-server)
v0.1.0 is released: Packer templates and a PowerShell wrapper that build
Windows Server 2019, 2022 and 2025 images on Hyper-V. Standard or
Datacenter, Server Core or Desktop Experience, from Microsoft ISO media. The
output is a generalized Generation 2 VHDX with Secure Boot, plus the reports
from every step, a JSON manifest and a SHA-256.

<!--more-->

## 💡 Why a factory

The plugins I've written so far each do one job: features, optimization,
hardening, auditing, cleanup. This repository puts them together into one
build, from an ISO to a sealed disk, so that every release and installation
type goes through the same steps in the same order.

It also does the parts around Packer that matter just as much: checking
that the ISO is the one you approved, refusing to reuse or overwrite
anything, and checking the finished disk offline before it's called a
release.

## 🏭 The pipeline

One shared HCL2 build covers all three releases. The steps are:

1. Unattended GPT/UEFI setup from a generated answer CD.
2. A scoped HTTPS WinRM listener for the build.
3. Remove PowerShell 2.0 and install .NET Framework 3.5 from the same ISO
   ([windows-features]({{< relref "posts/introducing-packer-plugin-windows-features" >}})).
4. Optimize
   ([windows-optimize]({{< relref "posts/introducing-packer-plugin-windows-optimize" >}})).
5. Harden against the Microsoft baseline in compatibility mode
   ([windows-harden]({{< relref "posts/introducing-packer-plugin-windows-harden" >}})).
6. Install approved offline MSU packages, with a restart after each.
7. Windows Update with [rgl/windows-update](https://github.com/rgl/packer-plugin-windows-update)
   0.18.4, then a restart.
8. Read-only optimize and harden passes, to check that updates didn't undo
   anything.
9. Supported cleanup
   ([windows-cleanup]({{< relref "posts/introducing-packer-plugin-windows-cleanup" >}})).
10. Final validation of features, updates, component store health, Secure
    Boot and reboot state.
11. Sysprep, started by the builder's shutdown command.
12. Offline validation of the captured VHDX, then the manifest and checksum.

A successful `packer build` isn't enough. The wrapper only writes the
release manifest after the offline check passes.

## 🚀 Getting started

You need an elevated, dedicated Windows Hyper-V host with Packer 1.16.2,
the Windows ADK Deployment Tools (`oscdimg` on the PATH), Windows PowerShell
5.1, the Hyper-V and DISM modules, DHCP and access to an update source.
Plan on at least 4 GB of guest RAM and about three times the disk size in
free space. Use an isolated virtual switch.

The template pins every plugin exactly. The four custom plugins it was
written against are pinned at versions that aren't the published ones, so
for now `packer init` can't install all of them. The repository has a
script that builds them from the sibling source repositories, which needs
Go 1.26 or later:

```powershell
./scripts/Install-LocalPlugins.ps1
$env:PACKER_PLUGIN_PATH = "$PWD/.validation/plugins"
packer init .
./tests/static/Test-Project.ps1
```

Next, copy one of the
[example settings files](https://github.com/larskroghdk/packer-hyperv-windows-server/tree/main/examples)
to a private location and fill in your own values:

```json
{
  "iso_path": "C:/ISO/WindowsServer2022.iso",
  "iso_checksum": "sha256:0000000000000000000000000000000000000000000000000000000000000000",
  "image_name": "COPY EXACT IMAGE NAME FROM APPROVED MEDIA",
  "edition": "Datacenter",
  "installation_type": "Core",
  "switch_name": "IsolatedImageBuild",
  "winrm_allowed_address": "192.0.2.10/32",
  "image_version": "2026.11.1",
  "cpus": 4,
  "memory_mb": 4096,
  "disk_size_mb": 81920,
  "approved_oob_packages": []
}
```

The all-zero checksum is deliberately unusable. Image names differ between
evaluation, volume and other media, so read the exact name with
`Get-WindowsImage` instead of guessing an index. No credentials go in this
file. The administrator password comes from `PKR_VAR_administrator_password`,
injected from your secret manager and unique for each build.

Then validate, and build when you're ready:

```powershell
./scripts/Build.ps1 -Version 2022 -SettingsPath C:/Private/ws2022.json
./scripts/Build.ps1 -Version 2022 -SettingsPath C:/Private/ws2022.json -Build
```

Without `-Build`, the wrapper only validates the Packer configuration.
With it, it first checks the ISO checksum and that the image really is the
release, edition, installation type, architecture and language you asked
for, then creates a newly named VM and disposes of it afterwards. It never
reuses or removes an existing VM, switch or output directory.

## 🔐 Security choices

WinRM always uses HTTPS with NTLM. The bootstrap creates a short-lived
self-signed certificate, so Packer skips certificate validation. That's why
the guest firewall only allows the host address you configure and why the
build belongs on an isolated network. There's no autologon and no option to
fall back to plain HTTP.

The answer CD has to contain the temporary administrator password. The
wrapper uses a private, ACL-protected TEMP directory for the build and
removes it afterwards. Don't turn on `PACKER_LOG` or upload caches or VM
working directories; only the reports and the VHDX are meant to leave the
host.

At the end, the finalizer removes the bootstrap certificate, listener and
firewall rule, invalidates the build password and starts Sysprep. The
sealed image keeps no management listener. Deploy from a copy, with your own
unattend that sets the password, network and management access.

## 🔌 Sysprep and offline validation

Sysprep can't run as an ordinary provisioner, because WinRM drops the moment
generalization starts. The Hyper-V `shutdown_command` schedules a delayed
SYSTEM task and returns while the connection still works. That task does
the cleanup above and runs `sysprep /generalize /oobe /shutdown`. If it
fails, it writes `generalization-error.txt` and deliberately doesn't shut
down, so the build times out instead of capturing a broken image.

After capture, the wrapper mounts the VHDX read-only, copies its registry
hive to a scratch directory, checks for `IMAGE_STATE_GENERALIZE_RESEAL_TO_OOBE`
and that known build artifacts are gone, and unmounts it. The canonical
image is never booted.

## 📦 Output

Each build gets a unique ID:

- `output/ws<release>-<edition>-<type>-<version>-<id>/Virtual Hard Disks/*.vhdx`
- `reports/<same id>/` with every plugin report, the guest and offline
  validations, Windows Update evidence, the Packer manifest and log, and
  the release manifest and checksum.

With `-ExternalStorage`, a validated image is copied to an existing mounted
volume or SMB share, into a new directory, and the hash is checked before
the manifest is copied. Existing destinations are never overwritten.

## 📥 Updates and offline packages

The online update stage installs applicable software updates from your
configured Windows Update or WSUS source, excluding previews, and the final
scan fails the build if required updates remain. It doesn't enroll the
guest in Microsoft Update or change WSUS policy.

For out-of-band or checkpoint updates, you list approved MSU files in
`approved_oob_packages` with their path, SHA-256 and KB number, in the order
Microsoft prescribes. Each one is hash checked on the host and in the guest,
installed with WUSA as SYSTEM and followed by a restart. The
[examples](https://github.com/larskroghdk/packer-hyperv-windows-server/blob/main/examples/README.md)
have a snippet that builds the list. Nothing is downloaded or approved
automatically.

## 🤖 CI

Two GitHub workflows are included. `validate.yml` runs the static checks on
hosted Windows runners for pushes and pull requests. `build.yml` is manual,
only runs from `main`, requires an explicit authorization input and a
protected environment, and runs on a dedicated self-hosted Hyper-V runner.
Pull requests never get access to the live host or the build secret.

## 🧪 Testing status

The static suite passes 80 checks: PowerShell parsing, answer file XML and
credential escaping, pipeline invariants, documentation links and full
Packer schema validation for all six release and installation type
combinations.

No live image has been built yet. That means no Sysprep run, no captured
VHDX, no real ISO and no deployment test. There's an authorized live test
entry point for exactly that, and all six combinations are release gates.
Until they've passed, v0.1.0 is a starting point to build on a disposable
host, not certified images.

## 🔗 Links

- 📂 [GitHub repository](https://github.com/larskroghdk/packer-hyperv-windows-server)
- 🏷️ [Releases](https://github.com/larskroghdk/packer-hyperv-windows-server/releases)
- 📖 [Operations guide](https://github.com/larskroghdk/packer-hyperv-windows-server/blob/main/docs/operations.md)
- 📖 [Plugin audit](https://github.com/larskroghdk/packer-hyperv-windows-server/blob/main/docs/plugin-audit.md)
- 🐛 [Issues](https://github.com/larskroghdk/packer-hyperv-windows-server/issues)
- 📘 [HashiCorp Packer Hyper-V builder](https://developer.hashicorp.com/packer/integrations/hashicorp/hyperv/latest/components/builder/iso)
