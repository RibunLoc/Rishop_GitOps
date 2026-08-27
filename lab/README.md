# SOPS Lab — học ở local, không đụng Argo CD

Mục tiêu: hiểu SOPS mã hoá cái gì và giải mã ra sao, bằng file giả.
Production của repo này vẫn dùng ESO — lab này hoàn toàn tách biệt.

## Cài sops

```bash
winget install Mozilla.SOPS
sops --version
```

Cần AWS credential có quyền `kms:Encrypt` / `kms:Decrypt` trên key
`arn:aws:kms:ap-southeast-1:758497006160:key/1b177727-62ca-4e21-9b9f-9e2b7061c91c`
(key bạn đã dùng với `aws-encryption-cli`). Kiểm tra:

```bash
aws sts get-caller-identity
```

## Bài 1 — Mã hoá

```bash
cd d:/Document/RiShop/GitOps
cp lab/demo-secret.yaml lab/demo-secret.enc.yaml
sops --encrypt --in-place lab/demo-secret.enc.yaml
```

Mở `lab/demo-secret.enc.yaml` ra xem. Điều cần quan sát:

- `apiVersion`, `kind`, `metadata.name` — **vẫn đọc được**
- Ba giá trị dưới `stringData` — thành `ENC[AES256_GCM,data:...]`
- Cuối file mọc thêm khối `sops:` chứa `arn` và `enc`

Khối `enc` đó chính là **data key đã được KMS bọc lại** — đúng cơ chế
envelope encryption bạn đã thấy trong `metadata.json` của aws-encryption-cli.

## Bài 2 — Giải mã

```bash
# In ra màn hình, không sửa file
sops --decrypt lab/demo-secret.enc.yaml
```

Chạy được nghĩa là sops đã gọi `kms:Decrypt` thành công để mở data key.

## Bài 3 — Sửa file đã mã hoá

```bash
sops lab/demo-secret.enc.yaml
```

Lệnh này: giải mã ra file tạm → mở editor → bạn sửa → lưu → tự mã lại.
Bạn không bao giờ phải tự tay giải mã rồi mã lại.

Đổi `DB_PASSWORD` thành giá trị khác, lưu, rồi so sánh:

```bash
git diff lab/demo-secret.enc.yaml
```

**Đây là điểm mạnh nhất của SOPS**: chỉ dòng `DB_PASSWORD` đổi, hai dòng kia
giữ nguyên ciphertext. Bạn review được *cái gì vừa đổi* mà không thấy giá trị.
(`aws-encryption-cli` mã cả file thành khối nhị phân → diff vô nghĩa.)

## Bài 4 — Deploy thủ công

```bash
sops --decrypt lab/demo-secret.enc.yaml | kubectl apply -f -
kubectl get secret sops-lab-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

Đây là cách dùng SOPS **không cần KSOPS**: giải mã ở máy bạn rồi pipe vào kubectl.
Đủ dùng cho thao tác tay, nhưng không phải GitOps — Argo CD không làm được bước này.

Dọn dẹp:
```bash
kubectl delete secret sops-lab-secret
```

## Bài 5 — Thử phá

Sửa tay một ký tự trong chuỗi `ENC[...]` rồi giải mã lại:

```bash
sops --decrypt lab/demo-secret.enc.yaml
```

Sẽ báo `MAC mismatch`. Vì AES-GCM có xác thực toàn vẹn — sửa một bit là
hỏng cả file, không giải mã được nữa. Cùng lý do với lỗi encryption context
không khớp mà bạn gặp với aws-encryption-cli.

Khôi phục: `git checkout lab/demo-secret.enc.yaml` (nếu đã commit).

## Vì sao lab này chưa dùng cho production

Argo CD **không hiểu SOPS**. `repo-server` clone repo về, thấy `ENC[...]`
và apply nguyên xi — pod nhận mật khẩu là chuỗi rác.

Muốn dùng thật phải cài **KSOPS** vào `repo-server`: thêm initContainer,
mount binary, cấp `kms:Decrypt` cho pod đó. Setup sai → repo-server crash →
**mọi** app ngừng sync, không riêng rishop.

Nên: production dùng ESO. SOPS để đây học và dùng cho thao tác tay.
