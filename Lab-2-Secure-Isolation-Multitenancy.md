# IKB42603 - Lab 2: Secure Isolation & Multi-Tenancy

| Item | Details |
| --- | --- |
| Course | IKB42603 - Cloud Computing Security Essentials |
| Lab | Lab 2 - Secure Isolation & Multi-Tenancy |
| Student name | Muhammad Arief Bin Zolkopli |
| Student ID | 52215125138 |
| Date completed | 12 August 2026 |

## Objective

This lab demonstrates isolation across the three dimensions of shared cloud infrastructure: **compute**, **network**, and **storage**, using Docker and Kubernetes. It models two tenants sharing one Kubernetes cluster, demonstrates the default-open risk, and applies the controls needed for secure multi-tenancy.

The outcomes demonstrated are:

1. Compute isolation using separate Kubernetes namespaces and resource quotas.
2. The default-open network behaviour of a shared Kubernetes cluster.
3. Network isolation using a Calico-enforced default-deny NetworkPolicy.
4. Storage isolation using namespace-scoped Secrets and RBAC.
5. Data remanence and secure deletion concepts.

## Environment Setup

The required command-line tools were verified before the lab environment was created.

![Required tools verified](Evidence/Setup%20-%20Verify%20all%20tools%20work.png)

The `kind` cluster was created with its default CNI disabled, then Calico was installed so NetworkPolicies could be enforced.

```bash
cat <<EOF | kind create cluster --name ccse-lab2 --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
EOF

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

![Cluster creation and Calico installation](Evidence/Setup%20-%20%231%20Create%20a%20cluster%20with%20the%20default%20CNI%20disabled%2C%20then%20install%20Calico.png)

![Calico rollout completion](Evidence/Setup%20-%20%232%20Create%20a%20cluster%20with%20the%20default%20CNI%20disabled%2C%20then%20install%20Calico.png)

The cluster was then verified as ready.

![Cluster readiness verified](Evidence/Setup%20-%20Verify%20cluster%20is%20ready.png)

## Task 1 - Two Tenants on One Cluster

Two namespaces, `tenant-a` and `tenant-b`, were created to represent separate customers on the same Kubernetes cluster. An NGINX `web` deployment and Service were created in each namespace.

```bash
kubectl create namespace tenant-a
kubectl create namespace tenant-b

kubectl -n tenant-a create deployment web --image=nginx
kubectl -n tenant-b create deployment web --image=nginx

kubectl -n tenant-a expose deployment web --port=80
kubectl -n tenant-b expose deployment web --port=80
kubectl get pods,svc -n tenant-a
```

Namespaces provide administrative and compute separation, but they do not automatically block network communication between workloads.

![Two tenant namespaces and web Services](Evidence/Task%201%20%E2%80%94%20Two%20Tenants%20on%20One%20Cluster.png)

## Task 2 - Observe the Default-Open Risk

The ClusterIP address of the `tenant-b` web Service was retrieved, then a probe was run from `tenant-a`.

```bash
B_IP=$(kubectl get svc web -n tenant-b -o jsonpath='{.spec.clusterIP}')
echo "Tenant-B IP is: $B_IP"

kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never \
  -- curl -s -m 5 http://$B_IP -o /dev/null -w 'HTTP %{http_code}\n'
```

The evidence shows an initial malformed request caused by an unset variable. Once `B_IP` was set correctly, the probe returned **HTTP 200**. This proves that `tenant-a` could reach `tenant-b` across namespace boundaries by default, which is unsafe in a multi-tenant environment.

![Cross-tenant probe returns HTTP 200](Evidence/Task%202%20%E2%80%94%20Observe%20the%20Default-Open%20Risk.png)

## Task 3 - Contain the Noisy Neighbour (Resource Quotas)

A ResourceQuota was applied to `tenant-a` to prevent one tenant from using excessive shared cluster resources.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 512Mi
    pods: "5"
```

This quota limits requested CPU, requested memory, and pod count. It reduces the noisy-neighbour risk by ensuring that one tenant cannot consume disproportionate capacity.

![ResourceQuota configured for tenant-a](Evidence/Task%203%20%E2%80%94%20Contain%20the%20Noisy%20Neighbour%20%28Resource%20Quotas%29.png)

## Task 4 - Default-Deny Network Isolation

A default-deny ingress NetworkPolicy was applied to `tenant-b`.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: tenant-b
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

![Default-deny policy applied](Evidence/Task%204%20%E2%80%94%20%231%20Default-Deny%20Network%20Isolation.png)

The cross-tenant connection test was repeated from the `tenant-a` web pod.

```bash
WEB_POD=$(kubectl get pod -n tenant-a -l app=web -o jsonpath='{.items[0].metadata.name}')
kubectl -n tenant-a exec -it $WEB_POD -- \
  curl -s -m 5 http://$B_IP -o /dev/null -w 'HTTP %{http_code}\n'
```

The request returned **HTTP 000** and timed out (exit code 28). This is direct evidence that Calico enforced the NetworkPolicy and blocked cross-tenant ingress to `tenant-b`.

![Cross-tenant request blocked with HTTP 000](Evidence/Task%204%20%E2%80%94%20%232%20Default-Deny%20Network%20Isolation.png)

| Test | Result |
| --- | --- |
| Before `default-deny-ingress` | HTTP 200 - reachable |
| After `default-deny-ingress` | HTTP 000 - blocked / timed out |

