# RB-02: Mitigating Canary Rollout Failures (Prometheus Connection Timeout)

**Document ID:** RB-02-ROLLOUT-CANARY-FAIL  
**Created:** 2026-06-19  
**Last Updated:** 2026-06-19  
**Severity:** Critical  
**On-Call Team:** Platform Engineering + SRE

---

## 📋 Overview

Hướng dẫn cách xử lý tình huống khẩn cấp khi tiến trình Canary Deployment của Argo Rollouts bị hủy bỏ (Aborted) do lỗi phân tích chỉ số tự động từ Prometheus.

---

## 🚨 Triệu Chứng (Symptoms)

- `kubectl describe rollout api -n demo` hiển thị thông báo lỗi:
  ```
  Metric "success-rate" assessed Error due to consecutiveErrors exceeded limit
  ```

- Event log chứa:
  ```
  dial tcp: lookup kube-prometheus-stack-prometheus on 10.96.0.10:53: no such host
  ```

- **Tiến trình sinh Pod bị đóng băng hoàn toàn** - không có Pod mới nào được thăng hạng (promote)

- Rollout status báo `Aborted` hoặc kẹt ở bước `SetWeight: 10` lâu hơn 5 phút

- Trên ArgoCD Dashboard:
  - Application báo `Progressing` nhưng không tiến triển
  - Canary pods vẫn ở tỷ lệ 10%, không bao giờ tới 50% hoặc 100%

---

## 🔍 Nguyên Nhân Gốc Rễ (Root Cause)

AnalysisTemplate (tài nguyên cấu hình kiểm tra chất lượng) gọi đến một cụm Prometheus không tồn tại hoặc chưa được cài đặt trong namespace `monitoring` dưới cụm Minikube local.

**Chi tiết kỹ thuật:**
- Argo Rollouts cố gắng query metric `success-rate` từ Prometheus endpoint
- DNS CoreDNS không thể resolve tên service `kube-prometheus-stack-prometheus`
- Network call timeout sau 30 giây
- AnalysisRun chuyển sang trạng thái `ERROR` → Rollout bị abort

---

## 🚑 Các Bước Cứu Vãn Nhanh (Mitigation Steps) - Tình Huống Khẩn Cấp

### Bước 1: Xác định Rollout đang bị kẹt

```bash
# Kiểm tra status hiện tại
kubectl get rollout api -n demo

# Chi tiết lỗi
kubectl describe rollout api -n demo | grep -A 10 "Failed"
```

**Dấu hiệu:** 
- Status column: `Aborted` hoặc kẹt ở `SetWeight: 10`
- Age > 5m (bị kẹt lâu rồi)

### Bước 2: Ép hệ thống thăng hạng thủ công (Manual Promotion)

Loại bỏ các bước Canary bị nghẽn qua patch cấu hình:

```bash
# Cách 1: Xóa hết steps để auto-promote
kubectl patch rollout api -n demo --type merge \
  -p '{
    "spec": {
      "strategy": {
        "canary": {
          "steps": []
        }
      }
    }
  }'

# Cách 2: Skip toàn bộ analysis
kubectl patch rollout api -n demo --type merge \
  -p '{
    "spec": {
      "strategy": {
        "canary": {
          "skipAnalysis": true
        }
      }
    }
  }'
```

### Bước 3: Xác nhận Pod mới đã ngoi lên trạng thái Running

```bash
# Theo dõi deployment real-time
kubectl get pod -n demo -w

# Chờ đến khi tất cả pod:
#   NAME                   READY   STATUS    RESTARTS
#   api-xxxxx              1/1     Running   0
#   api-yyyyy              1/1     Running   0
#   api-zzzzz              1/1     Running   0
#   api-wwwww              1/1     Running   0
```

**Timeout:** Chờ tối đa 10 phút. Nếu vẫn kẹt → chuyển qua bước 4.

### Bước 4: Nuclear Option - Reset Rollout Hoàn Toàn

```bash
# Xóa và tạo lại Rollout (chỉ khi thật sự cần thiết)
kubectl delete rollout api -n demo --ignore-not-found

# Chờ 30 giây
sleep 30

# Áp dụng manifest gốc
kubectl apply -f app-api/rollout.yaml
```

