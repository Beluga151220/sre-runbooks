# ContainerEscape

## Mô tả
Container đang cố thoát ra host OS — đây là incident nghiêm trọng nhất trong K8s security.

## Dấu hiệu nhận biết
- Process trong container gọi syscall bất thường (mount, chroot, ptrace)
- Container mount `/proc/host`, `/sys`, `/dev`
- Falco alert: `Terminal shell in container` hoặc `Write below root`
- Process chạy với UID 0 và có `CAP_SYS_ADMIN`

## Điều tra NGAY LẬP TỨC

### Bước 1 — Xác định container bị nghi ngờ
```bash
# Tìm containers chạy root
kubectl get pods -A -o json | jq -r '
  .items[] |
  select(.spec.containers[].securityContext.runAsUser == 0 or
         .spec.containers[].securityContext.privileged == true) |
  "\(.metadata.namespace)/\(.metadata.name)"'

# Xem host mounts
kubectl get pods -A -o json | jq -r '
  .items[] |
  select(.spec.volumes[]?.hostPath) |
  "\(.metadata.namespace)/\(.metadata.name): \([.spec.volumes[]? | select(.hostPath) | .hostPath.path] | join(", "))"'
```

### Bước 2 — Thu thập bằng chứng TRƯỚC khi xóa
```bash
# Dump pod spec
kubectl get pod <pod-name> -n <namespace> -o yaml > evidence-pod-spec.yaml

# Dump logs
kubectl logs <pod-name> -n <namespace> > evidence-logs.txt
kubectl logs <pod-name> -n <namespace> --previous >> evidence-logs.txt

# Xem processes
kubectl exec <pod-name> -n <namespace> -- ps auxef > evidence-processes.txt

# Xem network connections
kubectl exec <pod-name> -n <namespace> -- ss -antp > evidence-network.txt

# Xem file system changes (nếu container còn sống)
kubectl exec <pod-name> -n <namespace> -- find / -newer /proc/1 -type f 2>/dev/null \
  | grep -v proc | head -100 > evidence-changed-files.txt
```

### Bước 3 — Kiểm tra host node
```bash
# SSH vào node chứa container đó
NODE=$(kubectl get pod <pod-name> -n <namespace> \
  -o jsonpath='{.spec.nodeName}')

ssh $NODE

# Kiểm tra processes lạ trên host
ps auxef | grep -v '\[' | grep -v '/usr/lib/systemd'

# Kiểm tra file mới tạo gần đây trên host
find / -newer /proc/1 -type f 2>/dev/null \
  | grep -v '/proc\|/sys\|/run\|/dev' | head -50

# Kiểm tra cron jobs mới
crontab -l
ls /etc/cron* /var/spool/cron 2>/dev/null
```

## Hành động khắc phục KHẨN CẤP

### 1. Isolate ngay — KHÔNG XÓA (cần bằng chứng)
```bash
# Block network của pod
kubectl label pod <pod-name> -n <namespace> quarantine=true

kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: quarantine-pod
  namespace: <namespace>
spec:
  podSelector:
    matchLabels:
      quarantine: "true"
  policyTypes: [Ingress, Egress]
EOF
```

### 2. Cordon node bị ảnh hưởng
```bash
# Không schedule pod mới lên node này
kubectl cordon <node-name>

# Drain pods (sau khi đã thu thập bằng chứng)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

### 3. Thu thập forensics từ node
```bash
# Chụp memory (nếu có tool)
# dd if=/proc/mem of=/tmp/memory.dump

# Lưu audit logs
journalctl -u kubelet --since "1 hour ago" > /tmp/kubelet-audit.log
```

### 4. Xóa pod sau khi đã có đủ bằng chứng
```bash
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force
```

### 5. Rotate credentials TOÀN BỘ
```bash
# Rotate tất cả secrets trong namespace bị ảnh hưởng
kubectl get secrets -n <namespace> -o name | while read secret; do
  echo "Review: $secret"
done

# Đặc biệt: rotate service account tokens
kubectl delete serviceaccount <sa-name> -n <namespace>
kubectl create serviceaccount <sa-name> -n <namespace>
```

## Prevention
- Bật Falco để detect runtime threats
- Dùng seccomp profile để restrict syscalls
- Enable AppArmor/SELinux
- Chạy containers với read-only filesystem
- Không cho mount host paths trong production

```yaml
# seccomp profile trong pod spec
securityContext:
  seccompProfile:
    type: RuntimeDefault  # Dùng default seccomp profile
  runAsNonRoot: true
  runAsUser: 1000
```

## Escalate ngay với
- Security team
- Management
- Incident response team
- Xem xét notify khách hàng nếu data bị ảnh hưởng