## Task 5 - Storage & Secret Isolation

Separate Secrets were created for both tenants. A service account in `tenant-a` was then given permission to read Secrets only in its own namespace.

```bash
kubectl -n tenant-a create secret generic data --from-literal=value=SECRET_A
kubectl -n tenant-b create secret generic data --from-literal=value=SECRET_B

kubectl -n tenant-a create serviceaccount app-a
kubectl -n tenant-a create role reader --verb=get --resource=secrets
kubectl -n tenant-a create rolebinding rb --role=reader --serviceaccount=tenant-a:app-a

SA=system:serviceaccount:tenant-a:app-a
kubectl auth can-i get secrets -n tenant-a --as=$SA
kubectl auth can-i get secrets -n tenant-b --as=$SA
```

The service account can access Secrets in `tenant-a`, but it cannot access Secrets in `tenant-b`. This confirms RBAC-based storage isolation between tenants.

![Secrets and RBAC isolation verification](Evidence/Task%205%20%E2%80%94%20Storage%20%26%20Secret%20Isolation.png)

## Task 6 - Data Remanence & Secure Deletion

First, sensitive data was written to a Docker volume and removed using a normal delete operation.

```bash
docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; rm /data/phi.txt; \
  grep -a SENSITIVE /data/* 2>/dev/null; echo scan-done'
```

A normal `rm` removes a filesystem reference but does not guarantee that the underlying storage blocks are overwritten. This is known as **data remanence**.

![Normal deletion and remanence check](Evidence/Task%206%20%E2%80%94%20%231%20Data%20Remanence%20%26%20Secure%20Deletion.png)

Next, a second file was overwritten with zeros before it was deleted.

```bash
docker run --rm -v ccse-vol:/data alpine sh -c \
  'echo SENSITIVE > /data/phi2.txt; sync; \
  dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc; \
  rm /data/phi2.txt; echo wiped'
```

![Secure overwrite before deletion](Evidence/Task%206%20%E2%80%94%20%232%20Data%20Remanence%20%26%20Secure%20Deletion.png)

In cloud environments, customers generally do not control physical storage blocks. Encrypting data at rest and destroying the encryption key, known as **cryptographic erasure**, is therefore the practical way to make residual data unrecoverable.

## Verification Commands

```bash
kubectl get networkpolicy -A
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

![NetworkPolicy and ResourceQuota verification](Evidence/Verification%20Command.png)

## Cleanup & Teardown

The lab resources were removed after verification to keep the environment clean.

![Cleanup and teardown](Evidence/Cleanup%20%26%20Teardown.png)

## Short-Answer Questions

**Q1. Why can containers in different namespaces reach each other by default, and why is that dangerous in multi-tenant cloud?**

Namespaces are logical administrative boundaries, not network firewalls. Without NetworkPolicies, pods can communicate across namespaces over the shared cluster network. A compromised workload could therefore access or attack another tenant's services.

**Q2. Explain the default-deny principle and how your NetworkPolicy implements it.**

Default-deny blocks traffic unless an explicit rule allows it. The policy selects every pod in `tenant-b`, applies to ingress, and has no allow rules. Consequently, all inbound traffic to those pods is denied.

**Q3. How do virtual machines and containers differ in isolation strength? When would you add a VM boundary?**

VMs provide stronger isolation because each VM has its own guest operating system and kernel. Containers share the host kernel, making them lightweight but providing a weaker boundary. A VM boundary is appropriate for highly sensitive, untrusted, or high-risk workloads.

**Q4. What is data remanence, and why is cryptographic erasure the preferred cloud solution?**

Data remanence is residual data that may remain on storage after deletion. Because cloud users cannot normally overwrite or inspect provider storage media, destroying the encryption key is a practical way to make the remaining encrypted data unusable.

**Q5. Which isolation dimension did each task exercise?**

| Task | Isolation dimension |
| --- | --- |
| Task 1 - Separate namespaces | Compute |
| Task 2 - Default-open probe | Network |
| Task 3 - ResourceQuota | Compute |
| Task 4 - Default-deny NetworkPolicy | Network |
| Task 5 - Secrets and RBAC | Storage |
| Task 6 - Remanence and secure deletion | Storage |

## Security Best-Practices Checklist

- [x] Separate namespaces were used for `tenant-a` and `tenant-b`.
- [x] A ResourceQuota limits the resources requested by `tenant-a`.
- [x] A default-deny NetworkPolicy blocks ingress to `tenant-b`.
- [x] Network isolation was proven by the HTTP 200 to HTTP 000 before/after result.
- [x] RBAC restricts Secret access to the appropriate tenant namespace.
- [x] Secure deletion and cryptographic erasure concepts were examined.

## Conclusion

Lab 2 demonstrated that secure multi-tenancy requires explicit controls across compute, network, and storage layers. Compute isolation was achieved using namespaces and a ResourceQuota. Network isolation was proven with a before-and-after test: the `tenant-a` workload reached `tenant-b` with HTTP 200 before protection, but received HTTP 000 after Calico enforced a default-deny NetworkPolicy. Storage isolation was demonstrated through namespace-scoped Secrets and RBAC, while the data-remanence task showed why cryptographic erasure is the practical deletion strategy for cloud storage.
