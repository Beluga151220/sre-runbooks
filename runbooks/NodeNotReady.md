# NodeNotReady

## Mô tả
Node chuyển sang trạng thái NotReady — pods trên node đó không schedule được hoặc bị evict.

## Điều tra

### Bước 1 — Xác định node nào bị
```bash
kubectl get nodes
# Tìm node STATUS = NotReady
```

### Bước 2 — Xem lý do
```bash
kubectl describe node <node-name>
# Tìm phần Conditions:
# - Ready: False/Unknown
# - MemoryPressure: True
# - DiskPressure: True
# - PIDPressure: True
```

### Bước 3 — Xem events trên node
```bash
kubectl get events -A --field-selector involvedObject.name=<node-name>
```

### Bước 4 — SSH vào node kiểm tra
```bash
ssh <node-ip>

# Kiểm tra kubelet
systemctl status kubelet
journalctl -u kubelet -n 100 --no-pager

# Kiểm tra containerd
systemctl status containerd

# Kiểm tra tài nguyên
df -h          # disk
free -h        # memory
top            # CPU
```

## Nguyên nhân và fix

### kubelet bị crash
```bash
systemctl restart kubelet
systemctl enable kubelet
```

### Disk pressure (đĩa đầy)
```bash
df -h
# Dọn dẹp docker images cũ
docker system prune -af

# Dọn logs cũ
journalctl --vacuum-time=3d

# Xem thư mục nào chiếm nhiều nhất
du -sh /* 2>/dev/null | sort -rh | head -20
```

### Memory pressure
```bash
free -h
# Xem process nào ăn nhiều RAM
ps aux --sort=-%mem | head -20

# Nếu swap bị tắt (k8s yêu cầu), restart kubelet
systemctl restart kubelet
```

### Network issue
```bash
# Kiểm tra CNI plugin
ls /etc/cni/net.d/
systemctl status flanneld    # nếu dùng flannel
systemctl status calico-node # nếu dùng calico

# Restart CNI
kubectl delete pod -n kube-system -l k8s-app=calico-node
```

### containerd crash
```bash
systemctl restart containerd
systemctl restart kubelet
```

## Prometheus queries
```promql
# Node condition
kube_node_status_condition{condition="Ready", status="true"}

# Node disk usage
node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100

# Node memory available
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100
```

## Verify
```bash
kubectl get nodes -w
# Chờ STATUS = Ready
```

## Escalate nếu
- Node vẫn NotReady sau 10 phút restart kubelet
- Nhiều nodes cùng NotReady (network issue toàn cluster)
- Hardware failure (disk I/O error trong dmesg)
