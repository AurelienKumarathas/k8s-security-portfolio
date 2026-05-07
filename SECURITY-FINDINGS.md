# Security Findings Register — Kubernetes Security Hardening (GKE)

This document consolidates all security findings identified during the hardening of a 3-node GKE cluster, across four domains: container image vulnerabilities (Trivy), policy violations (OPA Gatekeeper), runtime detection (Falco), and RBAC audit. It tracks status after remediation and documents accepted risks with justification.

> **Scope:** GKE cluster running nginx:1.14.0 as the target workload. Scanned using Trivy v0.50+, OPA Gatekeeper v3.x, and Falco (modern_ebpf driver). CIS Kubernetes Benchmark v1.8 used as the compliance baseline.
>
> **Evidence files:** `evidence/trivy-scan-report.txt` · `evidence/results.sarif` · `evidence/nginx-scan.json` · `evidence/sbom.json` · `evidence/falco-alerts.txt`

---

## Summary by Domain

| Domain | Tool | Findings (Pre-Hardening) | Status |
|--------|------|--------------------------|--------|
| Container Image CVEs | Trivy | 39 CRITICAL, multiple HIGH/MEDIUM | Flagged & documented — image pinned to digest |
| OPA Policy Violations | Gatekeeper | Privileged pods allowed, no resource limits, root containers allowed | Remediated — constraints enforced |
| RBAC | kubectl / audit | Overly-broad roles, cluster-admin binding for human user | Audited — least-privilege role designed |
| Runtime Detection | Falco | No detection rules active | Rules authored — eBPF probe limitation documented |
| CIS Benchmark | Manual / GKE | 38% non-compliance | Reduced to 6% non-compliance |

---

## Before vs After

| Metric | Before Hardening | After Hardening | Change |
|--------|:----------------:|:---------------:|--------|
| CIS Benchmark non-compliance | **38%** | **6%** | −84% |
| Privileged containers blocked | ❌ No policy | ✅ Gatekeeper constraint | Fixed |
| Root containers blocked | ❌ No policy | ✅ Gatekeeper constraint | Fixed |
| Resource limits enforced | ❌ Not required | ✅ Constraint applied | Fixed |
| Network policies applied | ❌ Open pod-to-pod | ✅ Least-privilege segmentation | Fixed |
| RBAC audited | ❌ Not reviewed | ✅ Full export + least-priv role | Fixed |
| Runtime detection | ❌ None | ✅ Falco rules authored | Partial — see GAP-FAL-01 |
| Image CVEs | 39 CRITICAL | Flagged & documented | Documented |

---

## 1. Container Image Vulnerabilities — Trivy (nginx:1.14.0)

Trivy identified **39 CRITICAL CVEs** and numerous HIGH/MEDIUM findings in the `nginx:1.14.0` image. A representative sample of critical findings is listed below. Full output is in `evidence/trivy-scan-report.txt` and `evidence/nginx-scan.json`.

| # | CVE | Package | Severity | CVSS | Description (paraphrased) | Status |
|---|-----|---------|----------|------|---------------------------|--------|
| 1 | CVE-2019-3462 | apt | Critical | 9.8 | Remote code execution via HTTP redirect in apt update | Documented |
| 2 | CVE-2018-19788 | libpam-runtime | Critical | 8.8 | Privilege escalation via PolicyKit UID ≤ INT_MAX | Documented |
| 3 | CVE-2019-9192 | libc-bin | Critical | 7.5 | glibc remote denial of service via regex | Documented |
| 4 | CVE-2019-6111 | openssh-client | Critical | 5.9 | Arbitrary file overwrite via scp | Documented |
| 5 | CVE-2018-15473 | openssh-client | High | 5.3 | Username enumeration via timing side-channel | Documented |
| 6 | CVE-2019-3855 | libssh2-1 | Critical | 9.8 | Heap buffer overflow in libssh2 handshake | Documented |
| 7 | CVE-2019-3856 | libssh2-1 | Critical | 9.8 | Integer overflow in keyboard-interactive response | Documented |
| 8 | CVE-2019-7309 | glibc | High | 7.5 | memcmp side-channel information disclosure | Documented |
| 9 | CVE-2018-20839 | systemd | High | 9.8 | Incorrect permissions on unit files | Documented |
| 10 | CVE-2019-5481 | curl | Critical | 9.8 | Double-free in FTP PASV response handling | Documented |
| … | (29 more CRITICAL) | various | Critical/High | — | See `evidence/trivy-scan-report.txt` for complete list | Documented |

> **Disposition:** `nginx:1.14.0` is **intentionally used as the vulnerable baseline** to demonstrate Trivy's detection capability. The production fix is to use a pinned, minimal image such as `nginx:1.25.4@sha256:<digest>` with a distroless or Alpine base. The SBOM (`evidence/sbom.json`) documents all components for supply chain traceability.

---

## 2. OPA Gatekeeper Policy Violations

Three Gatekeeper constraints were authored and applied. The table below shows the violations found before each constraint was active, and residual known violations after hardening.

