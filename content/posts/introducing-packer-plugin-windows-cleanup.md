---
title: "Introducing packer-plugin-windows-cleanup: Cleaning Up Windows Images at the End of a Packer Build"
description: "A Packer provisioner for the last step of a Windows template build: supported component cleanup, removal of build artifacts, system validation and a JSON report."
date: 2026-11-24
draft: false
tags: ["packer", "windows", "automation", "devops"]
categories: ["Infrastructure Automation"]
keywords: ["packer-plugin-windows-cleanup", "packer windows cleanup", "windows image cleanup", "dism component cleanup packer", "windows template sysprep packer"]
---

[packer-plugin-windows-cleanup](https://github.com/larskroghdk/packer-plugin-windows-cleanup)
v0.1.0 is released: a Packer provisioner for the last step of a Windows
template build. It removes stale temporary files and the build artifacts you
list, runs the supported DISM component cleanup, checks that the system is
healthy and writes a JSON report on the Packer host.

<!--more-->

## Why a cleanup plugin

The end of most Windows image builds is a cleanup script that has grown over
the years. It deletes `C:\Windows\Temp\*`, maybe `SoftwareDistribution`,
maybe parts of WinSxS, and empties a few log folders. Some of that is fine.
Some of it breaks Windows Update or the ability to repair and uninstall
software later, and you only find out when a server built from the image
won't patch.

This plugin does the cleanup that's safe and supported, and it refuses the
rest. What it deletes is limited by fixed roots, by age and by lists you
write in the template, and every run ends with a report of what it removed.

## What it cleans

Everything except stale temporary files is off until you turn it on:

| Option | Default | What it does |
| --- | --- | --- |
| `temporary_files` | on (safe profile) | Removes Windows and user Temp content older than `minimum_age_hours` (24) |
| `component_store` | off | DISM analysis, `StartComponentCleanup` and a second analysis |
| `windows_update` | off | Selects the same superseded-component cleanup; with `component_store` it runs once |
| `reset_component_base` | off | Adds `/ResetBase`. Irreversible: installed updates can no longer be uninstalled |
| `installer_cleanup` | off | Removes the files listed in `installer_paths` |
| `provisioning_artifacts` | off | Removes the files listed in `artifact_paths`, such as answer files and build secrets |
| `log_cleanup` | off | Removes the files listed in `log_paths` |
| `error_reporting` | off | Runs the installed Disk Cleanup handler for Windows Error Reporting |
| `empty_recycle_bin` | off | Empties the communicator user's Recycle Bin |
| `remove_user_profiles` | off | Removes build profiles listed by exact SID |
| `clear_event_logs` | off | Exports and then clears the channels you list; never Security |

Some things are never touched: WinSxS contents, the Windows Installer cache,
`SoftwareDistribution`, `catroot2`, servicing metadata, the Security event
log, activation state, firewall rules and the WinRM configuration. Windows
Update cleanup here means DISM removing superseded components, not resetting
Windows Update.

## The safety rules

Deletion only happens inside fixed roots that the guest works out itself:
Windows Temp, the Temp folder of real user profiles, and
`C:\ProgramData\PackerCleanup\Staging`. HCL can't add roots. The only
exception is a short list of exact files that `artifact_paths` may name: the
unattend files under Panther and Sysprep, and `OS.ready`. If an installer
leaves files somewhere else, move its staging into the staging root earlier
in the build.

Paths are taken literally. There's no environment variable or wildcard
expansion, and UNC paths, alternate data streams, `..` segments, DOS device
names and trailing dots are rejected. Every ancestor of a path is checked
for junctions and symbolic links, and a link is reported as a failure, not
followed.

Ordinary temp files must be older than the threshold, and locked files are
skipped with a warning. Files you list explicitly are different: they can be
recent, and a locked one fails the build, because a credential file that
silently stays behind is exactly what you don't want.

Profile removal only accepts exact `S-1-5-21` SIDs for unloaded profiles
under `C:\Users\packer-build-*`. It never guesses from inactivity, and it
refuses the current communicator profile, loaded profiles and system
profiles.

The plugin doesn't restart Windows. A pending reboot or an update in
progress stops the build by default, so put Packer's `windows-restart`
before it. `pending_reboot_policy = "skip_servicing"` skips servicing for a
pending reboot instead of failing; an active update always blocks cleanup.

One thing it can't do: deleting a credential file isn't secure erasure, and
it doesn't remove copies in snapshots or host logs. Use short-lived build
credentials and revoke them as part of the build.

## Getting started

The plugin is published on GitHub with signed checksums, so `packer init`
can install it. It needs an elevated, native 64-bit Windows PowerShell 5.1
session over WinRM, and it works on Server Core. The release has binaries
for Windows, Linux and macOS.

Put it last among the provisioners that change the guest, after
installation, updates, hardening and restarts. Start with `dry_run`:

```hcl
packer {
  required_plugins {
    windows-cleanup = {
      source  = "github.com/larskroghdk/windows-cleanup"
      version = ">= 0.1.0"
    }
  }
}

build {
  sources = ["source.azure-arm.windows"]

  provisioner "windows-restart" {}

  provisioner "windows-cleanup" {
    temporary_files        = true
    component_store        = true
    installer_cleanup      = true
    provisioning_artifacts = true
    installer_paths        = ["C:\\ProgramData\\PackerCleanup\\Staging\\installers"]
    artifact_paths         = ["C:\\Windows\\Temp\\build-secrets.json"]
    required_services      = ["W32Time"]
    dry_run                = true
    report_path            = "reports/windows-cleanup.json"
  }
}
```

```powershell
packer init .
packer build .
```

With `dry_run = true` nothing is deleted, staged or changed; the report
lists the candidates instead. Read it on a disposable VM, then remove
`dry_run`.

## Validation

`validate_system` is on by default. After cleanup it checks that no reboot
is pending, that updates have finished, that WinRM and RpcSs are running and
that whatever you list in `required_services`, `required_software_paths` and
`required_features` is in place. Software is checked by file path, not
`Win32_Product`, which would trigger repairs. Errors fail the build by
default. `fail_on_error = false` turns ordinary category failures into
warnings, but never for explicit artifacts, transport errors or the final
validation.

## The report

The report is written even when the build fails. It has the profile, the
dry-run flag, timing, OS facts and reboot reasons before and after,
detected capabilities and one entry per operation with its status, the
literal paths removed, warnings, bytes removed and native output such as
DISM's analysis.

It's careful not to overstate savings. The free-space delta on the system
drive is reported as observed and can be negative, because Windows writes
files at the same time. Removed bytes are file lengths, not allocated
blocks. Where Windows APIs don't return a per-file list, as for WER or the
Recycle Bin, the list is empty rather than invented.

The report contains file names and servicing diagnostics, so write it to an
access-controlled directory.

## Supported Windows versions

| Guest | Builds |
| --- | --- |
| Windows Server 2016, 2019, 2022, 2025 | 14393, 17763, 20348, 26100; Core and Desktop Experience |
| Windows 10 | 14393 and later |
| Windows 11 | 22000 and later |

Domain controllers, unknown server builds, 32-bit PowerShell sessions and
older client builds are rejected. Capabilities that are missing on a given
system, such as the WER Disk Cleanup handler on Server Core, fail the build
unless you set `unsupported_policy = "skip"`.

## Sysprep

`sysprep = true` only stages a finalizer script. The provisioner never runs
Sysprep or shuts down the VM itself, because WinRM disappears the moment
generalization starts. The builder's `shutdown_command` runs the finalizer,
which checks health, pending reboots, domain state and Appx packages, starts
`sysprep /generalize /oobe /quiet /shutdown` and returns. The builder then
waits for power-off.

In v0.1.0 this is only accepted with the QEMU builder. Other builders are
rejected. My
[Hyper-V Windows Server factory]({{< relref "posts/introducing-packer-hyperv-windows-server" >}})
uses the plugin with `sysprep = false` and its own finalizer for that
reason.

## Testing status

The Go tests cover configuration decoding, path validation, OS
classification, pending reboots, reports and the Sysprep staging rules.
Pester tests run real deletions in a test drive, including stale and fresh
files, manifests, exclusions, locks and junction refusal. They never clean
the development machine.

What hasn't been done yet is the live VM matrix: Server 2016 to 2025 on
Core and Desktop, Windows 10 and 11, real updates and a Sysprep clone
through OOBE. There's an opt-in integration test for it, but until it has
run, treat v0.1.0 as something to try on a disposable VM before you rely on
it.

## Links

- [GitHub repository](https://github.com/larskroghdk/packer-plugin-windows-cleanup)
- [Releases](https://github.com/larskroghdk/packer-plugin-windows-cleanup/releases)
- [Configuration reference](https://github.com/larskroghdk/packer-plugin-windows-cleanup/blob/main/docs/configuration.md)
- [Safety and reports](https://github.com/larskroghdk/packer-plugin-windows-cleanup/blob/main/docs/safety.md)
- [Issues](https://github.com/larskroghdk/packer-plugin-windows-cleanup/issues)
- [HashiCorp Packer documentation](https://developer.hashicorp.com/packer)
