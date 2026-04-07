# SRE Runbooks

Bộ runbook xử lý incident cho Kubernetes cluster.

## Danh sách runbooks

| Runbook | Mô tả | Severity |
|---------|-------|----------|
| [CrashLoopBackOff](runbooks/CrashLoopBackOff.md) | Pod crash liên tục | Critical |
| [OOMKilled](runbooks/OOMKilled.md) | Pod bị kill do hết memory | Critical |
| [NodeNotReady](runbooks/NodeNotReady.md) | Node mất kết nối cluster | Critical |
| [DiskPressure](runbooks/DiskPressure.md) | Disk đầy trên node | Warning |
| [ImagePullBackOff](runbooks/ImagePullBackOff.md) | Không pull được image | Warning |
| [PodPending](runbooks/PodPending.md) | Pod không schedule được | Warning |
| [HighCPU](runbooks/HighCPU.md) | CPU usage vượt ngưỡng | Warning |

## Cấu trúc mỗi runbook

1. **Mô tả** — Incident là gì
2. **Điều tra** — Các lệnh để tìm root cause
3. **Fix** — Cách xử lý từng nguyên nhân
4. **Verify** — Kiểm tra sau khi fix
5. **Prometheus queries** — Query để monitor

## Thêm runbook mới

Tạo file `runbooks/<TênAlert>.md` theo template trên.
Đặt tên file trùng với `alertname` trong Prometheus Alert Rule để SRE Agent tìm được chính xác.