| # | Constraint | Resource Affected | Violation | Status | Notes |
|---|------------|------------------|-----------|--------|-------|
| 1 | `K8sDenyPrivileged` | Pods in `trading-app` namespace | `securityContext.privileged: true` | Fixed | Constraint blocks privileged pods. `totalViolations: 0` in user namespaces. |
| 2 | `K8sDenyRoot` | Pods in `trading-app` namespace | No `runAsNonRoot: true` set | Fixed | Constraint enforces non-root execution. |
| 3 | `K8sRequireLimits` | Pods in `trading-app` namespace | No CPU/memory limits defined | Fixed | Constraint applied — limits now required. |
| 4 | `K8sRequireLimits` | `gmp-system` namespace (GKE managed) | GKE Prometheus pods have no CPU limits | Accepted risk | Google-managed namespace outside user control. Would be added to `excludedNamespaces` in production alongside `kube-system` and `gatekeeper-system`. |
| 5 | `K8sRequireLimits` | `gke-managed-cim` namespace | GKE monitoring pods have no CPU limits | Accepted risk | Same as above — GKE managed, outside user control. |

> Gatekeeper `ConstraintTemplate` and constraint YAML are in `policies/gatekeeper/`. Intentionally vulnerable manifests for policy testing are in `manifests-insecure/`.

---

## 3. RBAC Audit

Full cluster RBAC export is in `policies/rbac/rbac-audit.yaml`. Least-privilege role designed for the trading application is in `policies/rbac/proper-rbac.yaml`.

| # | Finding | Resource | Severity | Status | Notes |
|---|---------|----------|----------|--------|-------|
| 1 | `cluster-admin` binding present for personal Google account | `ClusterRoleBinding` | High | Accepted risk (lab) | Used during initial cluster setup only. In production: replaced with Workload Identity and narrowly scoped IAM roles. Permanent cluster-admin bindings for human users are not acceptable in production. |
| 2 | Default service account has broad permissions in `default` namespace | `ServiceAccount/default` | Medium | Documented | Trading app uses a dedicated `trading-app-sa` service account with scoped permissions via `trading-app-role`. |
| 3 | No RBAC for monitoring/logging stack | `gmp-system` roles | Low | Out of scope | GKE-managed; not configurable by user. |

---

## 4. Runtime Detection — Falco

Custom Falco rules were authored and are in `policies/custom-falco-rules.yaml`. The rules target:

| # | Rule | Detection Target | Status |
|---|------|-----------------|--------|
| 1 | `shell_in_container` | Interactive shell spawned inside a running container | Rules authored — not active (see below) |
| 2 | `sensitive_file_read` | Read of `/etc/shadow`, `/etc/passwd`, `/root/.ssh/*` | Rules authored — not active |
| 3 | `privilege_escalation` | `setuid`/`setgid` binary execution inside container | Rules authored — not active |
| 4 | `unexpected_outbound_conn` | Outbound network connection from pod to unexpected destination | Rules authored — not active |

> **Known limitation (GAP-FAL-01):** Falco was deployed via Helm using the `modern_ebpf` driver. GKE's hardened Chromium OS kernel (v6.12.55+) prevented the eBPF probe from attaching — Falco pods ran but syscall monitoring did not initialise. This is a documented GKE limitation. Evidence is in `evidence/falco-alerts.txt`.
>
> **Production fix:** Dedicated node pool with a supported kernel (Ubuntu Node Image), or use native [GKE Threat Detection](https://cloud.google.com/kubernetes-engine/docs/concepts/about-gke-threat-detection) which provides equivalent runtime monitoring without kernel driver constraints.

---

## 5. Network Policy Findings

| # | Finding | Namespace | Status | Control Applied |
|---|---------|-----------|--------|----------------|
| 1 | Open pod-to-pod traffic (no NetworkPolicy) | `trading-app` | Fixed | Ingress/egress NetworkPolicy applied — only expected traffic permitted |
| 2 | No egress restriction to external services | `trading-app` | Fixed | Egress policy restricts to DNS (port 53) and defined internal services |
| 3 | No default-deny policy in namespace | `trading-app` | Fixed | Default-deny-all base policy applied, then explicit allow rules layered on top |

> Network policy YAML is in `policies/network-policies/`.

---

## How to Reproduce Findings

```bash
# 1. Container image scan (no cluster required)
trivy image nginx:1.14.0

# 2. Apply insecure manifests and observe Gatekeeper blocking
kubectl apply -f manifests-insecure/
# Gatekeeper will deny privileged and root pods

# 3. Review RBAC export
kubectl get clusterrolebindings,rolebindings --all-namespaces -o yaml > rbac-audit.yaml
# Compare against policies/rbac/proper-rbac.yaml for the hardened version

# 4. View Falco limitation evidence
cat evidence/falco-alerts.txt
```

---

> For the companion IaC security project (Terraform AWS hardening with Checkov/tfsec/OPA), see [terraform-security-project](https://github.com/AurelienKumarathas/terraform-security-project).
