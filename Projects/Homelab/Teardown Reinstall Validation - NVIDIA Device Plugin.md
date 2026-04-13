# Teardown / Reinstall Validation — NVIDIA Device Plugin

> Performed: 2026-04-13
> Node: `melody-beast` (10.0.0.7)
> Scripts: `~/src/home_infra/k8s-nvidia-device-plugin/`

## Summary

Initial install required fixing 6 root causes. Two teardown/reinstall cycles completed with **23/23 tests passed** (including vLLM inference) and zero failures across both runs.

| Cycle | Teardown | Residue checks | Install | Tests |
|---|---|---|---|---|
| 1 | Clean | 3/3 passed | Pass | **23/23 passed, 0 failed, 0 warnings** |
| 2 | Clean | 4/4 passed | Pass | **23/23 passed, 0 failed, 0 warnings** |

---

## Procedure

### Teardown
```bash
cd ~/src/home_infra/k8s-nvidia-device-plugin
./uninstall.sh
```

Removes: Helm release, RuntimeClass, device plugin pods.
Does NOT remove: node label `nvidia.com/gpu.present=true` (hardware fact), containerd config (pass `--revert-containerd` to also remove).

### Residue checks (run after teardown)
```bash
helm status nvidia-device-plugin -n kube-system
# Error: release: not found

kubectl get runtimeclass nvidia
# Error from server (NotFound)

kubectl get pods -n kube-system -l app.kubernetes.io/name=nvidia-device-plugin
# No resources found in kube-system namespace.

kubectl describe node melody-beast | grep -A5 'Allocatable:' | grep nvidia
# (empty — nvidia.com/gpu removed)
```

### Install
```bash
./install.sh
```

### Test
```bash
./test.sh
```

vLLM is included by default. To skip it: `./test.sh --skip-vllm`

---

## Test output (Cycle 2 — 2026-04-13 15:31:41)

```
  ── Prerequisites ──
    ✓ nvidia-smi available and functional
    ✓ GPU count: 2 (expected 2)
    ✓ nvidia-ctk available
    ✓ Docker active
    ✓ k3s active
    ✓ Kubernetes reachable
    ✓ Helm available
    ✓ containerd configured with NVIDIA runtime

  ── Device Plugin ──
    ✓ Helm release 'nvidia-device-plugin' installed
    ✓ RuntimeClass 'nvidia' exists
    ✓ Device plugin pods ready: 1/1
    ✓ Device plugin logs clean
    ✓ Device plugin reports successful GPU registration
    ✓ Node allocatable GPUs: 2 (expected 2)

  ── GPU Scheduling ──
    ✓ GPU pod scheduled and completed
    ✓ GPU detected in container
    ✓ nvidia-smi works in container

  ── GPU Isolation ──
    ✓ Both GPU pods completed
    ✓ Each pod got a distinct GPU (different UUIDs confirmed)

  ── vLLM Inference ──
    ✓ vLLM server ready (model loaded)
    ✓ vLLM inference returned a completion ("text":" blue.\nI'm not sure if")

  ── Integration with GPU Monitoring ──
    ✓ DCGM exporter still running
    ✓ DCGM metrics endpoint responding

  Results: 23 passed, 0 failed, 0 warnings
  All tests passed ✔
```

---

## What the tests cover

| Section | Tests | What's validated |
|---|---|---|
| **Prerequisites** | 8 | nvidia-smi, GPU count, nvidia-ctk, Docker, k3s, kubectl, Helm, containerd runtime config |
| **Device Plugin** | 6 | Helm release, RuntimeClass, pod readiness, log health, GPU registration message, node allocatable GPU count |
| **GPU Scheduling** | 3 | Pod schedules and completes, GPU visible in container, nvidia-smi works |
| **GPU Isolation** | 2 | Two pods complete, each assigned a distinct physical GPU (UUID verified) |
| **vLLM Inference** | 2 | vLLM server loads model, POST /v1/completions returns a real completion |
| **Integration** | 2 | DCGM exporter still running, metrics endpoint responding |

