# PrivilegedContainer

## Mô tả
Có container đang chạy với privileged=true hoặc hostPID/hostNetwork — đây là rủi ro bảo mật nghiêm trọng, container có thể escape ra host.

## Dấu hiệu nhận biết
- Alert: `KubernetesPrivilegedContainerRunning`
- Pod spec có `securityContext.privileged: true`
- Container mount `/proc`, `/sys`, hoặc host filesystem
- Container dùng `hostPID: true` hoặc `hostNetwork: true`

## Điều tra

### Bước 1 — Tìm tất cả privileged containers
```bash
# Tìm privileged containers trong toàn cluster
kubectl get pods -A -o json | jq -r '
  .items[] |
  select(.spec.containers[].securityContext.privileged == true) |
  "\(.metadata.namespace)/\(.metadata.name)"'

# Tìm pods dùng hostPID hoặc hostNetwork
kubectl get pods -A -o json | jq -r '
  .items[] |
  select(.spec.hostPID == true or .spec.hostNetwork == true) |
  "\(.metadata.namespace)/\(.metadata.name): hostPID=\(.spec.hostPID) hostNetwork=\(.spec.hostNetwork)"'
```

### Bước 2 — Xem ai tạo pod đó
```bash
kubectl describe pod <pod-name> -n <namespace>
# Xem: Labels, Annotations, Controlled By

kubectl get events -n <namespace> \
  --field-selector involvedObject.name=<pod-name>
```

### Bước 3 — Kiểm tra container đang làm gì
```bash
# Xem processes trong container
kubectl exec -it <pod> -n <namespace> -- ps aux

# Xem mounted volumes
kubectl exec -it <pod> -n <namespace> -- mount | grep -v tmpfs

# Xem network interfaces
kubectl exec -it <pod> -n <namespace> -- ip addr
```

## Hành động khắc phục

### Xóa pod ngay nếu không được phép
```bash
# Xóa pod (nếu là standalone pod)
kubectl delete pod <pod-name> -n <namespace>

# Nếu do deployment tạo ra → sửa deployment
kubectl edit deployment <deployment-name> -n <namespace>
# Xóa/sửa: spec.template.spec.containers[].securityContext
```

### Sửa securityContext đúng cách
```yaml
# Deployment spec đúng chuẩn security:
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: app
        securityContext:
          privileged: false          # Không privileged
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true  # Read-only filesystem
          capabilities:
            drop: [ALL]             # Drop tất cả Linux capabilities
            add: [NET_BIND_SERVICE] # Chỉ add những gì thực sự cần
```

### Áp dụng PodSecurity / OPA Gatekeeper
```bash
# Bật Pod Security Standards
kubectl label namespace <namespace> \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted

# Verify label đã apply
kubectl get namespace <namespace> --show-labels
```

### Tạo OPA policy ngăn privileged containers
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPPrivilegedContainer
metadata:
  name: no-privileged-containers
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    excludedNamespaces: ["kube-system"]
```

## Prevention
- Bật PodSecurity Admission Controller (K8s >= 1.23)
- Dùng OPA Gatekeeper hoặc Kyverno để enforce policy
- Review deployment manifests trong CI/CD pipeline
- Không cho developer deploy trực tiếp lên production

## Prometheus queries
```promql
# Số privileged containers đang chạy
kube_pod_container_info{container!=""} * on(pod, namespace) group_left()
  kube_pod_spec_volumes_persistentvolumeclaims_info

# Alert rule mẫu
- alert: PrivilegedContainerRunning
  expr: kube_pod_container_security_context_privileged > 0
  for: 0m
  labels:
    severity: critical
  annotations:
    summary: "Privileged container running: {{ $labels.pod }}"
```
