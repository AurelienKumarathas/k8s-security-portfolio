# Gatekeeper Library — Reference Templates

This directory contains **reference/template versions** of the OPA Gatekeeper constraint templates.
They are **not** the deployed constraints — they are kept here as a clean, standalone template store.

## What Was Actually Deployed

The constraints applied to the live GKE cluster are in `policies/gatekeeper/`:

| File | Kind | Purpose |
|---|---|---|
| `deny-privileged.yaml` | `K8sDenyPrivileged` | Blocks privileged containers at admission |
| `require-limits.yaml` | `K8sRequireLimits` | Enforces CPU/memory resource limits |
| `gatekeeper-constraints.yaml` | `K8sDenyPrivileged` + `K8sRequireLimits` | Live constraint objects with enforcement status |

The `gatekeeper-constraints.yaml` file contains the live constraint status including real UIDs,
timestamps, and violation counts — proving these were applied to a real cluster.

## Why Two Sets?

The `gatekeeper-library/` templates use a slightly different naming convention (`K8sBlockPrivileged`,
`K8sRequireResourceLimits`) following the upstream OPA Gatekeeper library pattern. The
`policies/gatekeeper/` versions were the ones actually deployed — they use shorter kind names
(`K8sDenyPrivileged`, `K8sRequireLimits`) scoped to this project.

In a team environment, the library folder would be the canonical source of truth with CI promotion
to the deployed policies directory.
