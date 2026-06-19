# ADR-001: Bypass Automated Metric Analysis in Local Minikube Environment

**Date:** 2026-06-19  
**Status:** ✅ Approved  
**Authors:** Platform Engineering Team  
**Decision Record ID:** ADR-001

---

## 📋 Context

### Bối Cảnh Dự Án

Dự án XBrain Internship Lab 2.2 triển khai chiến lược **Canary Deployment** sử dụng:
- **Argo Rollouts** cho progressive delivery
- **ArgoCD** cho GitOps reconciliation
- **Prometheus** cho automated metrics analysis (success-rate)
- **Minikube** làm môi trường development/testing local

### Thách Thức Kỹ Thuật

Chiến lược Canary Deployment yêu cầu một hệ thống giám sát hoạt động **(Prometheus)** để liên tục cào dữ liệu (success-rate) kiểm tra lỗi. Tuy nhiên:

1. **Giới hạn Tài nguyên Hardware:**
   - Minikube chạy trên máy local (RAM: 8-16GB, CPU: 2-4 cores)
   - Bộ Helm Stack Prometheus (~2-3GB RAM) cồng kềnh, thường xuyên gây sập cụm

2. **Lỗi DNS CoreDNS:**
   - AnalysisTemplate không thể resolve tên service `kube-prometheus-stack-prometheus`
   - Dẫn đến timeout, AnalysisRun lỗi consecutively

3. **Tác Động Deployment:**
   - Canary tiến trình bị nghẽn ở bước `SetWeight: 10`
   - ArgoCD Application kẹt ở trạng thái `Degraded`
   - Chu kỳ CI/CD bị delay 10-15 phút mỗi lần

4. **Overhead Vận Hành:**
   - Kỹ sư phải debug lỗi DNS, metrics endpoint liên tục
   - Không có giá trị thực tế cho testing vì metrics không chính xác trên local (synthetic load)

---

## 🎯 Decision

### Quyết Định Chính

Chúng ta quyết định **loại bỏ phụ thuộc Prometheus Metrics** cho **môi trường Minikube local**, thay thế bằng:

1. **Bỏ qua AnalysisTemplate** trong Rollout config cho local env
   - Đặt `skipAnalysis: true` hoặc xóa `steps` để instant promotion

2. **Thay thế bằng Health Checks cơ bản:**
   - Liveness Probe: `/healthz` → HTTP 200
   - Readiness Probe: `/healthz` → HTTP 200
   - Pod đạt trạng thái `Running` = deployment success

3. **Kích hoạt lại Prometheus trên Staging/Production:**
   - AWS environment sẽ có Prometheus stack đầy đủ
   - AnalysisTemplate sẽ hoạt động bình thường trên Staging/Prod

### Tóm Tắt Quyết Định

| Yếu Tố | Local (Minikube) | Staging/Prod (AWS) |
|--------|------------------|-------------------|
| **Analysis** | ❌ Disabled | ✅ Enabled |
| **Metrics Query** | ❌ N/A | ✅ Prometheus |
| **Canary Steps** | ⚡ Instant (0→100%) | 📊 Gradual (10→50→100%) |
| **Validation** | Liveness/Readiness Probe | Automated Metrics + Manual |

---

## ✅ Consequences

### Hậu Quả Tích Cực (Advantages)

1. **Giảm Tải Tài Nguyên:**
   - Tiết kiệm 2-3GB RAM trên máy local
   - Minikube ổn định, không bị sập/restart tự động
   - CPU usage giảm 40-50%

2. **Tăng Tốc Độ Deployment:**
   - Chu kỳ CI/CD nhanh hơn **5x**
   - Canary deployment từ 10-15 phút → 2-3 phút
   - Kỹ sư có thể test nhanh hơn

3. **Giảm Complexity Vận Hành:**
   - Không cần debug DNS CoreDNS issues
   - Không cần kiểm tra Prometheus endpoint availability
   - Kịch bản troubleshooting đơn giản hơn

4. **Thích Hợp Cho Development:**
   - Local testing không cần metrics analysis (synthetic data không có giá trị)
   - Phù hợp với agile development cycles

### Hậu Quả Tiêu Cực (Trade-offs)

1. **Mất Tự động Hóa Kiểm Tra Chất Lượng:**
   - Kỹ sư Ops phải kiểm tra lỗi ứng dụng **bằng tay** qua:
     - `kubectl logs api-xxxxx -n demo`
     - `kubectl exec` vào pod debug
     - Prometheus queries thủ công

2. **Nguy Hiểm Metrics False Negatives:**
   - Có thể deploy phiên bản có bug lên Staging vì không có tự động rollback
   - Cần kiểm tra thủ công hoặc smoke test trước khi merge

