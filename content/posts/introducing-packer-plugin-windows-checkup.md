---
title: "Introducing packer-plugin-windows-checkup: Auditing Windows Security Baselines in Packer Builds"
description: "An independent, read-only audit of the Microsoft security baseline on a Windows image, run as a Packer provisioner with a JSON report and build gates."
date: 2026-11-23
draft: false
tags: ["packer", "windows", "security", "automation", "compliance"]
categories: ["Infrastructure Automation"]
keywords: ["packer-plugin-windows-checkup", "packer windows compliance", "windows security baseline audit", "security compliance toolkit audit", "windows image compliance report"]
---

[packer-plugin-windows-checkup](https://github.com/larskroghdk/packer-plugin-windows-checkup)
v0.1.0 is released: a Packer provisioner that audits a Windows guest against
the Microsoft security baseline for its release and writes a JSON report on
the Packer host. It only reads. It never applies policy, changes a service,
touches the firewall or restarts Windows.

<!--more-->

## 💡 Why a separate audit

[packer-plugin-windows-harden]({{< relref "posts/introducing-packer-plugin-windows-harden" >}})
already verifies every setting it applies and writes a report. So why
another plugin?

Because a tool that changes something shouldn't be the only one that tells
you the change worked. If harden has a bug in how it reads a value, the same
bug can show up in its own verification. Checkup has its own readers, its
own comparison logic and no runtime or module dependency on harden. It
doesn't read harden's report, logs or binaries. It looks at the machine and
compares what it finds with the baseline.

That also means it works on its own. You can point it at an image hardened
by Group Policy, a script or by hand, and get the same report.

## 🔍 What it checks

The baselines are an exact, byte-for-byte snapshot of the six baseline files
in windows-harden, pinned to a source commit and with a SHA-256 for each
file. A test fails if the data changes unnoticed.

| System | Build | Baseline | Controls |
| --- | ---: | --- | ---: |
| Windows Server 2019 member server | 17763 | `ws2019-member` | 302 |
| Windows Server 2022 | 20348 | `ws2022` | 303 |
| Windows Server 2025 | 26100 | `ws2025-2602` | 347 |
| Windows 11 24H2 | 26100 | `win11-24h2` | 412 |
| Windows 11 25H2 | 26200 | `win11-25h2` | 411 |
| Windows 11 26H2 | 26300 | `win11-26h2` | 412 |

The baseline and the domain profile are selected automatically from the OS
and domain membership. Domain controllers, Windows client LTSC editions and
releases past end of support are rejected with an execution error, and no
compliance is claimed for them.

Each control is read directly: registry values and their types, local
account policy, user rights as SID sets, advanced audit subcategories and
service startup types. Security and audit policy are exported with
`secedit.exe` and `auditpol.exe` into a temporary directory, which is
removed afterwards.

The audit is strict about what counts as compliant:

- A missing registry value is `NotConfigured`, even where Windows has a
  similar default. The baseline expects explicit policy.
- A value is only absent-and-compliant when the baseline explicitly
  requires `ABSENT`.
- A service is `NotApplicable` only after the reader has verified that its
  service key doesn't exist.
- A failed, malformed or missing read is an `Error`, never a pass.

Two limits are worth stating plainly. The audit checks configured policy,
not runtime health: a Defender policy can be set while Defender itself is
broken. And a compliant image doesn't prove that domain Group Policy won't
change things after it joins a domain.

## 🚀 Getting started

The plugin is published on GitHub with signed checksums, so `packer init`
can install it. It needs an elevated, 64-bit Windows PowerShell 5.1 session
over WinRM. The release has binaries for Windows, Linux and macOS, so
Packer can run on any of them.

A natural place for it is right after windows-harden, as an independent
check before the image is captured:

```hcl
packer {
  required_plugins {
    windows-harden = {
      source  = "github.com/larskroghdk/windows-harden"
      version = ">= 0.1.0"
    }
    windows-checkup = {
      source  = "github.com/larskroghdk/windows-checkup"
      version = ">= 0.1.0"
    }
  }
}

build {
  sources = ["source.azure-arm.windows"]

  provisioner "windows-harden" {
    reboot_if_required = true
    report_path        = "reports/ws2025-harden.json"
  }

  provisioner "windows-checkup" {
    report_path                   = "reports/ws2025-checkup.json"
    fail_on_error                 = true
    minimum_compliance_percentage = 95
  }
}
```

```powershell
packer init .
packer build .
```

The repository also has a complete
[null-builder example](https://github.com/larskroghdk/packer-plugin-windows-checkup/tree/main/examples/local)
that audits an existing VM, which is the quickest way to see what a report
looks like.

## ⛔ Failing a build on the result

By default checkup only reports. Four options turn the report into a gate:

| Option | Default | Effect |
| --- | --- | --- |
| `fail_on_noncompliance` | `false` | Fail on any `NonCompliant` or `NotConfigured` control at or above `severity_threshold` |
| `minimum_compliance_percentage` | `0` | Fail below this percentage; `0` disables the check |
| `fail_on_error` | `false` | Fail on read errors and execution failures, including an unsupported OS |
| `severity_threshold` | `Low` | Filters which controls `fail_on_noncompliance` considers |

The percentage is `Compliant / (Compliant + NonCompliant + NotConfigured)`.
Errors aren't in the denominator, which is why `fail_on_error = true`
belongs in any strict build: otherwise a control that couldn't be read
simply disappears from the calculation.

Microsoft's baselines carry no severity ratings, so every control is
labeled `Medium`. That is a project convention, not an upstream rating, and
it has a consequence: `severity_threshold = "High"` currently excludes every
control from `fail_on_noncompliance`.

After windows-harden, don't expect 100 %. Harden deliberately defers
settings that would cut off Packer's WinRM session, such as disabling Basic
authentication when the build connects with it. Checkup doesn't know or
care why a setting is missing, so it reports those as noncompliant. That is
the point of an independent audit. A percentage threshold, or a review of
exactly which controls fail, works better than `fail_on_noncompliance`
there.

## 📊 The report

The report has the plugin and baseline version, the SHA-256 of the exact
baseline file used, the source commit, the domain profile, timing, totals
per category and severity, execution errors, and one finding per control in
stable ID order. Each finding carries the expected and actual value and
type, the read evidence and a remediation hint. There's a versioned
[JSON schema](https://github.com/larskroghdk/packer-plugin-windows-checkup/blob/main/docs/report.schema.json)
and a
[sample report](https://github.com/larskroghdk/packer-plugin-windows-checkup/blob/main/docs/sample-report.json).

```powershell
$report = Get-Content .\reports\ws2025-checkup.json -Raw | ConvertFrom-Json
$report.summary | Select-Object total, compliant, nonCompliant, notConfigured, errors, compliancePercentage
$report.findings |
    Where-Object status -ne 'Compliant' |
    Select-Object id, status, expectedValue, actualValue
```

The report is written to a temporary file next to the target and renamed
into place, so a pipeline never reads half a report. It contains no
credentials or session data, and raw guest exception text is left out.

## 🧪 Testing status

Every one of the 2,187 controls and the 18 non-domain-joined overrides is
exercised in tests for matching values, mismatches, missing configuration,
missing reader data and reader errors. The guest scripts have Pester tests,
and the example passes `packer validate` with Packer 1.16.2.

What hasn't been run yet is a live audit against real Windows VMs. There's
an opt-in integration test for it, but a green unit test run is not evidence
of a working audit on every OS. Try it on a disposable VM first.

## 🔗 Links

- 📂 [GitHub repository](https://github.com/larskroghdk/packer-plugin-windows-checkup)
- 🏷️ [Releases](https://github.com/larskroghdk/packer-plugin-windows-checkup/releases)
- 📖 [Complete control inventory](https://github.com/larskroghdk/packer-plugin-windows-checkup/blob/main/docs/control-inventory.md)
- 🐛 [Issues](https://github.com/larskroghdk/packer-plugin-windows-checkup/issues)
- 📘 [HashiCorp Packer documentation](https://developer.hashicorp.com/packer)
