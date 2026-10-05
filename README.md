

## 1. BỐI CẢNH & PHÂN TÍCH YÊU CẦU

### 1.1. Bối cảnh thực tế
Trong quy trình phát triển phần mềm, khi lập trình viên làm việc trên nhánh tính năng cục bộ (local feature branch), việc tạo ra các commit thử nghiệm nhỏ lẻ, vụn vặt là điều thường xuyên xảy ra:
- Sửa một lỗi chính tả nhỏ (`fix typo`).
- Thêm một vài hàm tiện ích bổ trợ (`adds utility functions`).
- Thêm tệp tin ghi log / debug tạm thời (`temp.txt`).

Nếu đẩy toàn bộ các commit vụn vặt này lên máy chủ mã nguồn dùng chung (Remote Repository), lịch sử Git sẽ bị "ô nhiễm" (polluted), gây khó khăn cho việc Code Review, theo dõi tiến độ, và truy vết lỗi bằng `git bisect`. Do đó, kỹ thuật **Interactive Rebase** là tiêu chuẩn bắt buộc để làm sạch lịch sử cục bộ trước khi tích hợp code.

### 1.2. Ràng buộc đề bài
1. Gộp 3 commit nhỏ cuối cùng liên quan đến tính năng authentication thành 1 commit duy nhất.
2. Đổi thông điệp commit sau khi gộp thành: `feat: hoan thien module authentication`.
3. Xóa bỏ hoàn toàn commit chứa tệp rác thử nghiệm (`temp.txt`) khỏi lịch sử và thư mục làm việc.
4. Lịch sử commit sau khi hoàn tất chỉ hiển thị duy nhất 1 commit sạch sẽ đại diện cho tính năng.

---

## 2. QUÁ TRÌNH THỰC HIỆN CHI TIẾT

### Bước 1: Khởi tạo mã nguồn và 4 commit mô phỏng ban đầu

Trên nhánh làm việc hiện tại (`main`), lần lượt tạo 4 commit:

```bash
# 1. Commit 1: Tạo auth.js với phiên bản khởi tạo
cat << 'EOF' > auth.js
// Module Authentication
function login(username, pasword) {
    console.log("Logging in with: " + username);
}
EOF
git add auth.js
git commit -m "feat: khoi tao module auth"

# 2. Commit 2: Sửa lỗi chính tả biến pasword -> password
cat << 'EOF' > auth.js
// Module Authentication
function login(username, password) {
    console.log("Logging in with: " + username);
}
EOF
git add auth.js
git commit -m "fix typo"

# 3. Commit 3: Bổ sung các hàm tiện ích logout, verifyToken
cat << 'EOF' > auth.js
// Module Authentication
function login(username, password) {
    console.log("Logging in with: " + username);
}

function logout() {
    console.log("User logged out");
}

function verifyToken(token) {
    return token !== null && token !== undefined;
}

module.exports = { login, logout, verifyToken };
EOF
git add auth.js
git commit -m "adds utility functions"

# 4. Commit 4: Tạo file rác tạm thời để debug
echo "Temporary debug log - to be deleted" > temp.txt
git add temp.txt
git commit -m "add temp file for debug"
```

#### Kiểm tra lịch sử commit trước khi Rebase:
```bash
git log --oneline
```

**Kết quả ghi nhận:**
```text
aa0cf47 add temp file for debug
d22487e adds utility functions
6637443 fix typo
fc2cd7a feat: khoi tao module auth
```

---

### Bước 2: Kích hoạt chế độ Interactive Rebase

Để tái cấu trúc lại 4 commit vừa tạo, chạy lệnh Interactive Rebase lùi về 4 commit tính từ con trỏ `HEAD`:

```bash
git rebase -i HEAD~4
```
*(Hoặc `git rebase -i --root` nếu đây là 4 commit đầu tiên của repository).*

---

### Bước 3: Cấu hình kịch bản Interactive Rebase (Giao diện soạn thảo `git-rebase-todo`)

Khi lệnh được thực thi, Git mở trình soạn thảo văn bản chứa danh sách 4 commit theo thứ tự từ cũ nhất đến mới nhất.

#### Giao diện soạn thảo cấu hình:
```text
pick fc2cd7a feat: khoi tao module auth
squash 6637443 fix typo
squash d22487e adds utility functions
drop aa0cf47 add temp file for debug

# Rebase fc2cd7a..aa0cf47 onto 0000000 (4 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup [-C | -c] <commit> = like "squash" but keep only previous commit's log message
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
```

#### Giải thích chi tiết các chỉ thị đã cấu hình:
| Commit Hash | Lệnh cấu hình | Ý nghĩa & Cơ chế thực hiện |
| :--- | :---: | :--- |
| `fc2cd7a` | `pick` | Giữ lại commit cơ sở đầu tiên làm đích để gộp các commit tiếp theo vào. |
| `6637443` | `squash` (hoặc `s`) | Gộp nội dung commit "fix typo" vào commit liền trước (`fc2cd7a`), kết hợp nội dung commit log. |
| `d22487e` | `squash` (hoặc `s`) | Tiếp tục gộp commit "adds utility functions" vào khối commit trên. Toàn bộ thay đổi của 3 commit đầu giờ hợp nhất làm một. |
| `aa0cf47` | `drop` (hoặc `d`) | Loại bỏ hoàn toàn commit này. Git sẽ bỏ qua commit `aa0cf47`, không áp dụng các thay đổi của nó, dẫn đến file `temp.txt` biến mất hoàn toàn. |

