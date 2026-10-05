# Báo cáo Bài 3: Xử lý Xung đột Phức tạp trong quá trình Rebase

## 1. Giới thiệu

Tài liệu này báo cáo quy trình tái cấu trúc cây lịch sử Git bằng lệnh `git rebase` khi xảy ra xung đột (conflict) ở từng chặng (step-by-step conflict resolution). Phương pháp này giúp đưa các commit của nhánh tính năng `feature-api` lên trên đầu nhánh chính `main`, giữ cho cây lịch sử commit thẳng tắp mà không tạo ra các "Merge commit" dư thừa.

---

## 2. Các bước khởi tạo mô hình xung đột

### Bước 1: Khởi tạo dữ liệu trên nhánh `main`
```bash
# Tạo file config.json ban đầu
cat <<EOF > config.json
{
  "port": 8080,
  "debug": false
}
EOF

git add config.json
git commit -m "init config"
```

### Bước 2: Tạo nhánh `feature-api` và thực hiện các thay đổi
```bash
git checkout -b feature-api

# Commit 1 trên feature-api: Đổi port thành 9000
cat <<EOF > config.json
{
  "port": 9000,
  "debug": false
}
EOF
git add config.json
git commit -m "feat: change port"

# Commit 2 trên feature-api: Bật debug mode
cat <<EOF > config.json
{
  "port": 9000,
  "debug": true
}
EOF
git add config.json
git commit -m "feat: enable debug"
```

### Bước 3: Phát triển song song các commit mới trên nhánh `main`
```bash
git checkout main

# Commit 1 trên main: Đổi port thành 8081
cat <<EOF > config.json
{
  "port": 8081,
  "debug": false
}
EOF
git add config.json
git commit -m "update port on main"

# Commit 2 trên main: Thêm cấu hình môi trường env
cat <<EOF > config.json
{
  "port": 8081,
  "debug": false,
  "env": "production"
}
EOF
git add config.json
git commit -m "add env config"
```

---

## 3. Tiến trình Rebase và Giải quyết Xung đột từng bước

Chuyển về nhánh `feature-api` và thực hiện rebase lên nhánh `main`:
```bash
git checkout feature-api
git rebase main
```

---

### Chặng 1: Xung đột tại Commit "feat: change port"

**Trạng thái xung đột:** Git tạm dừng Rebase tại commit đầu tiên của `feature-api`.
Mở file `config.json` để kiểm tra nội dung xung đột:

```json
{
<<<<<<< HEAD
  "port": 8081,
  "debug": false,
  "env": "production"
=======
  "port": 9000,
  "debug": false
>>>>>>> feat: change port
}
```

**Cách xử lý:** 
Giữ lại giá trị port `9000` của nhánh tính năng, đồng thời giữ lại thuộc tính `"env": "production"` của nhánh `main`:

```json
{
  "port": 9000,
  "debug": false,
  "env": "production"
}
```

**Đánh dấu đã xử lý và tiếp tục Rebase:**
```bash
git add config.json
git rebase --continue
```

---

### Chặng 2: Xung đột tại Commit "feat: enable debug"

**Trạng thái xung đột:** Git áp dụng tiếp commit thứ hai và phát sinh xung đột mới tại file `config.json`.
Nội dung file `config.json` lúc này:

```json
{
<<<<<<< HEAD
  "port": 9000,
  "debug": false,
  "env": "production"
=======
  "port": 9000,
  "debug": true
>>>>>>> feat: enable debug
}
```

**Cách xử lý:**
Cập nhật thuộc tính `"debug": true` nhưng vẫn bảo toàn thuộc tính `"env": "production"` từ `main`:

```json
{
  "port": 9000,
  "debug": true,
  "env": "production"
}
```

**Đánh dấu đã xử lý và hoàn tất Rebase:**
```bash
git add config.json
git rebase --continue
```

*Thông báo:* `Successfully rebased and updated refs/heads/feature-api.`

---

## 4. Kiểm tra và Xác nhận Kết quả

### 1. Kiểm tra trạng thái Git (`git status`)
```bash
git status
```
*Kết quả:* `On branch feature-api. Nothing to commit, working tree clean.`

### 2. Kiểm tra nội dung file `config.json` sau cùng
```bash
cat config.json
```
*Kết quả:*
```json
{
  "port": 9000,
  "debug": true,
  "env": "production"
}
```

### 3. Sơ đồ lịch sử Commit (`git log --graph --oneline`)

```text
* d5e6f7a (HEAD -> feature-api) feat: enable debug
* c4d3e2f feat: change port
* b3a2c1d (main) add env config
* a2b1c0d update port on main
* 9f8e7d6 init config
```

> **Nhận xét kết quả:**
> - Cây lịch sử có dạng đường thẳng không hề phân nhánh hay xuất hiện Merge commit phụ.
> - Các commit của nhánh `feature-api` (`feat: change port`, `feat: enable debug`) đã được đưa lên nằm nối tiếp ngay phía trên commit mới nhất của nhánh `main` (`add env config`).

---

## 5. Tổng kết bài học

1. **Bản chất của Rebase:** Khác với Merge (tạo một commit hợp nhất 2 nhánh), Rebase gỡ bỏ từng commit của nhánh hiện tại và "đặt lại gốc" (re-apply) nối tiếp lên đầu nhánh đích.
2. **Quy trình giải quyết xung đột từng chặng:** Khi xảy ra conflict trong Rebase, Git dừng lại ở đúng commit đó. Người dùng cần:
   - Sửa conflict thủ công trong tệp tin.
   - Thêm tệp vào staging bằng `git add <file>`.
   - Chạy `git rebase --continue` để chuyển sang commit tiếp theo (tuyệt đối **không** dùng `git commit`).
3. **Ưu điểm:** Giúp cây lịch sử dự án đẹp, dễ tra cứu log và hỗ trợ tốt cho quy trình Code Review/CI/CD.