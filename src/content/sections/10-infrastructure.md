---
title: "Infrastructure as Code"
order: 10
description: "Everything in the repo — manifests, configs, deploy scripts. Secrets in the vault."
---

## Infrastructure as Code

### 36. Everything in the Repo

All manifests, Dockerfiles, nginx configs, and deploy scripts live in the repo. Secrets live in a vault (1Password, etc.) and are injected at deploy time — never committed, never hardcoded, never in environment files checked into source. Secret references are in code (`op read 'op://Vault/item/field'`), secret values are not. Every environment (dev, staging, production) is reproducible from the repo alone plus vault access.
