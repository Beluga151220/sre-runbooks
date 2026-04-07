# PodPending

## Mô tả
Pod mắc kẹt ở trạng thái Pending — chưa được schedule lên node nào.

## Nguyên nhân phổ biến
1. Không đủ CPU/Memory trên bất kỳ node nào
2. Node selector / affinity không match
3. Taint trên node không có toleration tương ứng
4. PVC chưa được bound
5. Cluster autoscaler chưa scale up kịp

## Điều tra

### Bước 1 — Xem lý do Pending
```bash
kubectl describe pod <pod-name> -n <namespace>
# Tìm phần Events — thường thấy:
# "0/3 nodes are available: 3 Insufficient cpu"
# "0/3 nodes are available: 3 Insufficient memory"
# "0/3 nodes are available: node(s) had taint..."
```

### Bước 2 — Xem resource available trên nodes
```bash
kubectl describe nodes | grep -A 5 "Allocated resources"
kubectl top nodes
```

### Bước 3 — Xem resource request của pod
```bash
kubectl get pod <pod-name> -n <namespace> \
  -o jsonpath='{.spec.containers[*].resources}'
```

## Fix

### Insufficient CPU/Memory — giảm resource request
```bash
kubectl edit deployment <deployment-name> -n <namespace>
# Sửa resources.requests xuống hợp lý:
# resources:
#   requests:
#     cpu: "100m"      # thay vì "4"
#     memory: "256Mi"  # thay vì "64Gi"
```

### Node taint — thêm toleration
```bash
# Xem taint trên node
kubectl describe node <node-name> | grep Taint

# Thêm toleration vào pod spec
kubectl edit deployment <n> -n <namespace>
# spec.template.spec.tolerations:
# - key: "key"
#   operator: "Equal"
#   value: "value"
#   effect: "NoSchedule"
```

### PVC Pending — kiểm tra storage
```bash
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>

# Kiểm tra StorageClass
kubectl get storageclass
```

### Cluster đầy — cần thêm node
```bash
# Trên VNG Cloud: vào VKS portal → Node Group → Scale up
# Hoặc tăng max nodes của autoscaler
```

## Prometheus queries
```promql
# Pods đang Pending
kube_pod_status_phase{phase="Pending"}

# CPU allocatable còn lại
kube_node_status_allocatable{resource="cpu"}
  - kube_node_resource_requests{resource="cpu"}

# Memory allocatable còn lại  
kube_node_status_allocatable{resource="memory"}
  - kube_node_resource_requests{resource="memory"}
```

## Verify
```bash
kubectl get pods -n <namespace> -w
# Pod chuyển Pending → ContainerCreating → Running
```
