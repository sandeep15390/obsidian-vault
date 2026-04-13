# Homelab — Commands

Quick reference for bootstrap, startup, and operational commands.

---

## Bootstrap

```bash
# Full metrics stack (Prometheus + Alloy + Grafana datasource)
cd ~/src/home_infra/metrics && ./install.sh

# GPU monitoring stack (dcgm-exporter + Alloy patch + Grafana dashboard)
cd ~/src/home_infra/metrics/gpu && ./install.sh

# NVIDIA Device Plugin (GPU scheduling in k8s)
cd ~/src/home_infra/k8s-nvidia-device-plugin && ./install.sh

# Logging stack (Loki + Alloy + Grafana)
cd ~/src/home_infra/logging && ./install.sh

# Tailscale services (grafana, loki, prometheus)
cd ~/src/home_infra/tailscale && sudo ./apply.sh
```

---

## NVIDIA Device Plugin

```bash
# Install (containerd check, node label, RuntimeClass, Helm, smoke test, full test suite)
cd ~/src/home_infra/k8s-nvidia-device-plugin
sudo ./setup-permissions.sh   # one-time: sudoers drop-in + Claude Code settings
./install.sh 2>&1 | tee install.log

# Run full test suite (23 tests, vLLM included by default)
./test.sh

# Skip vLLM (faster, if image not yet pulled)
./test.sh --skip-vllm

# Override vLLM model
./test.sh --vllm-model Qwen/Qwen2.5-0.5B

# Diagnostic snapshot (read-only, safe to share)
./diag.sh 2>&1

# Uninstall (keeps containerd config + node label)
./uninstall.sh

# Uninstall and revert containerd config
./uninstall.sh --revert-containerd

# Quick GPU pod (requires device plugin installed)
kubectl run gpu-test --namespace default \
  --image=nvidia/cuda:12.8.1-base-ubuntu22.04 --restart=Never \
  --overrides='{"spec":{"runtimeClassName":"nvidia","containers":[{"name":"gpu-test","image":"nvidia/cuda:12.8.1-base-ubuntu22.04","command":["nvidia-smi"],"resources":{"limits":{"nvidia.com/gpu":"1"}}}]}}'
```

---

## GPU Monitoring

```bash
# Status
systemctl status dcgm-exporter

# View logs
journalctl -fu dcgm-exporter

# Quick sanity check — confirm both GPUs are exporting metrics
curl -s http://localhost:9400/metrics | grep DCGM_FI_DEV_GPU_UTIL

# Run full test suite (38 tests)
cd ~/src/home_infra/metrics/gpu && ./test.sh

# Reinstall / update dashboard only (no service restart)
cd ~/src/home_infra/metrics/gpu && ./install.sh --dashboard-only

# Tear down GPU monitoring (keeps base metrics stack)
cd ~/src/home_infra/metrics/gpu && ./uninstall.sh

# Tear down and keep Grafana dashboard
cd ~/src/home_infra/metrics/gpu && ./uninstall.sh --keep-dashboard
```

---

## Metrics Stack

```bash
# Install / upgrade
cd ~/src/home_infra/metrics && ./install.sh

# Run test suite (37+ tests)
cd ~/src/home_infra/metrics && ./test.sh

# Tear down (keep PVC data)
cd ~/src/home_infra/metrics && ./uninstall.sh --force

# Tear down completely (delete all data)
cd ~/src/home_infra/metrics && ./uninstall.sh --delete-data --delete-namespace --force
```

---

## Logging Stack

```bash
# Install / upgrade
cd ~/src/home_infra/logging && ./install.sh

# Run test suite (34 tests)
cd ~/src/home_infra/logging && ./test.sh

# Tear down (keep data)
cd ~/src/home_infra/logging && ./uninstall.sh --force

# Tear down completely
cd ~/src/home_infra/logging && ./uninstall.sh --delete-data --delete-namespace --force
```

---

## Startup

```bash
# Start Obsidian
obsidian
```

---

## Logs

```bash
# Tail dcgm-exporter logs
journalctl -fu dcgm-exporter

# Tail k3s logs
journalctl -fu k3s

# Tail a k8s pod's logs
kubectl logs -f <pod-name> -n <namespace>

# Tail all logs in a namespace
kubectl logs -f -l app=<label> -n <namespace> --max-log-requests=10
```

---

## Kubernetes

```bash
# List all pods across all namespaces
kubectl get pods -A

# Watch pod status
kubectl get pods -n metrics -w

# Describe a pod (events, resource requests)
kubectl describe pod <pod-name> -n <namespace>

# Restart a DaemonSet
kubectl rollout restart daemonset/<name> -n <namespace>

# Port-forward Prometheus
kubectl port-forward svc/prometheus-server 9090:80 -n metrics

# Port-forward Grafana
kubectl port-forward svc/grafana 3000:80 -n logging
```

---

## Network

```bash
# Quick connectivity check
ping -c 4 <host>

# Check Tailscale status
tailscale status

# Check registered Tailscale services
sudo tailscale serve get-config --all
```

---

## Useful One-Liners

```bash
# Watch GPU utilization live (requires nvidia-smi)
watch -n 1 nvidia-smi

# Check both GPUs quickly
nvidia-smi --query-gpu=index,name,utilization.gpu,memory.used,memory.total,temperature.gpu,power.draw --format=csv,noheader,nounits

# Check if dcgm-exporter container is running
sudo docker ps | grep dcgm

# Check Alloy targets (GPU scrape should show health=up)
kubectl port-forward svc/alloy-metrics 12345:12345 -n metrics &
curl -s http://localhost:12345/targets | python3 -m json.tool | grep -A5 dcgm
```
