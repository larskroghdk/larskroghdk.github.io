---
title: "Introducing packer-plugin-windows-harden: Microsoft Security Baselines in Your Packer Builds"
description: "Applying the Microsoft Security Compliance Toolkit baseline to Windows images during Packer builds, without cutting off Packer's own WinRM session."
date: 2026-11-19
draft: false
tags: ["packer", "windows", "security", "automation", "vmware", "azure"]
categories: ["Infrastructure Automation"]
keywords: ["packer-plugin-windows-harden", "packer windows hardening", "microsoft security baseline", "security compliance toolkit", "windows image hardening"]
---

Today I'm making
[packer-plugin-windows-harden](https://github.com/larskroghdk/packer-plugin-windows-harden)
public: a Packer provisioner that applies the Microsoft security baseline
for the guest's Windows release while the image is being built. The source
is available now. The first tagged release, v0.1.0, is being prepared and
will be installable with `packer init`.

<!--more-->

## The problem with hardening inside a build

Microsoft publishes security baselines for every supported Windows release
in the Security Compliance Toolkit: GPO backups, a spreadsheet, and a
script for applying them to a local machine. Getting them into an image usually means unpacking the
zip in a provisioner and running the script.

That works until the baseline does its job. It disables Basic
authentication and unencrypted traffic on the WinRM service, turns on the
firewall, tightens NTLM and adjusts remote UAC. Packer is connected over
WinRM while all of this happens. Depending on how the build connects, the
next provisioner simply can't reach the machine, and the build times out
with a half-hardened VM.

The usual workarounds are to apply the baseline at first boot instead, or
to maintain a hand-edited copy with the dangerous settings removed. The
first moves security out of the image. The second drifts from Microsoft's
baseline every time a new version ships.

## What it does

The plugin registers one provisioner, `windows-harden`. It detects the OS,
edition, domain membership, the WinRM configuration and the identity of the
Packer session, selects the matching baseline, and plans every setting
before it changes anything:

- **apply**: the setting can't affect the Packer session, so it's applied
  and verified.
- **defer**: the setting would break the session, so it's left alone and
  reported, with a reason.
- **guard**: the effect can't be determined from inside the guest, so the
  setting is applied under an automatic rollback.

Deferred settings aren't dropped. The report contains a `deferred_plan` with
their full definitions, so you can apply them at first boot or in your
deployment pipeline once Packer no longer needs WinRM. The plugin doesn't
apply that plan for you. If you'd rather not ship an image with deferred
settings at all, `mode = "strict"` fails the build before any change is
made.

## Baselines as data

The baselines aren't hand-maintained. A generator in the repository,
`cmd/sctimport`, reads the Security Compliance Toolkit zip files and writes
one JSON baseline per release, which is embedded in the plugin. Each source
is pinned to its `download.microsoft.com` URL and SHA-256 checksum, and the
report records which package a build used.

Every GPO in a package is either included or explicitly excluded with a
reason, and the generator fails on anything it can't represent. Two kinds
of components are excluded today:

- **User Configuration policies**: the plugin configures computer policy
  below HKLM only.
- **Credential Guard**: it needs virtualization-based security support in
  the virtual hardware and a restart, so it's left to Group Policy on
  capable hosts after deployment.

The report lists these exclusions in `baseline.excluded`.
`compliance.full_baseline` is only `true` when nothing was excluded,
deferred or skipped. Every current baseline excludes Credential Guard, so
today it is always `false`. I'd rather report that honestly than call an
image fully compliant when it isn't.

## How it keeps the session alive

The protection rules are specific. A few examples:

| Setting | Deferred when |
| --- | --- |
| WinRM `AllowBasic = 0` | Basic authentication is enabled, or the WinRM configuration couldn't be read |
| WinRM `AllowUnencryptedTraffic = 0` | Packer is, or may be, connected over an HTTP listener with unencrypted traffic allowed |
| `SeDenyNetworkLogonRight` | It names a SID in the Packer session token |
| `EnableLUA`, `LocalAccountTokenFilterPolicy`, `FilterAdministratorToken` | Together they would give the Packer account a filtered token on network logons |

When the information needed to decide is missing, the setting is deferred.
The rules fail closed.

Two settings, `LmCompatibilityLevel` and `NtlmMinServerSec`, are guarded
when Negotiate authentication is enabled, because whether the Packer client
meets the new NTLM requirements can't be seen from the guest. The plugin
snapshots their current values, registers a one-time SYSTEM scheduled task
that restores them after three minutes, applies them, and then tries new
WinRM commands for up to 90 seconds. If WinRM still works, the task is
removed. If it doesn't, the scheduled task puts the old values back and the
plugin reconnects. Settings that had to be rolled back are reported as
`deferred`, or as `failed` if the original value couldn't be confirmed.

After every change set, a new WinRM command must run, and the service,
listeners, authentication methods and unencrypted setting must be no weaker
than before. Otherwise the build fails.

## Getting started

The communicator must be `winrm`, with an elevated administrator session.
The release build targets Windows, Linux and macOS, so Packer can run on
any of them.

Until v0.1.0 is on the
[Releases](https://github.com/larskroghdk/packer-plugin-windows-harden/releases)
page, build it from source. You need Go 1.26:

```powershell
git clone https://github.com/larskroghdk/packer-plugin-windows-harden.git
cd packer-plugin-windows-harden
go build -o packer-plugin-windows-harden.exe .
packer plugins install --path packer-plugin-windows-harden.exe github.com/larskroghdk/windows-harden
```

A local build reports version `0.1.0-dev`, which a `version = ">= 0.1.0"`
constraint doesn't accept, so leave `windows-harden` out of
`required_plugins` while you use it. A dry run first, then the real run:

```hcl
build {
  sources = ["source.azure-arm.windows"]

  provisioner "windows-harden" {
    dry_run     = true
    report_path = "reports/ws2025-plan.json"
  }

  provisioner "windows-harden" {
    baseline           = "ws2025-2602"
    exclude            = ["registry:HKLM\\Software\\Microsoft\\Windows\\CurrentVersion\\Policies\\System\\InactivityTimeoutSecs"]
    reboot_if_required = true
    report_path        = "reports/ws2025.json"
  }
}
```

`baseline` defaults to `auto`. Pinning it makes the build fail if the image
turns out to be a different release than you expected. Note the escaped
backslashes in the setting ID: HCL rejects a single backslash.
`reboot_if_required` restarts after security policy changes, which only
take full effect after a restart, and re-verifies every setting.

The repository has complete templates for
[azure-arm](https://github.com/larskroghdk/packer-plugin-windows-harden/tree/main/examples/azure-arm)
(a Marketplace image, hardened and generalized to a managed image) and
[vsphere-iso](https://github.com/larskroghdk/packer-plugin-windows-harden/tree/main/examples/vsphere-iso)
(an unattended install converted to a template), with one variable file per
baseline. Both pin the release in `required_plugins`. Remove that entry
while you use a source build:

```powershell
packer init .
packer build -var-file=ws2025-2602.pkrvars.hcl -var-file=site.pkrvars.hcl .
```

The vSphere example is a good illustration of deferral. Its unattended
install enables WinRM over HTTP with Basic authentication so Packer can
connect. The baseline disables both, the plugin defers them, and the
template keeps them enabled until `deferred_plan` is applied.

Reviewing the result takes a few lines of PowerShell:

```powershell
$report = Get-Content .\reports\ws2025.json -Raw | ConvertFrom-Json
$report.compliance
$report.settings |
    Where-Object status -in 'deferred', 'failed' |
    Select-Object id, status, reason
```

`compliance.status` is `compliant`, `partial` (nothing failed, but
something was deferred or skipped) or `non_compliant`, and
`compliance.reasons` explains why.

## Supported Windows versions

| Baseline ID | OS | Microsoft baseline | Settings |
| --- | --- | --- | --- |
| `ws2019-member` | Windows Server 2019 member server | 1809 FINAL | 302 |
| `ws2022` | Windows Server 2022 member server | FINAL | 303 |
| `ws2025-2602` | Windows Server 2025 member server | 2602 | 347 |
| `win11-24h2` | Windows 11 24H2 | 24H2 | 412 |
| `win11-25h2` | Windows 11 25H2 | 25H2 | 411 |
| `win11-26h2` | Windows 11 26H2 | 26H2 v2 | 412 |

Settings cover registry policy, user rights, account and audit policy and,
on Windows 11, a few services. Security policy is applied with
`secedit.exe` and audit policy with `auditpol.exe`.

The plugin also checks the release lifecycle. Domain controllers, LTSC
editions and releases past end of support are rejected. For Windows 11 that
depends on the edition: 24H2 Home and Pro reached end of support in October
2026, so 24H2 is now only accepted on Enterprise, Education and IoT
Enterprise editions.

## Testing and what's next

The baseline model and catalog, the generator's workbook parsing, OS
classification and the protection and verification logic have Go unit
tests, and the guest scripts have Pester tests. There are no automated tests yet for the provisioner's
orchestration, the report, or a full Packer build against a real VM, so a
green test run doesn't prove an end-to-end build. That's the most important
gap, and the reason to start with `dry_run = true` on a disposable VM.

There is no published roadmap. New Microsoft baseline versions go through
the generator rather than hand edits, and releases without a baseline today
(Windows Server 2016, Windows 10, LTSC) are rejected rather than hardened
with a baseline that doesn't match.

## Pairs with windows-optimize

This plugin and
[packer-plugin-windows-optimize]({{< relref "posts/introducing-packer-plugin-windows-optimize" >}})
share an architecture: Go makes every decision, small PowerShell scripts
read and write state on the guest, and every change is verified and checked
against WinRM. They split the work cleanly. Optimize never touches security
controls, and harden only applies Microsoft's baseline.

- [GitHub repository](https://github.com/larskroghdk/packer-plugin-windows-harden)
- [Releases](https://github.com/larskroghdk/packer-plugin-windows-harden/releases)
- [Issues](https://github.com/larskroghdk/packer-plugin-windows-harden/issues)
- [HashiCorp Packer documentation](https://developer.hashicorp.com/packer)
