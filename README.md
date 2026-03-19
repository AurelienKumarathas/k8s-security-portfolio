# Kubernetes Security Hardening — GCP/GKE Portfolio Project
...
# 🔐 Kubernetes Security Hardening on GKE

![GCP](https://img.shields.io/badge/Google_Cloud-GKE-4285F4?style=flat&logo=googlecloud&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-1.29-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Trivy](https://img.shields.io/badge/Trivy-Vulnerability_Scanning-1904DA?style=flat&logo=aqua&logoColor=white)
![OPA](https://img.shields.io/badge/OPA_Gatekeeper-Policy_Enforcement-7B42BC?style=flat)
![Falco](https://img.shields.io/badge/Falco-Runtime_Security-00AEC7?style=flat)
![License](https://img.shields.io/badge/License-MIT-green?style=flat)

> End-to-end Kubernetes security hardening project on GCP — covering vulnerability scanning, policy enforcement, runtime threat detection, network segmentation and RBAC auditing.

---

## 📸 Screenshots

| GKE Cluster Running | Trivy Vulnerability Scan |
|---|---|
| ![GKE](screenshots/gke-cluster-running.png) | ![Trivy](screenshots/trivy-vulnerability-scan.png) |

| Gatekeeper Blocking Privileged Pod | Network Policies Deployed |
|---|---|
| ![Gatekeeper](screenshots/gatekeeper-blocking-privileged-pod.png) | ![Network](screenshots/network-policies-deployed.png) |

---

## ✅ Features

- 🔍 Container image scanning with **Trivy** — identified 39 CRITICAL CVEs on nginx:1.14.0
- 🛡️ Policy-as-code enforcement with **OPA Gatekeeper** — blocks privileged pods and root containers
- 🌐 **Network Policies** — least-privilege pod-to-pod traffic control
- 👤 **RBAC Audit** — full export and review of all roles and bindings across namespaces
- 🚨 **Falco** custom detection rules — shell spawning, sensitive file access, privilege escalation
- 📦 SBOM generation and SARIF reporting for supply chain visibility

---

## ⚡ Quick Start

```bash
git clone https://github.com/AurelienKumarathas/k8s-security-lab.git
cd k8s-security-lab
trivy image nginx:1.14.0
kubectl apply -f policies/gatekeeper/
kubectl apply -f policies/network-policies/
kubectl apply -f policies/rbac/
```

---

## 🛠️ Prerequisites

- GCP account with GKE enabled
- `gcloud` CLI authenticated
- `kubectl` configured against your cluster
- `trivy` installed locally (`brew install trivy`)
- Helm 3+

---

## 📁 Repo Structure

```
k8s-security-lab/
├── evidence/              # Trivy reports, SARIF, SBOM, Falco alerts
├── manifests-insecure/    # Intentionally vulnerable manifests for policy testing
├── policies/              # Gatekeeper, network policies, RBAC, Falco rules
├── screenshots/           # GCP Console and CLI evidence
└── gatekeeper-library/    # OPA Gatekeeper policy library
```

---

## 🏗️ Architecture

```mermaid
graph TD
    A[Developer] -->|kubectl apply| B[GKE Cluster]
    B --> C[OPA Gatekeeper]
    C -->|Blocks| D[Privileged Pods]
    C -->|Allows| E[Compliant Pods]
    B --> F[Falco]
    F -->|Monitors| G[Runtime Syscalls]
    F -->|Alerts| H[Security Events]
    B --> I[Network Policies]
    I -->|Restricts| J[Pod-to-Pod Traffic]
    K[Trivy] -->|Scans| L[Container Images]
    L -->|Reports| M[CVE Findings]
```

---

## 💼 Skills Demonstrated

| Skill | Tool | Relevance |
|---|---|---|
| Vulnerability Management | Trivy | Shift-left security, CVE triage |
| Policy-as-Code | OPA Gatekeeper | Automated compliance enforcement |
| Runtime Threat Detection | Falco | SIEM/SOC alerting, incident response |
| Network Segmentation | Kubernetes Network Policies | Zero-trust architecture |
| RBAC Hardening | Kubernetes RBAC | Least-privilege access control |
| Cloud Security | GKE / GCP | Cloud-native security posture |
| Supply Chain Security | SBOM, SARIF | DevSecOps pipeline integration |

---

## ⚠️ Notes on Falco

Falco was deployed via Helm using the `modern_ebpf` driver. GKE's hardened Chromium OS kernel (v6.12.55+) prevented the eBPF probe from attaching. Falco pods ran but syscall monitoring did not initialise — this is a known GKE limitation. Custom rules were authored and are in `policies/`. Production fix: dedicated node pool with a supported kernel, or native GKE Threat Detection.

---

## 📄 License

MIT — see [LICENSE](LICENSE) for details.


