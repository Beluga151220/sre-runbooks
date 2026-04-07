# CrashLoopBackOff

## Mô tả
Pod khởi động rồi crash liên tục. Kubernetes restart nhưng pod lại crash — tạo vòng lặp.

## Nguyên nhân phổ biến
1. Command/entrypoint sai hoặc không tồn tại trong image
2. Thiếu environment variable bắt buộc
3. Thiếu Secret hoặc ConfigMap
4. App lỗi khi khởi động (bug, config sai)
5. OOMKilled ngay lúc start (xem thêm OOMKilled.md)
6. Liveness probe fail quá sớm

## Điều tra

### Bước 1 — Xem log lỗi
```bash
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous   # log của lần crash trước
```

### Bước 2 — Xem lý do crash
```bash
kubectl describe pod <pod-name> -n <namespace>
# Tìm phần: Last State → Terminated → Reason + Exit Code
```

### Bước 3 — Kiểm tra events
```bash
kubectl get events -n <namespace> --field-selector involvedObject.name=<pod-name>
```

## Diễn giải Exit Code
| Exit Code | Ý nghĩa |
|-----------|---------|
| 0 | Thoát bình thường (không phải crash) |
| 1 | Lỗi chung của ứng dụng |
| 137 | OOMKilled (kill -9) |
| 139 | Segmentation fault |
| 143 | SIGTERM không được xử lý |
| 255 | Exit code không xác định |

## Fix

### Lỗi command/entrypoint
```bash
# Kiểm tra image có command đó không
docker run --rm <image> which <command>

# Sửa deployment
kubectl edit deployment <deployment-name> -n <namespace>
# Sửa spec.containers[].command hoặc args
```

### Thiếu env var / secret
```bash
# Kiểm tra secret tồn tại chưa
kubectl get secret <secret-name> -n <namespace>

# Tạo secret nếu thiếu
kubectl create secret generic <secret-name> \
  --from-literal=KEY=VALUE -n <namespace>
```

### Liveness probe fail sớm
```bash
# Tăng initialDelaySeconds trong deployment
kubectl edit deployment <deployment-name> -n <namespace>
# spec.containers[].livenessProbe.initialDelaySeconds: 30
```

## Verify sau fix
```bash
kubectl get pods -n <namespace> -w
# Chờ STATUS = Running, RESTARTS không tăng thêm
```

## Prometheus queries
```promql
# Số lần restart trong 1 giờ
increase(kube_pod_container_status_restarts_total{pod="<pod>"}[1h])

# Pods đang CrashLoop
kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}
```
