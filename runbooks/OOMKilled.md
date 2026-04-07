# OOMKilled

## Mô tả
Pod bị Linux kernel kill vì vượt quá memory limit. Exit code 137.

## Dấu hiệu nhận biết
- `kubectl describe pod` → Last State: Terminated, Reason: OOMKilled
- Exit Code: 137
- Pod restart liên tục

## Điều tra

### Bước 1 — Xác nhận OOMKilled
```bash
kubectl describe pod <pod-name> -n <namespace>
# Tìm:
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137
```

### Bước 2 — Xem memory usage hiện tại
```bash
kubectl top pod <pod-name> -n <namespace>
kubectl top pod -n <namespace> --sort-by=memory
```

### Bước 3 — Xem memory limit đang set
```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath=\
'{.spec.containers[*].resources}'
```

### Bước 4 — Query Prometheus xem trend
```promql
# Memory usage thực tế (bytes)
container_memory_working_set_bytes{pod="<pod>", container!=""}

# Memory limit
kube_pod_container_resource_limits{pod="<pod>", resource="memory"}

# % sử dụng so với limit
container_memory_working_set_bytes{pod="<pod>"}
  / kube_pod_container_resource_limits{pod="<pod>", resource="memory"}
```

## Fix

### Tăng memory limit (fix tạm thời)
```bash
kubectl set resources deployment <deployment-name> \
  -n <namespace> \
  --limits=memory=512Mi \
  --requests=memory=256Mi
```

### Hoặc edit trực tiếp
```bash
kubectl edit deployment <deployment-name> -n <namespace>
# Sửa:
# resources:
#   requests:
#     memory: "256Mi"
#   limits:
#     memory: "512Mi"
```

## Hướng dẫn chọn memory limit hợp lý
1. Xem peak memory trong 7 ngày qua trên Grafana
2. Set limit = peak × 1.5
3. Set request = peak × 0.8
4. Monitor 24h sau khi tăng

## Memory leak — fix lâu dài
```bash
# Xem memory tăng dần theo thời gian không
# Nếu tăng liên tục → có memory leak trong code
# Cần dev team điều tra heap dump
```

## Verify
```bash
kubectl get pods -n <namespace> -w
# Chờ Running, không OOMKilled nữa

kubectl top pod <pod-name> -n <namespace>
# Memory dưới limit
```