---

### Bước 4: Soạn thảo thông điệp commit hợp nhất (`COMMIT_EDITMSG`)

Sau khi lưu và đóng file cấu hình todo, Git bắt đầu quá trình rebase. Khi đến bước tổng hợp message cho khối commit gộp (`squash`), Git mở tiếp giao diện soạn thảo thông điệp commit:

```text
# This is a combination of 3 commits.
# This is the 1st commit message:

feat: khoi tao module auth

# This is the commit message #2:

fix typo

# This is the commit message #3:

adds utility functions

# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
```

Tiến hành xóa toàn bộ nội dung cũ và thay thế bằng thông điệp chuẩn mực theo yêu cầu đề bài:
```text
feat: hoan thien module authentication
```

Lưu và đóng trình soạn thảo. Git hoàn tất quy trình rebase với thông báo:
```text
[detached HEAD 6178295] feat: hoan thien module authentication
 Date: Mon Oct 5 15:45:52 2026 +0700
 1 file changed, 14 insertions(+)
 create mode 100644 auth.js
Successfully rebased and updated refs/heads/main.
```

---

## 3. KẾT QUẢ KIỂM TRA ĐÁNH GIÁ

### 3.1. Kiểm tra lịch sử commit sau khi hoàn tất
Chạy lệnh kiểm tra theo yêu cầu đề bài:
```bash
git log --oneline
```

**Kết quả thực tế:**
```text
6178295 feat: hoan thien module authentication
```

> **Đánh giá:** Lịch sử commit giờ chỉ còn duy nhất 1 commit sạch sẽ, thông điệp rõ ràng, chuẩn định dạng Conventional Commits. Toàn bộ các commit rác và vụn vặt đã bị loại bỏ hoàn toàn.

---

### 3.2. Kiểm tra tính toàn vẹn của mã nguồn (`Working Tree`)

Kiểm tra trạng thái thư mục làm việc:
```bash
git status
```
```text
On branch main
nothing to commit, working tree clean
```

Kiểm tra danh sách tệp tin trong thư mục:
```bash
ls -la
```
```text
auth.js
```
> **Đánh giá:** Tệp rác `temp.txt` đã bị xóa hoàn toàn khỏi thư mục làm việc và không còn tồn tại trong cây Git.

Kiểm tra nội dung tệp `auth.js`:
```bash
cat auth.js
```
```javascript
// Module Authentication
function login(username, password) {
    console.log("Logging in with: " + username);
}

function logout() {
    console.log("User logged out");
}

function verifyToken(token) {
    return token !== null && token !== undefined;
}

module.exports = { login, logout, verifyToken };
```
> **Đánh giá:** Mã nguồn của `auth.js` bảo toàn 100% tất cả các bản sửa lỗi và tính năng từ cả 3 commit (`khởi tạo`, `fix typo`, `utility functions`), đảm bảo chức năng hoạt động hoàn chỉnh mà không bị mất mát dữ liệu.

---

## 4. SƠ ĐỒ BIẾN ĐỔI LỊCH SỬ (BEFORE & AFTER)

```text
LỊCH SỬ TRƯỚC KHI REBASE:
(fc2cd7a)               (6637443)              (d22487e)                    (aa0cf47)
[feat: khoi tao auth] -> [fix typo] ---------> [adds utility functions] -> [add temp file for debug] (HEAD)
      |                       |                      |                            |
      |                       |                      |                            X (drop)
      +-----------------------+----------------------+
                              |
                              v (squash + reword)
LỊCH SỬ SAU KHI REBASE:
(6178295)
[feat: hoan thien module authentication] (HEAD -> main)
```

---

## 5. BẢNG TỔNG HỢP CÁC LỆNH INTERACTIVE REBASE PHỔ BIẾN

| Lệnh (Command) | Phím tắt | Ý nghĩa thực tế |
| :--- | :---: | :--- |
| **`pick`** | `p` | Sử dụng commit đó (giữ nguyên không thay đổi). |
| **`reword`** | `r` | Sử dụng commit, nhưng cho phép sửa lại thông điệp commit. |
| **`edit`** | `e` | Sử dụng commit, nhưng dừng lại để người dùng chỉnh sửa nội dung mã nguồn (bằng `git commit --amend`). |
| **`squash`** | `s` | Gộp commit vào commit liền trước và hợp nhất thông điệp commit. |
| **`fixup`** | `f` | Gộp commit vào commit liền trước nhưng tự động bỏ qua thông điệp của commit này (chỉ giữ thông điệp của commit trước). |
| **`drop`** | `d` | Xóa bỏ hoàn toàn commit khỏi lịch sử. |

---

## 6. KẾT LUẬN & BÀI HỌC KINH NGHIỆM

1. **Nguyên tắc "Golden Rule of Rebase":** Chỉ thực hiện Interactive Rebase trên các nhánh làm việc cá nhân (Local Branch) chưa đẩy lên Remote. Tuyệt đối không rebase trên các nhánh công cộng (`main`, `develop`) đã chia sẻ với các thành viên khác trong nhóm.
2. **Lợi ích:** Interactive Rebase là công cụ then chốt giúp xây dựng lịch sử commit nguyên tử (Atomic Commits), dễ đọc, dễ hiểu, tăng hiệu quả phối hợp và đảm bảo chất lượng kỹ thuật trong các dự án chuyên nghiệp.
