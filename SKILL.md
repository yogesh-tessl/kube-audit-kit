---
name: kube-audit-kit
description: Performs read-only Kubernetes security audits by exporting resources, sanitizing metadata, grouping applications by topology, and generating PSS/NSA-compliant audit reports. Use when the user requests auditing Kubernetes clusters, Namespaces, security reviews, or configuration analysis.
user-invocable: true
allowed-tools: Read, Write, Bash(python:*), Bash(uv:*), Bash(kubectl:*), Bash(export:*)
examples:
  - "Run a security audit for the payment namespace in prod-cluster"
  - "Check whether the backend apps in staging meet PSS standards"
  - "Analyze sensitive data leakage risk for all resources in the development namespace"
  - "Generate a full audit report for the default namespace in test-cluster"
  - "review k8s cluster security configuration"
  - "kubernetes security audit for production workload"
author: crazygit
repository: https://github.com/crazygit/kube-audit-kit
---

# Kube Audit Kit - Read-Only Kubernetes Security Audit Toolkit

Exports Kubernetes cluster resources in **read-only** mode (only `get/list` operations), sanitizes metadata, groups applications by topology, and performs a dual-layer security audit against PSS and NSA standards.

## Quick Start

### Prerequisites

1. **Environment setup**:

   ```bash
   uv sync
   ```

   See [SETUP.md](SETUP.md) for details.

2. **Verify kubectl**:
   ```bash
   kubectl config get-contexts
   ```

### Run an audit

When a user requests an audit, follow these steps strictly.

> **Output directory**: Every command below must be prefixed with `export KUBE_AUDIT_OUTPUT="$(pwd)/output" &&` so output lands in the user's working directory, not the SKILL install directory. See [WORKFLOW.md](WORKFLOW.md) for details on path resolution.

**Progress checklist**:

```
- [ ] Step 1: Export - Dynamic discovery and full resource export
- [ ] Step 2: Sanitize - Remove metadata and status fields
- [ ] Step 3: Group - Associate applications by workload topology
- [ ] Step 4: Audit - Dual-layer security audit
```

Each step follows the pattern: `export KUBE_AUDIT_OUTPUT="$(pwd)/output" && uv run python scripts/<script>.py --context <context> --namespace <namespace>`

| Step | Script | Output |
|------|--------|--------|
| 1. Export | `scripts/export.py` | `{OUTPUT_BASE}/export/` |
| 2. Sanitize | `scripts/sanitize.py` | `{OUTPUT_BASE}/sanitize/`, `{OUTPUT_BASE}/sanitize_fields/` |
| 3. Group | `scripts/group_apps.py` | `{OUTPUT_BASE}/group/`, `{OUTPUT_BASE}/ungrouped_resources.txt` |
| 4. Audit | `scripts/audit.py` | `{OUTPUT_BASE}/audit/` (see Output Structure below) |

#### Step 4: Audit — Phase 2 (AI Expert Deep Review)

AI independently reviews results without relying on phase 1 output:

1. **Independent analysis**: traverse `{OUTPUT_BASE}/group/*/` and read all original YAML files
2. **Deep review**: identify risks not covered by script rules
   - business logic risks (e.g., plaintext private keys, hardcoded passwords)
   - architecture risks (e.g., missing NetworkPolicy, overly broad RBAC)
   - configuration drift risks (e.g., `latest` images, missing resource limits)
3. **Supplement findings**: if sensitive data was missed, append to the CSV files
4. **Report summary**: merge phase 1 findings with AI analysis into `{OUTPUT_BASE}/audit/audit_report.md`

**Report template**: see `audit_report_template.md` in the same directory.

**Key requirements**:

- Must read original YAML files, not just audit_results.json
- Every application must have specific analysis; avoid vague statements like "not reviewed"
- If script misses sensitive data, update the CSV files to keep data complete

## Output Structure

All output writes to `output/{context}/{namespace}/` with subdirectories for each step: `export/`, `sanitize/`, `sanitize_fields/`, `group/{app_name}/`, and `audit/`. The final report is at `audit/audit_report.md`. See [EXAMPLES.md](EXAMPLES.md) for full directory layout and sample outputs.

## Reference Docs

- **[QUICKSTART.md](QUICKSTART.md)**: 30-second quick start
- **[WORKFLOW.md](WORKFLOW.md)**: full workflow and implementation details
- **[SETUP.md](SETUP.md)**: environment setup and dependency installation
- **[EXAMPLES.md](EXAMPLES.md)**: output examples and typical scenarios

## Key Behaviors

- **Volume mounts vs EnvVar references**: ConfigMaps/Secrets used as volume mounts skip sensitive scanning; those referenced as env vars are scanned. `config_usage.json` records the usage type.
- **Permission errors**: if a resource type is denied, it is skipped with a warning. Other types continue. The final report notes missing checks. Use a dedicated audit service account (see [SETUP.md](SETUP.md)).

## Security Reminder

**Warning**: the `output/` directory contains decrypted Secret data.

**After the audit**:

- Keep `audit_report.md` (it does not contain sensitive data)
- Securely delete other directories or store them encrypted
- Do not commit `output/` to version control
