---
title: "Introducing packer-plugin-windows-features: Windows Roles, Features and Capabilities in Packer Builds"
description: "Declaring Windows Server roles, client optional features and capabilities in a Packer template, with inventory-based verification, restarts and a JSON report."
date: 2026-11-23
draft: false
tags: ["packer", "windows", "automation", "devops", "iis", "hyper-v"]
categories: ["Infrastructure Automation"]
keywords: ["packer-plugin-windows-features", "packer windows features", "install-windowsfeature packer", "enable-windowsoptionalfeature packer", "windows capability packer"]
---

[packer-plugin-windows-features](https://github.com/larskroghdk/packer-plugin-windows-features)
v0.1.0 is released: a Packer provisioner that installs and removes Windows
Server roles and features, Windows client optional features and Windows
capabilities during an image build. You list what you want, and the plugin
plans the changes, applies them, restarts if needed and checks the result
against a fresh inventory.

<!--more-->

## 💡 The problem with feature scripts

Adding IIS or .NET Framework 3.5 to an image usually starts as one line of
PowerShell in a provisioner. Then the template has to work on Server Core
as well as Desktop Experience, on Windows 11 as well as Server, from
installation media because the build network has no Windows Update, and
through the restart that half of these changes want. The one line becomes a
script that parses `Install-WindowsFeature` results, decides whether to
restart and hopes the exit code means what it says.

A successful exit code isn't the same as an installed feature. Servicing
can report success and leave the feature pending a restart, or install it
without one of the role services you assumed would come along.

## 🧰 What it does

The plugin registers one provisioner, `windows-features`. A run goes like
this:

1. Detect the OS, edition, installation type, elevation and WinRM
   configuration, and pick the servicing interface: Server Manager on
   Windows Server, DISM optional features on Windows 10 and 11.
2. Read the full feature and capability inventory.
3. Resolve every requested name, skip the ones already in the desired state,
   and stop on anything protected, contradictory or unknown.
4. Finish a pending servicing restart first, if one is blocking.
5. Remove, then install.
6. Read the inventory again and verify every request against it.
7. Open a new WinRM command and compare the session with the baseline.
8. Restart if required, wait for the machine to come back, and verify
   again.

All decisions are made in Go. The guest only runs small PowerShell 5.1
scripts, and requests are passed in JSON files, so no feature name is ever
interpolated into a PowerShell command line. The plugin doesn't parse DISM's
console output either; it works on the structured objects the servicing
cmdlets return.

A second run finds everything compliant and changes nothing.

## 🛡️ Safety

Removing the wrong feature can take the build down with it, so some
removals are blocked in code:

- **Runtime:** PowerShell, PowerShellRoot, .NET Framework 4.x and WoW64 on
  Server, and NetFx4-AdvSrvs on client. Server Manager features known to
  depend on them are blocked too.
- **Networking and authentication:** DNS, DHCP, Remote Access, Routing,
  DirectAccess-VPN, AD Domain Services and Hyper-V on Server, and the
  Hyper-V core features on client. These can carry the network path or the
  authentication of the current session.

`fail_on_error = false` can't override these. After every change and every
restart, a new command must run and the WinRM service, listeners,
authentication and elevation must match what they were before. A
connectivity failure always fails the build.

That is a conservative guard, not a complete model of what every role does
to the network. Installing Hyper-V or Remote Access can still change the
networking a build relies on. Test those on a disposable VM with console
access.

Removal also has no rollback. Getting a removed feature back may need
installation media, so the plugin doesn't pretend it can undo it.

## 🚀 Getting started

The plugin is published on GitHub with signed checksums. The release has
binaries for Windows, Linux and macOS, so Packer can run on any of them. The
guest needs an elevated, 64-bit Windows PowerShell 5.1 session over WinRM.

```hcl
packer {
  required_plugins {
    windows-features = {
      source  = "github.com/larskroghdk/windows-features"
      version = ">= 0.1.0"
    }
  }
}

build {
  sources = ["source.hyperv-iso.ws2025"]

  # Plan only: inventory and validate, no servicing.
  provisioner "windows-features" {
    install     = ["Web-Server", "Web-Mgmt-Tools", "Web-Mgmt-Console"]
    remove      = ["Print-Server"]
    dry_run     = true
    report_path = "reports/ws2025-features-plan.json"
  }

  provisioner "windows-features" {
    install                  = ["Web-Server", "Web-Mgmt-Tools", "Web-Mgmt-Console"]
    remove                   = ["Print-Server"]
    include_management_tools = true
    restart_if_required      = true
    report_path              = "reports/ws2025-features.json"
  }
}
```

On Windows 10 and 11 the names are optional feature names. For .NET
Framework 3.5 from installation media, without contacting Windows Update:

```hcl
provisioner "windows-features" {
  install             = ["NetFx3"]
  source              = "D:\\sources\\sxs"
  limit_access        = true
  restart_if_required = true
}
```

Capabilities have their own lists, because a feature name and a capability
identity aren't interchangeable:

```hcl
provisioner "windows-features" {
  install_capabilities = ["OpenSSH.Server"]
}
```

The short identity resolves only if the guest has exactly one matching
version. That installs OpenSSH Server; it doesn't configure or start `sshd`
or open the firewall.

Names must be exact, ignoring case. Wildcards are rejected, and a name that
appears in both `install` and `remove` fails at `packer validate`. If a
template has to run on editions where a feature doesn't exist,
`ignore_unavailable = true` turns a missing name into a warning instead of
an error.

The repository has
[seven example templates](https://github.com/larskroghdk/packer-plugin-windows-features/tree/main/examples),
one per purpose: IIS, .NET 3.5 and Hyper-V on Server, .NET 3.5, Hyper-V and
an optional feature on client, and OpenSSH as a capability. They all
default to `dry_run = true`.

## 🔄 Restarts

`restart_if_required` is `false` by default. When it's on, the plugin
suppresses the servicing cmdlets' own restarts and issues one itself,
through the communicator Packer already has. It waits until the boot time
has changed and stayed stable for 30 seconds, checks WinRM and elevation,
and reads the inventory again. Each wait is bounded by `restart_timeout`
(30 minutes by default), with at most three restarts to complete the
changes.

Without it, a feature that needs a restart is reported as
`pending_reboot`. That is not the same as usable. Either enable restarts or
add a `windows-restart` provisioner and run the plugin again to verify.

## 📊 The report

The JSON report is written for successful and failed runs. It contains the
detected system, a summary, the state of every request, side effects
(Server Manager may remove dependent role services along with a feature you
removed), the connectivity checks, the restart state, and a `validation`
block.

```powershell
$report = Get-Content .\reports\ws2025-features.json -Raw | ConvertFrom-Json
$report.summary
$report.requests | Select-Object requested, kind, action, status, reason
$report.side_effects

if (-not $report.validation.passed -or $report.reboot.required) {
    throw "Features not in the desired state: $($report.validation.failing -join ', ')"
}
```

Check `validation.passed`, not just the Packer exit code. With
`fail_on_error = false` the build can succeed while a request failed.

## 🖥️ Supported Windows versions

| OS | Mechanism |
| --- | --- |
| Windows Server 2016, 2019, 2022, 2025 (Core and Desktop Experience) | Server Manager |
| Windows 10, build 14393 and later | DISM optional features |
| Windows 11 | DISM optional features |

Capabilities work on both, where the cmdlets exist. Newer builds are
accepted with a warning. Nano Server, older releases, 32-bit PowerShell and
non-elevated sessions are rejected. What you can install still depends on
edition, hardware and payload: Hyper-V needs a supported edition and, in a
VM, nested virtualization.

## 🧪 Testing status

Go unit tests and a fake guest cover planning, installation and removal,
idempotency, source errors, partial failures, timeouts, pending restarts
and reconnects. The servicing commands have Pester tests under Windows
PowerShell 5.1 with the cmdlets mocked.

No live servicing run on Windows Server or Windows 10/11 has been recorded
yet, and ARM64 is cross-compiled but not executed. Those are listed as open
items in the
[support documentation](https://github.com/larskroghdk/packer-plugin-windows-features/blob/main/docs/support.md).
Start with `dry_run = true` on a disposable VM.

## 🔗 Links

- 📂 [GitHub repository](https://github.com/larskroghdk/packer-plugin-windows-features)
- 🏷️ [Releases](https://github.com/larskroghdk/packer-plugin-windows-features/releases)
- 📖 [Configuration reference](https://github.com/larskroghdk/packer-plugin-windows-features/blob/main/docs/configuration.md)
- 📖 [Troubleshooting](https://github.com/larskroghdk/packer-plugin-windows-features/blob/main/docs/troubleshooting.md)
- 🐛 [Issues](https://github.com/larskroghdk/packer-plugin-windows-features/issues)
- 📘 [HashiCorp Packer documentation](https://developer.hashicorp.com/packer)
