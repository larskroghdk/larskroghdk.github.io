---
title: "Introducing packer-plugin-windows-optimize: Automated Windows Optimization for HashiCorp Packer"
description: "Automating Microsoft Windows optimization during VM template builds with HashiCorp Packer."
date: 2026-11-19
draft: false
tags: ["packer", "windows", "automation", "vmware", "azure", "devops"]
categories: ["Infrastructure Automation"]
keywords: ["packer-plugin-windows-optimize", "packer windows plugin", "windows vm template optimization", "hashicorp packer windows optimization"]
---

Today I'm making
[packer-plugin-windows-optimize](https://github.com/larskroghdk/packer-plugin-windows-optimize)
public: a Packer provisioner that applies a small, validated set of Windows
optimizations while a VM template is being built. The source is available
now and builds with a single `go build`. The first tagged release, v1.0.0,
is being prepared and will be installable with `packer init`.

<!--more-->

## 💡 Why I built it

Every Windows template pipeline I have worked on ended up with the same
folder: PowerShell scripts that disable hibernation, stop Server Manager from
opening at logon, set power timeouts, turn off a few services. Some were
written for a single Windows Server release and carried forward to newer
ones, and nobody could say with certainty which ones still did anything,
which ones were safe on Server Core, or how to undo them. When I evaluated
our own set for this plugin, many of them turned out to configure the image
rather than optimize it, and one configured the first run of Internet
Explorer, which is retired.

Whether the template ends up as a VMware template or an Azure image, the
questions are the same: what did we change, does it apply to this build of
Windows, and can it break the build? I wanted those answers in code, inside
the Packer run, instead of in a script folder and someone's memory.

## 🧰 What it is

`packer-plugin-windows-optimize` registers one provisioner,
`windows-optimize`. It runs over the WinRM communicator, so it isn't tied to
a hypervisor or cloud builder. The repository ships examples for the
`hyperv-vmcx` builder and for the `null` builder against an existing
machine.

It is deliberately narrow. It doesn't install updates, remove apps or harden
the OS. It runs after your update and configuration provisioners and before
sysprep, and it leaves security controls alone. Defender, Windows Update,
event logging, the firewall and authentication settings are protected in
code and can't be selected by any profile or `include` pattern.

## ✨ Key features

- **OS detection.** The plugin reads the build number, edition, installation
  type and architecture from the registry and WMI, not from `ProductName`,
  which still says "Windows 10" on Windows 11.
- **Rules per OS.** Each of the 31 optimizations in the
  [catalog](https://github.com/larskroghdk/packer-plugin-windows-optimize/blob/main/docs/catalog.md)
  declares which installation types, builds and editions it applies to.
  Anything else is reported as `unsupported`, so one profile works on Server
  Core, Desktop Experience and client.
- **Profiles.** `recommended` (the default) contains only changes that are
  documented by Microsoft and need no restart. `performance` and `vdi` build
  on it, and `custom` applies only what you list in `include`.
- **WinRM protection.** The services WinRM depends on are read from the guest
  and never touched. After applying, the plugin opens a new WinRM command and
  checks that the service, listeners and authentication methods are
  unchanged. A regression fails the build, even with `fail_on_error = false`.
- **Rollback journal.** The original value of every target is written to a
  journal on the guest *before* anything changes. `rollback = true` restores
  it. The journal survives sysprep, so deployed machines can be rolled back
  too.
- **Verification and idempotency.** Every change is read back before it
  counts as applied. A second run finds everything compliant and changes
  nothing.
- **Dry run and report.** `dry_run = true` writes the plan without touching
  the guest. Every run, including a failed one, writes a JSON report.

## ⚙️ How it works

The decisions are made in Go. The guest only runs small PowerShell 5.1
scripts that read or write a single target, with JSON files for input and
output. A build goes through these steps:

1. Packer starts the VM, and your earlier provisioners install updates and
   applications.
2. The plugin uploads its scripts and detects the OS, the elevated session,
   the WinRM configuration and the power schemes. An unsupported OS, a domain
   controller or a non-elevated session stops the run here.
3. It builds a plan from the profile, removes what doesn't apply to this OS
   and skips anything that would touch a protected target.
4. It inspects every target and records the originals in the journal.
5. It applies the changes, reads them back and checks WinRM.
6. If a change needs a restart and `reboot_if_required = true`, it restarts,
   waits for WinRM and verifies everything again.
7. Packer continues, typically with sysprep through `shutdown_command`.

The plugin never restarts on its own. Without `reboot_if_required`, the
report tells you a restart is still needed.

## 🚀 Getting started

The plugin runs with Packer on Windows, and the release build targets
Windows amd64 and arm64. The communicator must be `winrm`, with an elevated
administrator session.

Until v1.0.0 is on the
[Releases](https://github.com/larskroghdk/packer-plugin-windows-optimize/releases)
page, build it from source. You need Go 1.26:

```powershell
git clone https://github.com/larskroghdk/packer-plugin-windows-optimize.git
cd packer-plugin-windows-optimize
go build -o packer-plugin-windows-optimize.exe .
packer plugins install --path packer-plugin-windows-optimize.exe github.com/larskroghdk/windows-optimize
```

A local build reports version `1.0.0-dev`, which a `version = ">= 1.0.0"`
constraint doesn't accept. Leave `windows-optimize` out of
`required_plugins` while you use it. The installed plugin is still found by
its provisioner name:

```hcl
build {
  sources = ["source.hyperv-vmcx.windows"]

  # Plan only: shows what would change.
  provisioner "windows-optimize" {
    profile     = "performance"
    dry_run     = true
    report_path = "reports/ws2022-plan.json"
  }

  provisioner "windows-optimize" {
    profile            = "performance"
    exclude            = ["power.high_performance_scheme"]
    reboot_if_required = true
    report_path        = "reports/ws2022.json"
  }
}
```

Then run:

```powershell
packer init .
packer validate .
packer build -var-file=ws2022.pkrvars.hcl -var "winrm_password=$env:WINRM_PASSWORD" .
```

`packer init` installs the builder plugins your template declares.
Configuration is checked in `packer validate`. A typo in an `include` or
`exclude` pattern matches no optimization, and that is an error rather than a
silent no-op. The complete Hyper-V template is in
[examples/hyperv](https://github.com/larskroghdk/packer-plugin-windows-optimize/tree/main/examples/hyperv).
It already pins the release, so remove its `windows-optimize` entry from
`required_plugins` while you use a source build.

Once v1.0.0 is released, pin it instead and let `packer init` download and
checksum-verify it:

```hcl
packer {
  required_plugins {
    windows-optimize = {
      source  = "github.com/larskroghdk/windows-optimize"
      version = ">= 1.0.0"
    }
  }
}
```

The report is ordinary JSON, so reviewing it, or failing a pipeline on it,
takes a few lines of PowerShell:

```powershell
$report = Get-Content .\reports\ws2022.json -Raw | ConvertFrom-Json
$report.summary
$report.optimizations |
    Where-Object status -in 'skipped', 'failed' |
    Select-Object id, status, reason

if ($report.summary.failed -gt 0) { throw 'Optimization failed' }
```

Each optimization in the report lists its targets with the `before`,
`expected` and `after` values, and links to the documentation it is based
on.

## ⏱️ Why optimize during template creation?

A setting applied in the template exists on every machine from the first
boot. A setting applied after deployment depends on a GPO, a configuration
management run or a script that may not have happened yet, and every new VM
repeats the same work.

Putting it in the Packer template also puts it in version control. The
profile and the exclusions sit in the same HCL as the rest of the build,
get reviewed in the same pull requests and produce a report with every
image. When someone asks why Server Manager doesn't open on a new server,
the answer is an optimization ID and a link, not an archaeology project.

I make no claims about boot times or benchmark numbers. The point is a
known, documented and reversible starting state, not a faster number on a
graph.

## 🖥️ Supported Windows versions

| OS | Build | Installation types |
| --- | --- | --- |
| Windows Server 2016 | 14393 | Desktop Experience, Server Core |
| Windows Server 2019 | 17763 | Desktop Experience, Server Core |
| Windows Server 2022 | 20348 | Desktop Experience, Server Core |
| Windows Server 2025 | 26100 | Desktop Experience, Server Core |
| Windows 10 | 14393 and later | 64-bit client editions |
| Windows 11 | 22000 and later | 64-bit client editions |

Newer builds are accepted with a warning, so a feature update doesn't break
existing templates. Domain controllers, Nano Server and older releases are
rejected.

To be precise about what "supported" means today: the plugin is covered by
Go unit tests, Pester tests of the guest scripts on Windows PowerShell 5.1,
and end-to-end tests against an in-memory fake Windows Server 2022 guest.
The read-only guest actions have run on Windows 11 25H2. A live apply and
rollback against a real VM hasn't been run yet. ARM64 is accepted but
untested. The
[testing documentation](https://github.com/larskroghdk/packer-plugin-windows-optimize/blob/main/docs/testing.md)
lists exactly what has been verified.

## 🗺️ Roadmap

There are no dates attached, but the
[research notes](https://github.com/larskroghdk/packer-plugin-windows-optimize/blob/main/docs/research.md#remaining-opportunities)
list the candidates that still need validation:

- Microsoft Edge and OneDrive update tasks, which change name and folder
  between versions and need detection by publisher rather than by path.
- A Storage Sense policy for persistent desktops.
- Windows Search on session hosts, as an explicit-only, high-risk
  optimization.
- Measured benchmarks per optimization (boot time, idle CPU, disk I/O) on
  each supported OS, to replace the qualitative impact descriptions with
  numbers.

Beyond that, the open items are the testing gaps above: a live apply and
rollback on real VMs, and ARM64. The catalog will stay small. A new
optimization has to be documented by Microsoft, reversible, verifiable and
safe for WinRM.

## 🔗 Try it

Start with `dry_run = true` on a disposable VM, read the report, and then
decide what to apply. If you build vSphere or Azure templates with it, I'd
like to hear how it behaves.

- 📂 [GitHub repository](https://github.com/larskroghdk/packer-plugin-windows-optimize)
- 🏷️ [Releases](https://github.com/larskroghdk/packer-plugin-windows-optimize/releases)
- 🐛 [Issues](https://github.com/larskroghdk/packer-plugin-windows-optimize/issues)
- 📘 [HashiCorp Packer documentation](https://developer.hashicorp.com/packer)
