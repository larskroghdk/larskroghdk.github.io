---
title: "Validating Terraform Variables Before Plan"
description: "Use validation blocks to reject bad input early, and test the rules with terraform test."
date: 2026-11-18T09:00:00+01:00
draft: false
tags: ["terraform", "hcl", "testing", "ci-cd"]
categories: ["Infrastructure as Code"]
---

A typo in a variable should fail in seconds, not halfway through an apply.
Terraform's `validation` blocks run before `plan` touches any provider.

<!--more-->

## A validated variable

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment."

  validation {
    condition     = contains(["dev", "test", "prod"], var.environment)
    error_message = "environment must be one of: dev, test, prod."
  }
}

variable "storage_account_name" {
  type = string

  validation {
    condition     = can(regex("^[a-z0-9]{3,24}$", var.storage_account_name))
    error_message = "Use 3-24 lowercase letters and digits."
  }
}
```

## Testing the rules

`terraform test` can assert that invalid input is rejected:

```hcl
run "rejects_unknown_environment" {
  command = plan

  variables {
    environment          = "staging"
    storage_account_name = "stlarskrogh01"
  }

  expect_failures = [var.environment]
}
```

Run it with:

```bash
terraform init -backend=false
terraform test
```

## Machine-readable output

`terraform show -json` turns a saved plan into JSON that other tools can
check. A trimmed resource change looks like this:

```json
{
  "address": "azurerm_storage_account.this",
  "change": {
    "actions": ["create"],
    "after": {
      "name": "stlarskrogh01",
      "account_replication_type": "ZRS"
    }
  }
}
```

A few lines of Go are enough to fail a pipeline when a plan deletes
anything:

```go
package main

import (
	"encoding/json"
	"fmt"
	"os"
	"slices"
)

type plan struct {
	ResourceChanges []struct {
		Address string `json:"address"`
		Change  struct {
			Actions []string `json:"actions"`
		} `json:"change"`
	} `json:"resource_changes"`
}

func main() {
	var p plan
	if err := json.NewDecoder(os.Stdin).Decode(&p); err != nil {
		fmt.Fprintln(os.Stderr, "read plan:", err)
		os.Exit(2)
	}
	failed := false
	for _, rc := range p.ResourceChanges {
		if slices.Contains(rc.Change.Actions, "delete") {
			fmt.Println("delete:", rc.Address)
			failed = true
		}
	}
	if failed {
		os.Exit(1)
	}
}
```

```bash
terraform plan -out=tfplan
terraform show -json tfplan | go run ./cmd/nodelete
```

## Related

This site's own pipeline is described in
[Publishing This Blog with Hugo and GitHub Pages]({{< relref "posts/publishing-with-hugo-and-github-pages" >}}).
