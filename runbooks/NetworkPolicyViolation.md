# NetworkPolicyViolation

## Mô tả
Traffic không được phép giữa pods/namespaces — có thể là lateral movement, data exfiltration, hoặc misconfiguration.

## Dấu hiệu nhận biết
- Alert: `UnexpectedNetworkTraffic` hoặc `NetworkPolicyDenied`
- Pod kết nối đến service không thuộc scope
- Traffic từ namespace không được phép
- Port scanning trong cluster

## Điều tra

### Bước 1 — Kiểm tra NetworkPolicy hiện tại
```bash
# Xem tất cả NetworkPolicy
kubectl get networkpolicies -A

# Chi tiết policy của namespace
kubectl describe networkpolicy -n <namespace>

# Namespace nào KHÔNG có NetworkPolicy (mặc định allow all)
kubectl get namespaces -o json | jq -r '.items[].metadata.name' | while read ns; do
  count=$(kubectl get networkpolicies -n $ns 2>/dev/null | wc -l)
  if [ $count -le 1 ]; then
    echo "⚠️  $ns: NO NetworkPolicy"
  fi
done
```

### Bước 2 — Xem kết nối mạng trong pod nghi ngờ
```bash
# Xem connections hiện tại
kubectl exec -it <pod> -n <namespace> -- netstat -an 2>/dev/null \
  || kubectl exec -it <pod> -n <namespace> -- ss -an

# Xem DNS queries
kubectl exec -it <pod> -n <namespace> -- cat /etc/resolv.conf

# Test kết nối đến pod khác
kubectl exec -it <pod> -n <namespace> -- \
  curl -v http://<other-service>.<other-namespace>.svc.cluster.local
```

### Bước 3 — Kiểm tra CNI logs (nếu dùng Calico/Cilium)
```bash
# Calico
kubectl logs -n kube-system -l k8s-app=calico-node | grep -i "denied\|drop"

# Cilium
kubectl exec -n kube-system ds/cilium -- \
  cilium monitor --type drop 2>/dev/null | head -50
```

## Hành động khắc phục

### Áp dụng NetworkPolicy deny-all ngay
```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: <namespace>
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
```

### Mở chỉ những traffic cần thiết
```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-traffic
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: database
    ports:
    - protocol: TCP
      port: 5432
  # Cho phép DNS
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
EOF
```

### Isolate pod nghi ngờ ngay lập tức
```bash
# Thêm label để NetworkPolicy deny
kubectl label pod <pod-name> -n <namespace> quarantine=true

# Tạo NetworkPolicy deny tất cả cho pod bị quarantine
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: quarantine
  namespace: <namespace>
spec:
  podSelector:
    matchLabels:
      quarantine: "true"
  policyTypes:
  - Ingress
  - Egress
EOF
```

## Kiến trúc NetworkPolicy tốt
```
Internet → Ingress Controller (namespace: ingress)
    ↓ (chỉ port 80/443)
Frontend pods (namespace: frontend)
    ↓ (chỉ port 8080)
API pods (namespace: backend)
    ↓ (chỉ port 5432)
Database (namespace: database)
```

## Prometheus queries
```promql
# Số packets bị drop bởi NetworkPolicy (Calico)
rate(felix_denied_packets_total[5m])

# Network I/O bất thường
rate(container_network_transmit_bytes_total[5m]) > 100000000  # > 100MB/s
```
