# Install Guide — RKE2 + MetalLB + Argo CD

This guide installs a single-node **RKE2** Kubernetes control plane and deploys **MetalLB** and **Argo CD** using **vendored Helm charts**
from this repo. It also documents the **app‑of‑apps** pattern so Argo CD can manage MetalLB and **itself**.


---

## Repo structure

```

── gitops/
    ├── vendor-versions.env                 # Helmsharts
    ├── scripts/
    │   └── vendor_charts.sh                # fetches charts into vendor/
    │
    ├── clusters/
    │   └── prod.yaml                       # ROOT app-of-apps (directory.recurse: true)
    │
    ├── argocd/
    │   ├── values/
    │   │   ├── base.yaml                   # baseline values
    │   │   └── custom-values.yaml          
    │   ├── vendor/
    │   │   ├── argo-cd-9.0.5/              
    │   │   └── argo-cd -> argo-cd-9.0.5    # symlink to current version
    │   └── app.yaml                        # Argo CD self-managing Application
    │
    ├── metallb/
    │   ├── values/
    │   │   └── base.yaml                   
    │   ├── manifests/
    │   │   ├── ipaddresspool.yaml          
    │   │   └── l2advertisement.yaml
    │   ├── vendor/
    │   │   ├── metallb-0.15.2/             
    │   │   └── metallb -> metallb-0.15.2   
    │   ├── app-metallb.yaml                # Application (wave 0) installs chart
    │   └── app-metallb-config.yaml         # Application (wave 1) applies pool/L2
    │
    └── INSTALL.md / README.md              # this doc
```


---

## 1) Install RKE2 (Server)

Run on the target node:

```bash
# Become root only for the installer
sudo su
curl -sfL https://get.rke2.io | sh -
systemctl enable rke2-server.service
systemctl start rke2-server.service
exit
```

Wait a minute for the cluster to come up, then configure `kubectl` for your user:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown "$(id -u)":"$(id -g)" ~/.kube/config

# Add RKE2 kubectl to PATH for this shell and persist it in bashrc
export PATH=$PATH:/var/lib/rancher/rke2/bin
echo 'export PATH=$PATH:/var/lib/rancher/rke2/bin' >> ~/.bashrc

# (Optional) shell conveniences
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' >> ~/.bashrc
alias k=kubectl
complete -F __start_kubectl k
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -F __start_kubectl k' >> ~/.bashrc
```

Verify the control plane is healthy:

```bash
kubectl get nodes -o wide
kubectl -n kube-system get pods
```

You should see the node `Ready` and core pods starting/Running.

---

## 2) Vendor exact chart versions

```bash
# Edit versions here to bump later
sed -n '1,200p' vendor-versions.env
```
```bash
# Vendor the exact chart versions into ./gitops/*/vendor/
bash gitops/scripts/vendor_charts.sh
```

This will create (and symlink):
- `gitops/argocd/vendor/argo-cd-9.0.5/`  and `gitops/argocd/vendor/argo-cd -> argo-cd-9.0.5`
- `gitops/metallb/vendor/metallb-0.15.2/` and `gitops/metallb/vendor/metallb -> metallb-0.15.2`

---

## 3) Install MetalLB (chart + your address pools)

> **Edit the IP address pool** in `gitops/metallb/manifests/ipaddresspool.yaml` if needed (same L2 as the node, not overlapping DHCP).

Install the chart from the vendored path:

```bash
# From repo root
helm upgrade --install metallb \
  ./gitops/metallb/vendor/metallb \
  --namespace metallb-system --create-namespace \
  -f ./gitops/metallb/values/base.yaml
# (Optional) add your own overrides: -f ./gitops/metallb/values/custom-values.yaml
```

Apply your IP pool and L2Advertisement CRs:

```bash
kubectl -n metallb-system apply -f ./gitops/metallb/manifests/ipaddresspool.yaml
kubectl -n metallb-system apply -f ./gitops/metallb/manifests/l2advertisement.yaml
```

Verify pods:

```bash
kubectl -n metallb-system get pods
```

You should get an external IP from your MetalLB pool.

---

## 4) Install Argo CD (chart) with a fixed MetalLB IP

Put your fixed IP and settings in `gitops/argocd/values/custom-values.yaml`, for example:

```yaml
server:
  extraArgs:
    - --insecure
  service:
    type: LoadBalancer
    loadBalancerIP: 192.168.68.240
    annotations:
      metallb.universe.tf/address-pool: default-pool
  ingress:
    enabled: false
```

Install Argo CD from the vendored path:

```bash
helm upgrade --install argocd \
  ./gitops/argocd/vendor/argo-cd \
  --namespace argocd --create-namespace \
  -f ./gitops/argocd/values/base.yaml \
  -f ./gitops/argocd/values/custom-values.yaml
```

Check status:

```bash
kubectl -n argocd get pods
```

Retrieve the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

Access options:
- **LoadBalancer**: `kubectl -n argocd get svc argocd-server` and use the external IP assigned by MetalLB

> Default username is `admin`.

---

## 5) Let Argo CD manage itself and MetalLB

This repo includes a root **Application** (`gitops/clusters/prod.yaml`) that scans the `gitops/` tree and applies any Application YAMLs.

Apply the root Application (after Argo CD is installed):
```bash
kubectl apply -f gitops/clusters/prod.yaml
kubectl -n argocd get applications
```

Argo CD will discover:
- `gitops/argocd/app.yaml` → Argo CD **self‑manages** (reconciles itself).
- `gitops/metallb/app-metallb.yaml` (wave 0) → installs MetalLB.
- `gitops/metallb/app-metallb-config.yaml` (wave 1) → applies your pool/L2Advertisement.

> With the **vendor symlinks** (`vendor/argo-cd`, `vendor/metallb`) you will not need to change `app.yaml` paths on every version bump—just update `vendor-versions.env` and re-run the vendoring script.

---

## Troubleshooting

- **Pods Pending**: Check node status and `kubectl -n kube-system get pods`. Inspect containerd logs (RKE2 uses containerd).
- **No External IP for LB**: Confirm MetalLB is running and your `IPAddressPool` range is valid and not overlapping DHCP.
- **Argo CD UI not reachable**: Verify Service type is `LoadBalancer` and that MetalLB assigned the IP; otherwise port‑forward first.

---

## Summary

You now have:
- A working **RKE2** control plane
- **MetalLB** providing LoadBalancer IPs
- **Argo CD** deployed via vendored charts and optionally self‑managed via the app‑of‑apps pattern