3. **Khác Biệt Environment:**
   - Local behavior ≠ Production behavior
   - Kỹ sư phải quen với 2 deployment strategies khác nhau

---

## 🔄 Implementation Plan

### Giai Đoạn 1: Local Environment (Hiện Tại)

**File thay đổi:** `app-api/rollout.yaml`

```yaml
# Trước (with analysis)
strategy:
  canary:
    analysis:
      templates:
      - templateName: success-rate
    steps:
    - setWeight: 10
    - pause: {duration: 2m}
    - setWeight: 50
    - pause: {duration: 2m}
    - setWeight: 100

# Sau (without analysis - local)
strategy:
  canary:
    skipAnalysis: true  # ← Thêm dòng này
    steps:
    - setWeight: 100    # ← Instant promotion
```

**Hoặc phương pháp khác - xóa steps:**

```yaml
strategy:
  canary:
    steps: []  # ← Empty steps = instant promotion to 100%
```

### Giai Đoạn 2: Staging/Production (Tương Lai)

Khi triển khai lên AWS:

```yaml
# app-api/rollout-prod.yaml
strategy:
  canary:
    analysis:
      templates:
      - templateName: success-rate
    steps:
    - setWeight: 10
    - pause: {duration: 5m}
    - setWeight: 50
    - pause: {duration: 5m}
    - setWeight: 100
```

---

## 📊 Alternatives Considered

### ❌ Alternative 1: Dùng Prometheus Stack Stack nhỏ hơn
- **Rejection Reason:** Prometheus stack nhỏ nhất vẫn cần 1.5GB+, còn DNS issues vẫn xảy ra
- **Cost:** Cao, complexity vẫn ở mức cao

### ❌ Alternative 2: Tắt Minikube hoàn toàn, dùng Docker Compose
- **Rejection Reason:** Không phù hợp test Kubernetes objects (Rollout, Application)
- **Cost:** Mất platform validation

### ✅ Alternative 3: Bypass Analysis cho Local (SELECTED)
- **Acceptance Reason:** Simple, scalable, no overhead
- **Cost:** Thấp, chỉ cần update 1 file manifest

### ❓ Alternative 4: Dùng Mock Prometheus Endpoint
- **Status:** Future consideration
- **Feasibility:** Có thể implement nếu cần sophisticated testing later

---

## 📝 Implementation Checklist

- [ ] Update `app-api/rollout.yaml` với `skipAnalysis: true`
- [ ] Test deployment local: `kubectl apply -f app-api/rollout.yaml`
- [ ] Verify pods tới Running trong < 2 phút
- [ ] Update CI/CD pipeline để bỏ Prometheus dependency cho local tests
- [ ] Document trong README: "Local uses instant promotion, Prod uses canary with metrics"
- [ ] Tạo separate config file cho Staging/Prod (future)
- [ ] Train team về 2 deployment strategies

---

## 🔐 Sigstore/Cosign Integration (Unchanged)

**Lưu ý quan trọng:** Decision này **KHÔNG ảnh hưởng** đến:
- Verify image signature với Cosign (`cosign.pub` key)
- Policy constraints từ Gatekeeper
- RBAC rules

Các biện pháp bảo mật này vẫn được áp dụng bình thường trên cả local & production.

---

## 📅 Review & Revision

| Date | Status | Reviewer | Notes |
|------|--------|----------|-------|
| 2026-06-19 | ✅ Approved | @Platform-Lead | Initial decision for Lab 2.2 |
| TBD | ⏳ Review | @SRE-Team | Reassess after AWS deployment |

---

## 🔗 Related Documents

- [RB-01: Troubleshooting ArgoCD App Degraded](./RB-01-API-TROUBLESHOOTING.md)
- [RB-02: Mitigating Canary Rollout Failures](./RB-02-ROLLOUT-CANARY-FAIL.md)
- [Analysis Template Documentation](../app-analysis/analysis.yaml)
- [Argo Rollouts Official Docs](https://argoproj.github.io/rollouts/)

---

## 📞 Questions & Clarifications

**Q: Làm sao biết phiên bản mới có bug nếu không có Prometheus metrics?**  
A: Dùng Liveness/Readiness Probe + manual logs inspection. Trên Staging, Prometheus sẽ bắt được.

**Q: Có thể revert decision này không?**  
A: Có. Chỉ cần bỏ `skipAnalysis: true` khi Prometheus stack sẵn sàng.

**Q: Khi nào chúng ta nên migrate tới Production strategy?**  
A: Khi deploy lên AWS Staging environment (target: Week 3 of internship).

