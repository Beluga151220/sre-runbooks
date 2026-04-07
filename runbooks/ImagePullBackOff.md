# ImagePullBackOff

## Mô tả
Pod không pull được image từ registry. K8s retry với exponential backoff.

## Nguyên nhân phổ biến
1. Tên image hoặc tag sai
2. Image không tồn tại trên registry
3. Registry private nhưng thiếu imagePullSecret
4. Credentials hết hạn
5. Registry không accessible từ cluster (network/firewall)

## Điều tra

### Bước 1 — Xem lỗi cụ thể
```bash
kubectl describe pod <pod-name> -n <namespace>
# Tìm phần Events:
# Failed to pull image: ...
# Error: ErrImagePull
```

### Bước 2 — Xác định loại lỗi

**Lỗi "manifest unknown"** → Tag không tồn tại
```
Failed to pull image "nginx:version-xyz": 
manifest for nginx:version-xyz not found
```

**Lỗi "unauthorized"** → Thiếu credentials
```
Failed to pull image "vcr.vngcloud.vn/myrepo/app:latest": 
unauthorized: authentication required
```

**Lỗi "connection refused" / timeout** → Registry không accessible
```
Failed to pull image: dial tcp: connection refused
```

## Fix

### Tag sai — sửa image name
```bash
# Kiểm tra image tồn tại
docker pull <image>:<tag>

# Sửa deployment
kubectl set image deployment/<name> \
  <container>=<correct-image>:<correct-tag> \
  -n <namespace>
```

### Thiếu imagePullSecret — tạo secret
```bash
# Với VNG Container Registry
kubectl create secret docker-registry vcr-secret \
  --docker-server=vcr.vngcloud.vn \
  --docker-username=<username> \
  --docker-password=<password> \
  -n <namespace>

# Gắn vào deployment
kubectl patch deployment <name> -n <namespace> \
  -p '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"vcr-secret"}]}}}}'
```

### Credentials hết hạn — update secret
```bash
# Xóa secret cũ và tạo lại
kubectl delete secret vcr-secret -n <namespace>
kubectl create secret docker-registry vcr-secret \
  --docker-server=vcr.vngcloud.vn \
  --docker-username=<new-username> \
  --docker-password=<new-password> \
  -n <namespace>

# Restart deployment để pull lại
kubectl rollout restart deployment/<name> -n <namespace>
```

### Registry không accessible — kiểm tra network
```bash
# Từ trong cluster
kubectl run debug --image=busybox -it --rm --restart=Never \
  -- wget -O- https://vcr.vngcloud.vn/v2/

# Kiểm tra DNS
kubectl run debug --image=busybox -it --rm --restart=Never \
  -- nslookup vcr.vngcloud.vn
```

## Prometheus queries
```promql
# Pods đang ImagePullBackOff
kube_pod_container_status_waiting_reason{reason="ImagePullBackOff"}

# Pods đang ErrImagePull
kube_pod_container_status_waiting_reason{reason="ErrImagePull"}
```

## Verify
```bash
kubectl get pods -n <namespace> -w
# Chờ STATUS chuyển từ ImagePullBackOff → Running
```
