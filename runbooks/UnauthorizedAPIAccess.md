# UnauthorizedAPIAccess

## Mô tả
Có request trái phép đến Kubernetes API server — có thể là token bị lộ, service account bị lạm dụng, hoặc tấn công từ bên ngoài.

## Dấu hiệu nhận biết
- Audit log có nhiều lỗi 401/403
- Service account lạ gọi API không thuộc scope
- Alert: `KubernetesAPIServerErrors` hoặc `AbnormalAPIRequestRate`

## Điều tra

### Bước 1 — Xem audit logs
```bash
# Nếu audit logging đã bật
kubectl logs -n kube-system kube-apiserver-<node> | grep '"code":403'
kubectl logs -n kube-system kube-apiserver-<node> | grep '"code":401'

# Tìm IP nguồn bất thường
kubectl logs -n kube-system kube-apiserver-<node> \
  | grep -E '"verb":"(create|delete|patch)"' \
  | grep -v '"user":{"username":"system:'
```

### Bước 2 — Kiểm tra service accounts có quyền rộng
```bash
# Liệt kê ClusterRoleBinding nguy hiểm
kubectl get clusterrolebindings -o json | jq -r '
  .items[] |
  select(.roleRef.name == "cluster-admin") |
  "\(.metadata.name): \(.subjects[].name)"'

# Xem RBAC của một service account cụ thể
kubectl auth can-i --list \
  --as=system:serviceaccount:<namespace>:<sa-name>
```

### Bước 3 — Kiểm tra secrets bị truy cập bất thường
```bash
kubectl get events -A | grep -i "secret"

# List secrets trong namespace nghi ngờ
kubectl get secrets -n <namespace> -o yaml
```

### Bước 4 — Prometheus queries
```promql
# Request rate vào API server theo verb
rate(apiserver_request_total{verb=~"create|delete|patch|update"}[5m])

# Số lỗi 403
rate(apiserver_request_total{code="403"}[5m])

# Số lỗi 401
rate(apiserver_request_total{code="401"}[5m])
```

## Hành động khắc phục

### Revoke token bị nghi lộ
```bash
# Xóa secret chứa token cũ — K8s sẽ tự tạo token mới
kubectl delete secret <sa-token-secret> -n <namespace>

# Hoặc rotate service account
kubectl delete serviceaccount <sa-name> -n <namespace>
kubectl create serviceaccount <sa-name> -n <namespace>
```

### Hạn chế quyền ClusterRole
```bash
# Xóa binding nguy hiểm
kubectl delete clusterrolebinding <binding-name>

# Tạo Role giới hạn thay vì dùng cluster-admin
kubectl create role limited-role \
  --verb=get,list \
  --resource=pods \
  -n <namespace>
```

### Block IP bất thường (NetworkPolicy)
```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-external
  namespace: <namespace>
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector: {}
EOF
```

## Verify
```bash
kubectl auth can-i --list --as=system:serviceaccount:<namespace>:<sa>
# Chỉ thấy các quyền cần thiết, không có wildcard (*)
```

## Escalate nếu
- Phát hiện lateral movement giữa namespaces
- Có pod lạ được tạo với quyền privileged
- Token bị dùng từ IP ngoài cluster
