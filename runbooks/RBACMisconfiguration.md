# RBACMisconfiguration

## Mô tả
RBAC cấu hình sai — service account có quyền quá rộng, hoặc ai đó có cluster-admin không cần thiết.

## Dấu hiệu nhận biết
- Service account có wildcard (*) trong verbs hoặc resources
- Nhiều subjects được bind với cluster-admin
- Pod có thể tạo/xóa pods trong namespace khác
- Alert: `KubernetesRBACPrivilegeEscalation`

## Điều tra

### Bước 1 — Tìm tất cả cluster-admin bindings
```bash
kubectl get clusterrolebindings -o json | jq -r '
  .items[] |
  select(.roleRef.name == "cluster-admin") |
  "Binding: \(.metadata.name)\n  Subjects: \([.subjects[]? | "\(.kind)/\(.name)"] | join(", "))\n"'
```

### Bước 2 — Tìm roles có quyền wildcard nguy hiểm
```bash
# Tìm roles có * trong verbs
kubectl get clusterroles -o json | jq -r '
  .items[] |
  select(.rules[]?.verbs[]? == "*") |
  "\(.metadata.name): \([.rules[]? | select(.verbs[]? == "*") | .resources] | flatten | join(", "))"'

# Tìm roles có thể đọc secrets
kubectl get clusterroles -o json | jq -r '
  .items[] |
  select(.rules[]?.resources[]? == "secrets") |
  .metadata.name'
```

### Bước 3 — Kiểm tra quyền của service account cụ thể
```bash
# Xem tất cả quyền của một SA
kubectl auth can-i --list \
  --as=system:serviceaccount:<namespace>:<sa-name>

# Check quyền cụ thể
kubectl auth can-i delete pods \
  --as=system:serviceaccount:<namespace>:<sa-name> -A
```

### Bước 4 — Tìm pods dùng default service account
```bash
# Default SA thường có quyền rộng hơn cần thiết
kubectl get pods -A -o json | jq -r '
  .items[] |
  select(.spec.serviceAccountName == "default" or
         .spec.serviceAccountName == null) |
  "\(.metadata.namespace)/\(.metadata.name)"'
```

## Hành động khắc phục

### Revoke cluster-admin không cần thiết
```bash
# Xóa clusterrolebinding nguy hiểm
kubectl delete clusterrolebinding <binding-name>

# Tạo role giới hạn thay thế
kubectl create role namespace-admin \
  --verb=* \
  --resource=pods,deployments,services \
  -n <specific-namespace>

kubectl create rolebinding namespace-admin-binding \
  --role=namespace-admin \
  --serviceaccount=<namespace>:<sa-name> \
  -n <specific-namespace>
```

### RBAC theo nguyên tắc Least Privilege
```yaml
# Role tốt: chỉ những gì app cần
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: sre-agent
  namespace: sre-agent
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "events", "nodes", "services"]
  verbs: ["get", "list", "watch"]      # Chỉ read
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "patch"]      # Không có delete
```

### Disable automountServiceAccountToken
```yaml
# Với pods không cần gọi API
spec:
  automountServiceAccountToken: false  # Tắt auto-mount token
  containers:
  - name: app
    ...
```

### Audit RBAC định kỳ
```bash
# Script kiểm tra RBAC hàng tuần
kubectl get clusterrolebindings -o json \
  | jq -r '.items[] | "\(.metadata.name)\t\(.roleRef.name)\t\([.subjects[]? | .name] | join(","))"' \
  | grep -v "^system:" \
  | sort > rbac-audit-$(date +%Y%m%d).txt
```

## Nguyên tắc RBAC an toàn
1. **Least Privilege**: Chỉ cấp quyền tối thiểu cần thiết
2. **Namespace scope**: Dùng Role/RoleBinding thay vì ClusterRole/ClusterRoleBinding
3. **Không dùng default SA**: Tạo SA riêng cho mỗi app
4. **Tắt automount**: Với pods không cần API access
5. **Audit định kỳ**: Review RBAC mỗi tháng
6. **Không dùng wildcard**: Tránh `verbs: ["*"]` và `resources: ["*"]`

## Prometheus queries
```promql
# Alert khi có cluster-admin mới
changes(kube_clusterrolebinding_info{role_name="cluster-admin"}[1h]) > 0
```
