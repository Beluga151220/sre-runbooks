# Runbook: Cilium Kube-proxy Replacement Troubleshooting

**Applies to:** VKS (GreenNode Kubernetes Service) với Cilium CNI  
**Severity:** Warning → Critical  
**Last updated:** 2026-04-29

---

## 1. Tổng quan

Cilium kube-proxy replacement cho phép Cilium xử lý Service load balancing thay thế hoàn toàn kube-proxy, sử dụng eBPF để forward traffic trực tiếp — giảm latency và tăng throughput.

Khi kube-proxy replacement **không hoạt động**:
- Service ClusterIP/NodePort không route đúng
- ExternalIPs không forward
- Health checks fail do mất kết nối
- Pods không reach được Services

---

## 2. Triệu chứng thường gặp

| Triệu chứng | Nguyên nhân có thể |
|---|---|
| Service ClusterIP không ping được | kube-proxy replacement disabled |
| NodePort timeout từ ngoài | BPF NodePort not loaded |
| Pod connect Service bị reset | Conntrack state mismatch |
| DNS timeout | CoreDNS Service unreachable |
| `cilium status` báo KubeProxy: Disabled | Config sai hoặc kernel không support |

---

## 3. Quick Diagnosis

```bash
# 3.1 Check Cilium status tổng quan
kubectl exec -n kube-system ds/cilium -- cilium status | grep -A5 "KubeProxy\|NodePort\|Masquerade"

# 3.2 Check kube-proxy replacement mode
kubectl exec -n kube-system ds/cilium -- cilium status --verbose | grep -i "kube-proxy"

# 3.3 Xem config hiện tại
kubectl get configmap -n kube-system cilium-config -o yaml | grep -E "kube-proxy-replacement|node-port|bpf"

# 3.4 Check BPF programs loaded
kubectl exec -n kube-system ds/cilium -- cilium bpf lb list | head -20

# 3.5 Check Services có được sync không
kubectl exec -n kube-system ds/cilium -- cilium service list | head -20

# 3.6 Check node ports
kubectl exec -n kube-system ds/cilium -- cilium bpf lb list --frontends
```

---

## 4. Root Cause & Fix

### 4.1 Kernel version không support

**Triệu chứng:**
```
level=warning msg="BPF kube-proxy replacement probe failed" error="kernel too old"
```

**Diagnosis:**
```bash
uname -r  # Cần >= 4.19.57 (strict mode: >= 5.10)
kubectl exec -n kube-system ds/cilium -- cilium status | grep "Kernel"
```

**Fix:**
```bash
# Không thể fix trực tiếp — cần upgrade kernel node
# Trên VKS: nâng version nodegroup lên node image mới hơn
# Hoặc disable strict mode:
kubectl edit configmap cilium-config -n kube-system
# Thêm: kube-proxy-replacement: "partial"
kubectl rollout restart ds/cilium -n kube-system
```

---

### 4.2 Cilium config sai — kube-proxy-replacement disabled

**Diagnosis:**
```bash
kubectl get configmap cilium-config -n kube-system -o yaml | grep kube-proxy-replacement
# Nếu ra: kube-proxy-replacement: "disabled" → đây là vấn đề
```

**Fix:**
```bash
kubectl edit configmap cilium-config -n kube-system
# Đổi: kube-proxy-replacement: "strict"  (hoặc "true" tùy Cilium version)
# Hoặc với Helm:
helm upgrade cilium cilium/cilium -n kube-system --reuse-values \
  --set kubeProxyReplacement=strict

kubectl rollout restart ds/cilium -n kube-system
kubectl rollout status ds/cilium -n kube-system
```

---

### 4.3 NodePort BPF program không load

**Triệu chứng:** NodePort service không accessible từ ngoài cluster.

**Diagnosis:**
```bash
# Check BPF NodePort programs
kubectl exec -n kube-system ds/cilium -- cilium bpf lb list --frontends | grep NodePort

# Check network device cilium đang listen
kubectl exec -n kube-system ds/cilium -- cilium status | grep "NodePort SNAT"

# Check devices config
kubectl get cm cilium-config -n kube-system -o yaml | grep -E "devices|node-port-bind"
```

**Fix:**
```bash
# Thêm devices config nếu thiếu
kubectl edit configmap cilium-config -n kube-system
# Thêm:
# devices: "eth0"  # hoặc interface chính của node
# nodeport-addresses: ""  # để auto detect

kubectl rollout restart ds/cilium -n kube-system
```

