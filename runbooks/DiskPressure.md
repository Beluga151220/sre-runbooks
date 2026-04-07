# DiskPressure

## Mô tả
Node báo DiskPressure khi disk usage vượt ngưỡng. K8s sẽ evict pods để giải phóng dung lượng.

## Dấu hiệu
- `kubectl describe node` → Conditions: DiskPressure = True
- Pods bị evict với reason: `Evicted`
- Kubelet log: `eviction manager: pods evicted to reclaim ephemeral storage`

## Điều tra

### Bước 1 — Xem disk usage
```bash
# SSH vào node bị DiskPressure
df -h

# Xem thư mục nào chiếm nhiều nhất
du -sh /var/lib/docker/*  2>/dev/null | sort -rh | head -10
du -sh /var/lib/containerd/* 2>/dev/null | sort -rh | head -10
du -sh /var/log/* 2>/dev/null | sort -rh | head -10
```

### Bước 2 — Kiểm tra images cũ
```bash
docker images | sort -k7 -rh | head -20
# hoặc với containerd:
crictl images | sort -k3 -rh | head -20
```

### Bước 3 — Kiểm tra logs lớn
```bash
find /var/log -name "*.log" -size +100M 2>/dev/null
find /var/log/pods -name "*.log" -size +100M 2>/dev/null
```

## Fix

### Dọn container và images không dùng
```bash
# Docker
docker system prune -af --volumes

# Containerd
crictl rmi --prune
```

### Dọn logs hệ thống
```bash
journalctl --vacuum-size=500M
journalctl --vacuum-time=7d
```

### Dọn pod logs cũ
```bash
find /var/log/pods -name "*.log" -mtime +7 -delete
```

### Dọn tmp files
```bash
rm -rf /tmp/* /var/tmp/*
```

### Tăng disk (nếu cần lâu dài)
```bash
# Trên VNG Cloud: vào portal → resize disk của node
# Sau khi resize:
growpart /dev/vda 1
resize2fs /dev/vda1
df -h  # verify
```

## Phòng tránh

### Cấu hình log rotation cho containerd
```bash
cat > /etc/docker/daemon.json << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
EOF
systemctl restart docker
```

### Set eviction threshold trong kubelet
```yaml
# /var/lib/kubelet/config.yaml
evictionHard:
  nodefs.available: "10%"
  nodefs.inodesFree: "5%"
  imagefs.available: "10%"
```

## Prometheus queries
```promql
# Disk available %
(node_filesystem_avail_bytes{mountpoint="/"} 
  / node_filesystem_size_bytes{mountpoint="/"}) * 100

# Disk sẽ đầy sau bao lâu (predict 24h)
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 24*3600)
```

## Verify
```bash
df -h  # Disk dưới 80%
kubectl describe node <node-name> | grep DiskPressure
# Conditions: DiskPressure = False
```
