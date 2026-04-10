# SecretExposed

## Mô tả
Kubernetes Secret bị lộ — có thể do mount sai, log in ra stdout, hoặc RBAC quá rộng cho phép đọc secrets.

## Dấu hiệu nhận biết
- Secret value xuất hiện trong pod logs
- Pod mount secret không cần thiết
- Service account có quyền `get secrets` trong namespace rộng
- Alert: `KubernetesSecretAccessedByUnusualUser`

## Điều tra

### Bước 1 — Tìm secrets bị log ra stdout
```bash
# Tìm trong logs các pattern nghi là secret
kubectl logs <pod> -n <namespace> | grep -iE \
  "password|secret|token|key|credential|apikey"

# Tìm trong tất cả pods namespace
for pod in $(kubectl get pods -n <namespace> -o name); do
  echo "=== $pod ==="
  kubectl logs $pod -n <namespace> 2>/dev/null \
    | grep -iE "password|secret|token" | head -5
done
```

### Bước 2 — Kiểm tra ai có quyền đọc secrets
```bash
# Ai có thể đọc secrets trong namespace này?
kubectl get rolebindings,clusterrolebindings -A -o json \
  | jq -r '.items[] | select(.roleRef.name | test("secret|admin|edit")) | .metadata.name'

# Verify service account cụ thể
kubectl auth can-i get secrets \
  --as=system:serviceaccount:<namespace>:<sa-name> \
  -n <namespace>
```

### Bước 3 — Kiểm tra secrets được mount không cần thiết
```bash
kubectl get pods -n <namespace> -o json \
  | jq -r '.items[] | .metadata.name + ": " + (.spec.volumes[]? | select(.secret) | .secret.secretName)'
```

### Bước 4 — Xem secret được tạo/access gần đây
```bash
kubectl get events -n <namespace> \
  | grep -i secret | sort -k1 -r | head -20
```

## Hành động khắc phục

### Rotate secret ngay lập tức
```bash
# Xóa secret cũ
kubectl delete secret <secret-name> -n <namespace>

# Tạo secret mới với giá trị mới
kubectl create secret generic <secret-name> \
  --from-literal=password=NEW_STRONG_PASSWORD \
  -n <namespace>

# Restart pods đang dùng secret cũ
kubectl rollout restart deployment/<deployment> -n <namespace>
```

### Sửa code không log sensitive data
```bash
# Tạm thời scale down deployment
kubectl scale deployment <deployment> --replicas=0 -n <namespace>

# Sau khi fix code, redeploy
kubectl rollout restart deployment/<deployment> -n <namespace>
```

### Hạn chế quyền đọc secret
```bash
# Xóa quyền đọc secret khỏi role
kubectl edit role <role-name> -n <namespace>
# Xóa: - secrets từ resources

# Hoặc tạo role mới chỉ đọc những secret cần thiết
kubectl create role pod-reader \
  --verb=get \
  --resource=pods \
  -n <namespace>
```

### Bật Encryption at Rest (nếu chưa có)
```bash
# Kiểm tra encryption đã bật chưa
kubectl get apiserver -o yaml | grep encryptionConfig

# Xem encryption config
cat /etc/kubernetes/encryption-config.yaml
```

## Prevention
- Không log biến môi trường chứa secrets
- Dùng `secretKeyRef` thay vì hardcode trong env
- Bật `audit logging` để track secret access
- Dùng `Sealed Secrets` hoặc `Vault` cho secrets management
- Review RBAC định kỳ: không ai nên có quyền `list secrets` trong production

## Prometheus queries
```promql
# Số lần đọc secrets
rate(apiserver_request_total{
  resource="secrets",
  verb="get"
}[5m])
```
