# HighCPU

## Mô tả
CPU usage của pod hoặc node vượt ngưỡng cảnh báo (>80%).

## Điều tra

### Bước 1 — Xác định pod/node nào cao
```bash
kubectl top nodes
kubectl top pods -A --sort-by=cpu | head -20
```

### Bước 2 — Xem chi tiết process bên trong pod
```bash
kubectl exec -it <pod-name> -n <namespace> -- top
kubectl exec -it <pod-name> -n <namespace> -- ps aux --sort=-%cpu | head -20
```

### Bước 3 — Query Prometheus xem trend
```promql
# CPU usage theo pod (cores)
rate(container_cpu_usage_seconds_total{
  pod="<pod>", container!=""
}[5m])

# CPU throttling %
rate(container_cpu_throttled_seconds_total{pod="<pod>"}[5m])
  / rate(container_cpu_usage_seconds_total{pod="<pod>"}[5m]) * 100

# Top pods CPU
topk(10, rate(container_cpu_usage_seconds_total{container!=""}[5m]))
```

## Fix

### Tăng CPU limit tạm thời
```bash
kubectl set resources deployment <n> -n <namespace> \
  --requests=cpu=500m \
  --limits=cpu=2000m
```

### Scale horizontal (khuyến khích hơn)
```bash
# Scale thủ công
kubectl scale deployment <n> -n <namespace> --replicas=3

# Hoặc cấu hình HPA
kubectl autoscale deployment <n> -n <namespace> \
  --cpu-percent=70 \
  --min=2 \
  --max=10
```

### Kiểm tra CPU spike có phải do traffic không
```promql
# So sánh CPU vs request rate
rate(http_requests_total[5m])
```

## Prometheus Alert Rule mẫu
```yaml
- alert: HighCPUUsage
  expr: |
    rate(container_cpu_usage_seconds_total{container!=""}[5m])
      / kube_pod_container_resource_limits{resource="cpu"} > 0.8
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "CPU usage cao: {{ $labels.pod }}"
    description: "Pod {{ $labels.pod }} đang dùng {{ $value | humanizePercentage }} CPU limit"
```
