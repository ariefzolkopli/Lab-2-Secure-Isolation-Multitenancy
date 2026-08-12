# IKB42603 - Lab 2: Secure Isolation & Multi-Tenancy

## 📋 Lab Overview
This lab demonstrates secure isolation in multi-tenant Kubernetes environments across three dimensions:
- **Compute Isolation** - Namespaces and Resource Quotas
- **Network Isolation** - Calico NetworkPolicies (Default-Deny)
- **Storage Isolation** - RBAC and Secret Management

## 🎯 Learning Outcomes
- ✅ Demonstrate compute isolation using Kubernetes namespaces
- ✅ Identify the default-open network risk in shared clusters
- ✅ Implement network isolation with default-deny NetworkPolicies
- ✅ Enforce storage isolation with RBAC and Secrets
- ✅ Understand data remanence and secure deletion

## 🛠️ Technologies Used
| Tool | Purpose |
|------|---------|
| **Kubernetes (Kind)** | Container orchestration platform |
| **Calico** | CNI for NetworkPolicy enforcement |
| **kubectl** | Kubernetes command-line tool |
| **Docker** | Container runtime |


## 📊 Key Results

### Before vs After Network Policy
| Metric | Before | After |
|--------|--------|-------|
| Cross-tenant Access | ✅ HTTP 200 (Allowed) | ❌ HTTP 000 (Blocked) |
| Security Posture | ❌ Default-Open | ✅ Default-Deny |

### RBAC Secret Isolation
| Attempt | Result |
|---------|--------|
| tenant-a reads own secrets | ✅ **yes** (Permitted) |
| tenant-a reads tenant-b's secrets | ❌ **no** (Denied) |

## 📝 Lab Tasks
| Task | Description | Isolation Type |
|------|-------------|----------------|
| Task 1 | Two tenants on one cluster (namespaces) | Compute |
| Task 2 | Default-open risk demonstration | Network |
| Task 3 | Resource quotas (noisy neighbor prevention) | Compute |
| Task 4 | Default-deny NetworkPolicy enforcement | Network |
| Task 5 | Storage & Secret isolation via RBAC | Storage |
| Task 6 | Data remanence & secure deletion | Storage |

## 🔒 Security Best Practices Demonstrated
- ✅ Tenants separated into distinct namespaces
- ✅ Default-deny NetworkPolicy blocks cross-tenant traffic
- ✅ Resource quotas prevent noisy-neighbor attacks
- ✅ RBAC prevents cross-tenant secret access
- ✅ Secure deletion for data remanence protection

## 👤 Student Information
| Detail | Information |
|--------|-------------|
| **Student Name** | Muhammad Arief Bin Zolkopli |
| **Student ID** | 52215125138 |
| **Course** | IKB42603 - Cloud Computing Security Essentials |
| **Lab** | Lab 2 - Secure Isolation & Multi-Tenancy |
| **Date** | 12 August 2026 |

## 📖 References
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Calico Documentation](https://docs.tigera.io)
- [CSA Security Guidance v5](https://cloudsecurityalliance.org/research/security-guidance/)

## 🚀 How to Run This Lab
```bash
# Create kind cluster with Calico
cat <<EOF | kind create cluster --name ccse-lab2 --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
EOF

# Install Calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# Verify cluster
kubectl get nodes
