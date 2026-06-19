# RB-01: Troubleshooting ArgoCD App Degraded & InvalidImageName

**Document ID:** RB-01-API-TROUBLESHOOTING  
**Created:** 2026-06-19  
**Last Updated:** 2026-06-19  
**Severity:** High  
**On-Call Team:** Platform Engineering

---

## 📋 Overview

Hướng dẫn khắc phục nhanh sự cố khi ứng dụng API bị kẹt trạng thái Degraded hoặc hiện lỗi cấu hình tên Image trên cụm Minikube.

---

## 🚨 Triệu Chứng (Symptoms)

- Lệnh `kubectl get pod -n demo` trả về trạng thái **InvalidImageName**
- Trên ArgoCD Dashboard, ứng dụng báo **Degraded** (màu đỏ hoặc cam)
- Event trong Pod cho thấy: `Failed to pull image "ghcr.io/Ninhnguyen1003/w10-api:0.0.4"`
- ArgoCD không thể sync ứng dụng, sync status = OutOfSync

---

## 🔍 Nguyên Nhân Gốc Rễ (Root Cause)

Kubernetes bắt buộc các đường dẫn Image Registry (GHCR, DockerHub, ECR, v.v.) phải viết thường hoàn toàn (**lowercase**).

Việc đặt tên User GitHub có chữ viết hoa (ví dụ: `Ninhnguyen1003`) trong manifest sẽ gây lỗi vì:
- Container runtime (Containerd/Docker) không thể resolve tên image với chữ hoa
- DNS resolution sinh ra lỗi case-sensitive
- Kubernetes kubelet từ chối pull image

---

## ✅ Các Bước Xử Lý Từng Bước (Resolution Steps)

### Bước 1: Định vị tệp manifest có lỗi
```bash
# Kiểm tra image hiện tại trong Rollout
kubectl get rollout api -n demo -o yaml | grep image
```

**Kết quả dự kiến:** 
```
image: ghcr.io/Ninhnguyen1003/w10-api:0.0.4   # ❌ WRONG (chữ hoa)
```

### Bước 2: Sửa đổi file rollout.yaml trên local

Mở `app-api/rollout.yaml` và sửa dòng image:

**Trước (WRONG):**
```yaml
- name: api
  image: ghcr.io/Ninhnguyen1003/w10-api:0.0.4  # ❌ N viết hoa
```

**Sau (CORRECT):**
```yaml
- name: api
  image: ghcr.io/ninhnguyen1003/w10-api:0.0.4  # ✅ n viết thường
```

### Bước 3: Push lên GitHub

```bash
cd /path/to/repo
git add app-api/rollout.yaml
git commit -m "fix: correct image registry name to lowercase"
git push origin main
```

### Bước 4: Ép ArgoCD xóa cache và đồng bộ lại

```bash
# Ép ArgoCD refresh cache
kubectl annotate application api -n argocd \
  argocd.argoproj.io/refresh=hard \
  --overwrite

# Ép sync ngay lập tức
kubectl patch application api -n argocd \
  --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/compare-result":"Unknown"}}}'
```

### Bước 5: Kiểm tra trạng thái pod

```bash
# Đợi 30-60 giây rồi kiểm tra
kubectl get pod -n demo -w
```

**Trạng thái mong đợi:** Pods chuyển từ `ImagePullBackOff` → `Pending` → `Running`

---

## 🎯 Nghiệm Thu (Verification)

Chạy lần lượt các lệnh dưới để xác nhận sự cố đã được khắc phục:

```bash
# 1. Kiểm tra ArgoCD Application Status
kubectl get applications -n argocd
# Kết quả: api app phải có STATUS = Healthy, SYNC = Synced

# 2. Kiểm tra Rollout Status
kubectl get rollout api -n demo
# Kết quả: Desired=4, Current=4, Updated=4, Ready=4

# 3. Kiểm tra Pod Details
kubectl get pod -n demo
# Kết quả: Tất cả pod phải có STATUS = Running

# 4. Kiểm tra Event Log
kubectl describe rollout api -n demo | tail -20
# Kết quả: Không có lỗi Image Pull, chỉ thấy "Deployment created"

# 5. Kiểm tra Image trên Pod
kubectl get pod api-xxxxx -n demo -o jsonpath='{.spec.containers[0].image}'
# Kết quả: ghcr.io/ninhnguyen1003/w10-api:0.0.4  ✅ lowercase
```

---

## 📞 Escalation & Liên Hệ

| Tình Huống | Liên Hệ |
|-----------|---------|
| Sau 5 phút vẫn còn ImagePullBackOff | Kiểm tra Docker credentials trong imagePullSecrets |
| ArgoCD vẫn Degraded | Xóa finalizers: `kubectl patch application api -n argocd -p '{"metadata":{"finalizers":[]}}' --type=merge` |
| Pod không pull được image từ GHCR | Kiểm tra GitHub Token expiration, xin new token từ @DevOps team |

---

## 📝 Notes

- **Quy tắc chung:** Tất cả image URL trong Kubernetes phải ở dạng **lowercase**
- **Prevention:** Thêm validation hook vào CI/CD pipeline để kiểm tra case của image name
- **Automation:** Xem xét dùng Kyverno hoặc OPA để enforce policy này

