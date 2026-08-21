# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

| | |
|---|---|
| Họ và tên | Nguyễn Hoàng Đạt |
| MSSV | 2A202601460 |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/Noxxsanl/K4-Track2-Day21-2A202601460-NguyenHoangDat |
| Ngày nộp | 21/08/2026 |

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---|---|---|---|---|
| 1 | 100 | 0.1 | 3 | 0.7109 | 0.8780 |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.8460 |
| 3 | 200 | 0.1 | 5 | **0.7149** | 0.8740 |
| 4 | 300 | 0.05 | 4 | 0.7070 | 0.8740 |

**Bộ siêu tham số đã chọn:** `n_estimators=200`, `learning_rate=0.1`, `max_depth=5`.

**Lý do:** Bộ này đạt `f1_score` cao nhất (0.7149), tức bắt được nhiều trường hợp thu nhập cao
nhất. Đáng chú ý, lần có accuracy cao nhất là lần 1 (0.8780) chứ **không** phải lần có F1 cao
nhất: chọn mô hình theo accuracy sẽ chọn nhầm. Về đánh đổi, lần 4 hạ `learning_rate` xuống 0.05
và bù bằng 300 cây nhưng F1 vẫn thấp hơn lần 3 — tăng số cây không bù đủ cho learning_rate thấp.

---

## 2. Vì Sao Ngưỡng Chất Lượng Đặt Trên F1 Chứ Không Phải Accuracy

Tập Adult chỉ có 24,8% mẫu thuộc lớp thu nhập cao. Vì vậy một mô hình vô dụng luôn trả lời
"thu nhập thấp" đã đạt accuracy 0,752 mà `f1_score` bằng 0 — nó không bắt được một trường hợp
thu nhập cao nào. Nếu quality gate đặt trên accuracy, mô hình đó sẽ đi thẳng ra sản phẩm.

Số liệu của tôi cho thấy rõ: qua bốn lần chạy, accuracy chỉ dao động 0,846–0,878 (biên độ
0,032) trong khi f1_score dao động 0,605–0,715 (biên độ 0,110, gấp hơn ba lần). Accuracy gần
như đứng yên nên không phân biệt được mô hình tốt và kém; F1 của lớp dương thì có, vì nó là
trung bình điều hòa của precision và recall **tính riêng cho lớp thiểu số**. Bằng chứng cuối:
lần chạy #2 có accuracy 0,846 nhưng bị Quality Gate chặn vì F1 chỉ 0,6051.

Không dùng `average="weighted"` hay `average="macro"` vì cả hai đều gộp thêm lớp đa số vào kết
quả, kéo giá trị lên cao và làm ngưỡng 0.65 mất ý nghĩa.

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| `ModuleNotFoundError: pkg_resources` khi chạy train.py | setuptools 84 đã gỡ bỏ `pkg_resources` mà mlflow 2.13 vẫn dùng | Ghim `setuptools<81` vào `requirements.txt` để CI không lặp lại lỗi |
| `AccessDenied` khi tạo S3 bucket và khi lấy AMI qua SSM | IAM user chưa được cấp quyền S3/EC2, và không có quyền `ssm:GetParameters` | Gắn `AmazonS3FullAccess` + `AmazonEC2FullAccess`, thay lệnh SSM bằng `aws ec2 describe-images` |
| Lo model không load được trên VM | Phiên bản scikit-learn trên VM có thể khác phiên bản lúc huấn luyện | Cài đúng `scikit-learn==1.4.2` trên EC2 để khớp định dạng joblib |

---

## 4. So Sánh Bước 2 và Bước 3

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1`, 22.361 mẫu) | 0.7149 | 0.8740 |
| Bước 3 (thêm `train_batch2`, 44.722 mẫu) | 0.7354 | 0.8820 |

**Nhận xét:** Gấp đôi dữ liệu chỉ giúp F1 tăng 0,0205. Mức tăng nhỏ này đúng như dự đoán vì
hai batch chia ngẫu nhiên từ cùng một nguồn nên cùng phân phối — dữ liệu mới không mang thêm
thông tin mà mô hình chưa học được. Điều được kiểm chứng ở Bước 3 không phải chỉ số cao hơn,
mà là quy trình: một commit dữ liệu duy nhất chạy hết bốn job và triển khai lên VM, không có
bước thủ công nào.

---

## 5. Phần Bonus Đã Thực Hiện

- [x] Kiểm chứng Quality Gate: đẩy bộ tham số yếu (50 / 0.05 / 2), Quality Gate chặn với
      `f1_score 0.6051 < 0.65` và job Release không chạy — xem `06-quality-gate-fail.png`
      và `06b-quality-gate-log.png`.