---

## 🛠️ Biện Pháp Lâu Dài (Long-term Fix)

### Tùy Chọn A: Khôi Phục Prometheus Stack (Recommended)

```bash
# Kiểm tra xem Prometheus đã cài đặt chưa
kubectl get pod -n monitoring | grep prometheus

# Nếu không có, liên hệ Infra team để cài đặt
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --create-namespace \
  --values prometheus-values.yaml
```

### Tùy Chọn B: Sửa AnalysisTemplate để Trỏ Đúng Endpoint

```bash
# Kiểm tra endpoint hiện tại
kubectl get analysistemplate -n demo -o yaml

# Sửa app-analysis/analysis.yaml
# Thay:
#   address: "http://kube-prometheus-stack-prometheus:9090"
# Thành:
#   address: "http://prometheus-service.monitoring.svc.cluster.local:9090"

kubectl apply -f app-analysis/analysis.yaml
```

### Tùy Chọn C: Tắt Analysis Cho Môi Trường Local (Immediate Relief)

```bash
# Sửa app-api/rollout.yaml - bỏ analysis block
# Trước:
# strategy:
#   canary:
#     analysis:
#       templates:
#       - templateName: success-rate
#     steps:
#     - setWeight: 10
#     - pause: {duration: 2m}
#     ...

# Sau:
# strategy:
#   canary:
#     steps:
#     - setWeight: 10
#     - pause: {duration: 2m}
#     ...

kubectl apply -f app-api/rollout.yaml
```

---

## 🎯 Nghiệm Thu (Verification)

Xác nhận sự cố đã được giải quyết:

```bash
# 1. Kiểm tra Rollout status
kubectl get rollout api -n demo
# Expected: Desired=4, Current=4, Updated=4, Ready=4, Synced=100%

# 2. Kiểm tra AnalysisRun status
kubectl get analysisrun -n demo
# Expected: Không có AnalysisRun nào ở trạng thái Error

# 3. Kiểm tra Pod đầy đủ
kubectl get pod -n demo | grep api
# Expected: Tất cả pod Running

# 4. Kiểm tra Rollout không còn Error
kubectl describe rollout api -n demo | tail -30
# Expected: Không thấy "ConsecutiveErrors", "Metric assessment failed"

# 5. Kiểm tra ArgoCD sync lại
kubectl get applications -n argocd
# Expected: api app = Healthy & Synced
```

---

## 📞 Escalation & Contact

| Tình Huống | Liên Hệ | SLA |
|-----------|--------|-----|
| Pods vẫn không tới Running sau 10 phút | @Platform-On-Call | 5 min |
| Prometheus vẫn không thể resolve | @SRE-Infra-Team | 15 min |
| Rollout bị lỗi thêm lần nữa trong 1 giờ | @Engineering-Lead | 30 min |
| Cần rollback phiên bản cũ | @DevOps-Lead | Immediate |

---

## 🔔 Prevention Checklist

- [ ] Kiểm tra Prometheus stack trước khi deploy bất kỳ rollout nào
- [ ] Validate AnalysisTemplate endpoint bằng curl từ trong cluster:
  ```bash
  kubectl run -it debug --image=curlimages/curl --restart=Never -- \
    curl http://prometheus-service.monitoring.svc.cluster.local:9090/api/v1/query
  ```
- [ ] Thêm health check cho AnalysisTemplate trong pipeline CI
- [ ] Tạo alert nếu AnalysisRun lỗi liên tiếp > 2 lần
- [ ] Tài liệu hoá đối phó "xóa steps" trong runbook

---

## 📝 Notes

- **TTL cho AnalysisRun:** Default 30 phút, có thể điều chỉnh
- **Consecutive Errors Limit:** Default là 3, kiểm tra bằng `kubectl get analysistemplate -o yaml`
- **Debug Query:** Chạy metrics query trực tiếp:
  ```bash
  kubectl exec -it <prometheus-pod> -n monitoring -- \
    curl 'http://localhost:9090/api/v1/query?query=rate(http_requests_total[1m])'
  ```