---

### 4.4 kube-proxy vẫn đang chạy song song — conflict

**Triệu chứng:** Cilium và kube-proxy cùng manage iptables → conflict rules.

**Diagnosis:**
```bash
# Check kube-proxy có đang chạy không
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# Check iptables rules conflict
kubectl exec -n kube-system ds/cilium -- iptables-save | grep -c KUBE
# Nếu > 0 và cilium dùng strict mode → conflict
```

**Fix:**
```bash
# Option 1: Scale down kube-proxy (nếu Cilium strict mode)
kubectl scale deployment kube-proxy -n kube-system --replicas=0
# Hoặc delete nếu dùng DaemonSet
kubectl delete ds kube-proxy -n kube-system

# Option 2: Dùng partial mode (giữ kube-proxy)
kubectl edit configmap cilium-config -n kube-system
# Đổi: kube-proxy-replacement: "partial"
kubectl rollout restart ds/cilium -n kube-system
```

---

### 4.5 Conntrack state mismatch sau restart

**Triệu chứng:** Connections bị reset sau khi Cilium restart hoặc node reboot.

**Diagnosis:**
```bash
# Check conntrack table
kubectl exec -n kube-system ds/cilium -- cilium bpf ct list global | head -20

# Monitor drops
kubectl exec -n kube-system ds/cilium -- cilium monitor --type drop -n 20
```

**Fix:**
```bash
# Flush conntrack table (gây brief disruption)
kubectl exec -n kube-system ds/cilium -- cilium bpf ct flush global

# Hoặc restart cilium để rebuild state
kubectl rollout restart ds/cilium -n kube-system
```

---

### 4.6 BPF map full — service không được add

**Triệu chứng:**
```
level=error msg="Unable to add service entry" error="map update: map is full"
```

**Diagnosis:**
```bash
# Check map usage
kubectl exec -n kube-system ds/cilium -- cilium bpf lb list | wc -l

# Check map capacity
kubectl exec -n kube-system ds/cilium -- \
  bpftool map show | grep -A5 "cilium_lb4_services"
```

**Fix:**
```bash
# Tăng map size
kubectl edit configmap cilium-config -n kube-system
# Thêm:
# bpf-lb-map-max: "65536"   # default 65536, tăng nếu cần
# bpf-map-dynamic-size-ratio: "0.0025"  # auto scale based on RAM

kubectl rollout restart ds/cilium -n kube-system
```

---

## 5. Verification sau fix

```bash
# 5.1 Check Cilium status
kubectl exec -n kube-system ds/cilium -- cilium status
# Expect: KubeProxyReplacement: Strict

# 5.2 Test Service connectivity
kubectl run test-pod --image=busybox --rm -it --restart=Never -- \
  wget -qO- http://<service-name>.<namespace>.svc.cluster.local

# 5.3 Test NodePort từ ngoài
curl http://<node-ip>:<node-port>

# 5.4 Check không còn drops
kubectl exec -n kube-system ds/cilium -- \
  cilium monitor --type drop -n 10 --timeout 10s

# 5.5 Verify tất cả Services được sync
kubectl exec -n kube-system ds/cilium -- cilium service list | wc -l
# Số lượng phải match: kubectl get svc --all-namespaces | wc -l
```

---

## 6. Escalation

Nếu sau các bước trên vẫn không fix được:

1. **Collect full diagnostics:**
```bash
kubectl exec -n kube-system ds/cilium -- cilium sysdump > cilium-sysdump.tar.gz
```

2. **Check Cilium version compatibility với K8s:**
```bash
kubectl exec -n kube-system ds/cilium -- cilium version
kubectl version --short
```

3. **Escalate lên:** GreenNode Support / Cilium Slack (#general hoặc #kube-proxy-replacement)

4. **Reference:**
   - https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/
   - https://docs.cilium.io/en/stable/operations/troubleshooting/

---

## 7. Prevention

```yaml
# cilium-config best practices cho VKS
kube-proxy-replacement: "strict"
bpf-map-dynamic-size-ratio: "0.0025"  # auto scale BPF maps
bpf-lb-map-max: "65536"
enable-node-port: "true"
node-port-mode: "snat"
enable-health-check-nodeport: "true"
enable-session-affinity: "true"
```

**Alert rule cần enable:** `CiliumKubeProxyReplacementNotActive` (xem cilium-alerts.yaml)