---

## Bugs found and fixed

### During initial install

| Bug | Impact | Fix |
|---|---|---|
| `sudo grep` missing in containerd check | Config check silently failed (config.toml is root-owned) — containerd config re-ran on every install | Changed `grep` to `sudo grep` |
| Missing `nvidia.com/gpu.present=true` node label | Chart v0.17.1 nodeAffinity requires this label; without it `desiredNumberScheduled: 0`; `helm --wait` returned success vacuously | Added `kubectl label node ... nvidia.com/gpu.present=true --overwrite` to `deploy_device_plugin()` |
| Wrong image registry in values.yaml | `docker.io/nvidia/k8s-device-plugin:v0.17.1` not found — NVIDIA moved to nvcr.io | Set `image.repository: "nvcr.io/nvidia/k8s-device-plugin"` |
| Test pods missing `runtimeClassName: nvidia` | Pods used plain runc; nvidia-smi not injected into container | Added `--overrides` JSON with `runtimeClassName: nvidia` + GPU resource limit to all `kubectl run` calls |
| Wrong label selector for device plugin pods | `app=nvidia-device-plugin` doesn't match v0.17.1 chart labels | Changed to `app.kubernetes.io/name=nvidia-device-plugin` throughout |
| `grep -c \|\| echo "0"` bug | `grep -c` exits 1 on 0 matches → `\|\| echo "0"` fires → variable gets `"0\n0"` → integer comparison syntax error | Changed to `{ grep -c ... \|\| true; }` with `${var:-0}` fallback |
| `helm list --name` invalid in Helm 3 | uninstall.sh didn't filter by release name | Replaced with `helm status "$HELM_RELEASE"` |
| Missing `CONTAINERD_CONFIG` in uninstall.sh | `set -u` crash on `--revert-containerd` path | Added variable definition alongside `CONTAINERD_DROPIN` |
| Unclosed quote in `log` call (uninstall.sh) | Bash parse error, script exits immediately | Fixed closing quote |
| `kubectl run --limits` / `--resources` removed in kubectl v1.34.6 | All test pods failed to launch | Switched to `--overrides` JSON for all `kubectl run` calls |

### During teardown/reinstall cycle 1

| Bug | Impact | Fix |
|---|---|---|
| Pod readiness check was one-shot immediately after `helm --wait` | `helm --wait` returns once DaemonSet is created; pod may still be scheduling → install exits with `0/1 not ready` | Added 60s retry loop (5s interval) polling pod readiness in `install.sh` |
| GPU allocatable check was one-shot immediately after pod became Ready | Device plugin registers with kubelet via gRPC after pod readiness; propagation takes 10–30s → check always fires during that window | Added 60s retry loop polling `nvidia.com/gpu` in `kubectl describe node` in both `install.sh` and `test.sh` |
| vLLM `--gpu-memory-utilization` default (0.9) exceeds available GPU memory | DCGM and other processes consume ~41 GiB on GPU 0, leaving ~54 GiB free; vLLM tried to claim 85 GiB → `Engine core initialization failed` | Added `--gpu-memory-utilization 0.3` to vLLM pod args (~28 GiB, fits in 54 GiB free) |

---

## Conclusion

The NVIDIA Device Plugin install/test/uninstall scripts are fully repeatable. Two teardown → reinstall cycles completed with 23/23 tests passing and zero manual intervention. The test suite covers every critical path: prerequisites, Helm deployment, RuntimeClass, pod and GPU resource readiness (with retry), GPU scheduling, physical GPU isolation by UUID, real vLLM model inference, and DCGM monitoring integration.

## See Also

- [[Teardown Reinstall Validation]] — index of all component reports
- [[NVIDIA Device Plugin]] — full setup guide, architecture, and troubleshooting
- [[GPU Monitoring]] — DCGM exporter
