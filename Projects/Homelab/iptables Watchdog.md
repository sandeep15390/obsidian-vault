# iptables Watchdog — k3s / Tailscale Conflict

## Problem

When Tailscale is installed or restarted on the same host as k3s, it can overwrite flannel's iptables rules, breaking pod-to-service-CIDR routing. This causes all Traefik ingresses to return 404 and Traefik floods its logs with connection errors.

See also: [[Kubernetes#All ingresses return 404 / Traefik loses API server connectivity]]

---

## Symptoms

- All Traefik ingresses return **404**
- Traefik logs show repeated reflector errors:
  ```
  dial tcp 10.43.0.1:443: connect: no route to host
  ```
- `curl` to any ingress host returns 404 or connection refused
- Pods themselves are running fine (`kubectl get pods` works)

---

## Root Cause

Tailscale manages its own iptables rules for routing traffic through the WireGuard tunnel. When it starts or restarts, it can overwrite or conflict with the iptables rules that flannel (k3s CNI) uses for pod-to-service networking (the `10.43.0.0/16` service CIDR).

This breaks the route from Traefik pods to the Kubernetes API server (`10.43.0.1:443`), which Traefik needs for watching ingress resources.

---

## Manual Recovery

```bash
# 1. Verify the issue — should fail with "no route to host"
kubectl exec -n kube-system deploy/traefik -- wget -qO- --timeout=2 https://10.43.0.1:443 2>&1 || true

# 2. Restart k3s to restore flannel iptables rules
sudo systemctl restart k3s

# 3. Wait for pods to stabilize (~30s)
kubectl get pods -A

# 4. Verify Traefik is healthy
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik --tail=10
```

---

## Watchdog Script

A systemd timer or cron job that detects the broken state and auto-recovers.

### `/usr/local/bin/k3s-iptables-watchdog.sh`

```bash
#!/bin/bash
# k3s-iptables-watchdog.sh
#
# Detects when Tailscale has broken k3s flannel iptables rules
# by checking if Traefik can reach the Kubernetes API server.
# If broken, restarts k3s to restore routing.

set -euo pipefail

LOG_TAG="k3s-watchdog"
SERVICE_CIDR_GW="10.43.0.1"

# Check if k3s is running at all
if ! systemctl is-active --quiet k3s; then
  exit 0
fi

# Check if the service CIDR gateway is routable
# Use a quick TCP probe on port 443 (API server)
if timeout 3 bash -c "echo > /dev/tcp/${SERVICE_CIDR_GW}/443" 2>/dev/null; then
  exit 0
fi

# Service CIDR is unreachable — likely iptables conflict
logger -t "$LOG_TAG" "WARNING: ${SERVICE_CIDR_GW}:443 unreachable. Restarting k3s to restore flannel iptables rules."
systemctl restart k3s
logger -t "$LOG_TAG" "k3s restarted successfully."
```

### Systemd timer (recommended over cron)

**`/etc/systemd/system/k3s-iptables-watchdog.service`**
```ini
[Unit]
Description=Check k3s/Tailscale iptables health
After=k3s.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/k3s-iptables-watchdog.sh
```

**`/etc/systemd/system/k3s-iptables-watchdog.timer`**
```ini
[Unit]
Description=Run k3s iptables watchdog every 2 minutes

[Timer]
OnBootSec=60
OnUnitActiveSec=120

[Install]
WantedBy=timers.target
```

### Install

```bash
sudo cp k3s-iptables-watchdog.sh /usr/local/bin/
sudo chmod +x /usr/local/bin/k3s-iptables-watchdog.sh

# Copy unit files to /etc/systemd/system/ then:
sudo systemctl daemon-reload
sudo systemctl enable --now k3s-iptables-watchdog.timer

# Verify
sudo systemctl list-timers | grep k3s
journalctl -t k3s-watchdog
```

---

## When Does This Happen?

- `sudo systemctl restart tailscaled`
- `sudo tailscale up` (first time or re-auth)
- System reboot where Tailscale starts before or alongside k3s
- Tailscale auto-update that restarts the daemon

---

## Status

- [ ] Deploy watchdog script to `melody-beast`
- [ ] Verify watchdog detects simulated failure
- [ ] Add watchdog install to bootstrap checklist
