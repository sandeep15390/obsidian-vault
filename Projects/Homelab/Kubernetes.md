# Homelab Kubernetes (k3s)

## Status
- [x] Install k3s on Ubuntu node
- [x] Configure kubeconfig
- [x] Fix kubeconfig permissions (no sudo needed)
- [x] Install Helm
- [x] Deploy first Helm chart (logging stack)
- [ ] Verify remote kubectl access from LAN
- [ ] Set up external access (Cloudflare Tunnel or port forward)
- [ ] Verify Traefik ingress is working end-to-end

---

## Cluster Info

| Property | Value |
|---|---|
| Node | `melody-beast` |
| IP | `192.168.1.97` |
| k3s version | `v1.34.6+k3s1` |
| Role | `control-plane` |
| Ingress | Traefik (built-in) |
| Storage | local-path-provisioner (built-in) |
| LB | ServiceLB / klipper-lb (built-in) |

---

## Network Architecture

```mermaid
graph TD
    Internet["🌐 Internet"]
    Router["🔀 Home Router"]
    LAN["🏠 LAN Devices\n(laptops, phones, etc.)"]
    Tunnel["☁️ Cloudflare Tunnel\n(optional, external access)"]

    subgraph Node["Ubuntu Node — melody-beast (192.168.1.97)"]
        k3s["k3s API Server\n:6443"]
        Traefik["Traefik Ingress\n:80 / :443"]
        ServiceLB["ServiceLB\n(LoadBalancer)"]
        subgraph Workloads["Workloads"]
            W1["logging stack"]
            W2["..."]
        end
    end

    Internet -->|port forward| Router
    Router --> LAN
    Router -->|optional| Tunnel
    LAN -->|kubectl :6443| k3s
    LAN -->|HTTP/HTTPS| Traefik
    Tunnel --> Traefik
    Traefik --> Workloads
    ServiceLB --> Workloads
```

---

## Quick Commands

```bash
# Check cluster
kubectl get nodes
kubectl get pods -A

# Get kubeconfig (if permissions reset)
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
export KUBECONFIG=~/.kube/config
```

---

## Install Steps (completed)

### 1. Install k3s

```bash
curl -sfL https://get.k3s.io | sh -
```

### 2. Configure kubeconfig

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc && source ~/.bashrc
```

### 3. Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 4. Access from other LAN machines

On the remote machine:
```bash
scp melody@192.168.1.97:~/.kube/config ~/.kube/config
# Edit ~/.kube/config and replace 127.0.0.1 with 192.168.1.97
kubectl get nodes
```

---

## Troubleshooting

### All ingresses return 404 / Traefik loses API server connectivity

**Symptom:** Every request through Traefik returns `404 page not found`, including rules that were previously working. Traefik logs flood with:
```
Failed to watch: dial tcp 10.43.0.1:443: connect: no route to host
```

**Cause:** The k3s cluster service network (`10.43.0.0/16`) becomes unreachable from inside pods. This is typically caused by Tailscale (or any tool that modifies iptables) flushing or overwriting the kernel routing rules that flannel sets up.

**Fix:** Restart k3s — this reinitialises flannel and restores all service network iptables rules:
```bash
sudo systemctl restart k3s
# Wait ~30s then verify
kubectl get pods -A
kubectl logs -n kube-system deployment/traefik --tail=5
```

Traefik should reconnect to the API server, reload all Ingress/IngressRoute rules, and start routing correctly.

**Check for this problem:**
```bash
# From the Traefik pod — if this fails, the service network is broken
kubectl exec -n kube-system deployment/traefik -- wget -qO- https://10.43.0.1:443 2>&1 | head -3
```

---

### tailscale serve rewrites the Host header

When using `tailscale serve https / http://127.0.0.1:80` to expose a service through the tailnet, `tailscale serve` does **not** forward the original client hostname. It replaces the `Host` header with the backend address:

| tailscale serve target | Host header forwarded to backend |
|---|---|
| `http://127.0.0.1:80` | `Host: 127.0.0.1` |
| `http://127.0.0.1:9998` | `Host: 127.0.0.1:9998` |

This means Traefik ingress rules matching `host: melody-beast.tailc98a25.ts.net` will never fire. Use a Traefik **IngressRoute CRD** matching `Host("127.0.0.1")` instead (standard Kubernetes Ingress rejects IP addresses as hosts):

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: my-service-tailscale
  namespace: my-namespace
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`127.0.0.1`)
      kind: Rule
      services:
        - name: my-service
          port: 80
```

---

## External Access (Optional)

**Option A — Cloudflare Tunnel** (no port forwarding needed):
```bash
cloudflared tunnel create homelab
cloudflared tunnel route dns homelab <your-domain>
cloudflared tunnel run homelab
```

**Option B — Router port forward**
- Forward ports `80` and `443` on your router to `192.168.1.97`
