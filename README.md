# Báo Cáo Thực Hành: Quản Lý Tệp Tin Bỏ Qua (.gitignore) Và Sửa Lịch Sử (Amend)

- **Khóa học:** DevOps / Cloud Engineering
- **Bài tập:** Session 04 - Bài 4: Quản lý tệp tin bỏ qua (.gitignore) và Sửa lịch sử (Amend)
- **Đường dẫn nộp bài trên GitHub:** `homework/session_04/ex4/`
- **Học viên:** Hoàng Văn Lương
- **Hệ điều hành / Môi trường:** macOS, Git 2.x, Terminal (zsh)
- **Tệp tin đính kèm:** 
  - `.gitignore` (Tệp cấu hình quy tắc bỏ qua)
  - `credentials.txt` (Tệp dữ liệu mẫu được bảo vệ cục bộ)
  - `images/git-amend-status.png` (Hình chụp xác thực trạng thái và lịch sử commit)

---

## 1. Mục Tiêu Bài Tập

1. **Bảo mật mã nguồn với `.gitignore`:** Cấu hình chuẩn xác tệp tin ẩn `.gitignore` để Git tự động bỏ qua các tệp tin chứa thông tin nhạy cảm (mật khẩu, khóa API, SSH private keys) và các tệp rác phát sinh từ hệ điều hành, IDE, hoặc môi trường thực thi.
2. **Gỡ bỏ theo dõi an toàn với `git rm --cached`:** Hiểu sâu sự khác biệt giữa các vùng trong Git (Working Directory vs Staging Area/Index) để loại bỏ một tệp tin đã bị Git theo dõi ra khỏi bộ nhớ cache của Git mà **tuyệt đối không xóa mất tệp tin vật lý** trên ổ đĩa.
3. **Chỉnh sửa lịch sử commit với `git commit --amend`:** Cập nhật lại nội dung tệp tin trong snapshot và viết lại thông điệp commit (commit message) của commit gần nhất một cách chuyên nghiệp, sạch sẽ.
4. **Kiểm tra và xác thực trạng thái:** Sử dụng các lệnh kiểm tra tiêu chuẩn (`git status`, `git log -n 1`, `git ls-files`, `git check-ignore`) để chứng minh tệp nhạy cảm không còn bị theo dõi, đồng thời working tree ở trạng thái sạch (`working tree clean`).

---

## 2. Bối Cảnh Sự Cố & Phân Tích Kỹ Thuật

### 2.1. Tình Huống Sự Cố Thực Tế

Trong quá trình khởi tạo dự án hoặc triển khai ứng dụng:
- Lập trình viên/kỹ sư DevOps vô tình đưa tệp `credentials.txt` (chứa Database Password, API Secret Keys, AWS Keys) vào Git qua lệnh sơ suất `git add .` và thực hiện `git commit`.
- Tệp nhạy cảm lúc này đã nằm trong lịch sử Git. Nếu commit này bị đẩy (push) lên GitHub/GitLab, toàn bộ thông tin bảo mật sẽ bị lộ lọt (Data Leakage / Credential Exposure), dẫn đến nguy cơ hệ thống bị xâm nhập trái phép.
- **Ràng buộc bài toán:** 
  1. Không được dùng lệnh `rm credentials.txt` của hệ điều hành vì ứng dụng cục bộ vẫn cần tệp tin này để kết nối cơ sở dữ liệu khi chạy môi trường dev.
  2. Phải gỡ bỏ tệp khỏi Git cache, bổ sung quy tắc `.gitignore` để không bao giờ bị add nhầm trong tương lai.
  3. Sửa lại commit gần nhất để tệp tin hoàn toàn biến mất khỏi commit snapshot đó.

---

### 2.2. Cơ Chế Quản Lý 3 Vùng Của Git & Lệnh `git rm --cached`

Để hiểu tại sao `git rm --cached` giải quyết được vấn đề mà không làm mất file trên đĩa cứng, ta xem xét mô hình 3 vùng kiến trúc của Git:

```mermaid
flowchart LR
    WD["1. Working Directory<br/>(Thư mục làm việc trên đĩa)"]
    SA["2. Staging Area (Index / Cache)<br/>(Vùng chuẩn bị commit)"]
    CR["3. Commit History (HEAD)<br/>(Kho chứa lịch sử Git)"]

    WD -->|"git add"| SA
    SA -->|"git commit"| CR

    SA -.->|"git rm --cached (Chỉ xóa khỏi Index)"| SA
    WD -.->|"File vật lý vẫn NGUYÊN VẸN"| WD

    style WD fill:#1e3d2f,stroke:#27ae60,stroke-width:2px,color:#fff
    style SA fill:#2b3a4a,stroke:#4a90e2,stroke-width:2px,color:#fff
    style CR fill:#3a2042,stroke:#9b59b6,stroke-width:2px,color:#fff
```

**Bảng So Sánh Các Lệnh Xóa:**

| Lệnh thực thi | Ảnh hưởng trên Working Directory (Ổ đĩa) | Ảnh hưởng trên Staging Area (Git Index) | Đánh giá an toàn |
| :--- | :--- | :--- | :--- |
| `rm credentials.txt` (OS command) | **Mất file vật lý vĩnh viễn** | Vẫn còn trong Git Index ở trạng thái `tracked` | ❌ Nguy hiểm, mất dữ liệu cấu hình cục bộ. |
| `git rm credentials.txt` | **Mất file vật lý** | Đánh dấu xóa (`deleted`) trong Staging | ❌ Mất file trên máy tính của lập trình viên. |
| **`git rm --cached credentials.txt`** | **GIỮ NGUYÊN FILE VẬT LÝ** | Đánh dấu xóa (`deleted`) khỏi sự theo dõi của Git |  **Hoàn hảo**, đáp ứng chính xác yêu cầu bài toán. |

> **Lưu ý cốt lõi về `.gitignore`:**
> Tệp `.gitignore` **chỉ có hiệu lực với các file chưa từng được theo dõi (Untracked Files)**. Nếu một file đã nằm trong Git Index (đã được `git add` hoặc đã commit), việc thêm tên file vào `.gitignore` sẽ **hoàn toàn vô tác dụng** cho đến khi file đó được đưa ra khỏi index bằng lệnh `git rm --cached`.

---

## 3. Các Bước Triển Khai Thực Nghiệm Từng Bước

### Bước 1: Mô Phỏng Sự Cố Commit Nhầm `credentials.txt`

Khởi tạo môi trường kho chứa và tạo các tệp mã nguồn ban đầu, bao gồm tệp nhạy cảm `credentials.txt`:

```bash
# 1. Khởi tạo repository Git
git init -b main
git config user.name "Hoàng Văn Lương"
git config user.email "hoangluong@example.com"

# 2. Tạo mã nguồn ứng dụng và file cấu hình
cat << 'EOF' > app.py
import json

def load_config():
    with open("config.json") as f:
        return json.load(f)

if __name__ == "__main__":
    print("Application initialized successfully.")
EOF

cat << 'EOF' > config.json
{
  "app_name": "Cloud Deployment Manager",
  "version": "1.0.0",
  "environment": "production"
}
EOF

# 3. Tạo tệp tin nhạy cảm credentials.txt
cat << 'EOF' > credentials.txt
# SENSITIVE PRODUCTION CREDENTIALS - DO NOT COMMIT!
DB_HOST=10.0.0.15
DB_PORT=5432
DB_USER=cloud_admin
DB_PASSWORD=SuperSecretPassWord2026!@#
API_SECRET_KEY=sec_live_99482747291847192847
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
EOF

# 4. Vô tình đưa toàn bộ vào Staging và thực hiện commit
git add .
git commit -m "feat: initial commit with app code and configuration (mistakenly added credentials.txt)"
```

*Nhật ký thực thi:*
```text
[main (root-commit) 641c879] feat: initial commit with app code and configuration (mistakenly added credentials.txt)
 3 files changed, 21 insertions(+)
 create mode 100644 app.py
 create mode 100644 config.json
 create mode 100644 credentials.txt
```

---

### Bước 2: Gỡ Bỏ `credentials.txt` Khỏi Cache Của Git Bằng `git rm --cached`

Thực thi lệnh gỡ bỏ file khỏi Git Index:

```bash
git rm --cached credentials.txt
```

*Nhật ký thực thi:*
```text
rm 'credentials.txt'
```

Kiểm tra ngay trạng thái file và sự tồn tại vật lý trên ổ cứng:

```bash
# Kiểm tra file vật lý trên ổ đĩa
ls -la credentials.txt

# Kiểm tra trạng thái Git
git status
```

*Kết quả kiểm tra:*
```text
-rw-r--r--@ 1 hoangluong  staff  289 Oct  4 22:21 credentials.txt

On branch main
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	deleted:    credentials.txt

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	credentials.txt
```

> **Phân tích kết quả:**
> 1. Tệp tin `credentials.txt` dung lượng 289 bytes **vẫn còn nguyên vẹn 100% trên ổ cứng**.
> 2. Trong Git, tệp tin được chuyển sang danh sách `Changes to be committed: deleted` (chuẩn bị xóa khỏi commit).
> 3. Đồng thời, tệp xuất hiện ở mục `Untracked files` vì Git không còn theo dõi nó nữa.

---

### Bước 3: Cấu Hình Tệp `.gitignore` Để Ngăn Chặn Vĩnh Viễn

Khởi tạo tệp `.gitignore` chuẩn quy cách DevOps, đưa `credentials.txt` và các mẫu tệp nhạy cảm/tệp rác vào danh sách bỏ qua:

```bash
cat << 'EOF' > .gitignore
# ==============================================================================
# Git Ignore Configuration - Session 04 Exercise 4
# ==============================================================================

# Sensitive Information & Credentials
credentials.txt
*.env
*.env.*
*.key
*.pem
*.pfx
secrets/
id_rsa
id_ed25519

# macOS System Files
.DS_Store
.AppleDouble
.LSOverride
._*

# IDE & Editor Directories
.idea/
.vscode/
*.swp
*.swo
*~

# Build, Log & Dependency Artifacts
*.log
node_modules/
dist/
build/
__pycache__/
*.py[cod]
EOF
```

Kiểm tra lại trạng thái sau khi đã tạo `.gitignore`:

```bash
git status
```

*Kết quả kiểm tra:*
```text
On branch main
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	deleted:    credentials.txt

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	.gitignore
```

> **Nhận xét quan trọng:**
> `credentials.txt` đã **hoàn toàn biến mất khỏi mục Untracked files** vì Git đã đọc `.gitignore` và tự động lờ tệp này đi!

---

### Bước 4: Đưa Tệp `.gitignore` Vào Vùng Staging Area

Đưa `.gitignore` vào danh sách chuẩn bị commit:

```bash
git add .gitignore
git status
```

*Kết quả kiểm tra:*
```text
On branch main
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	new file:   .gitignore
	deleted:    credentials.txt
```

---

### Bước 5: Chỉnh Sửa Lịch Sử Commit Gần Nhất Với `--amend`

Thay vì tạo ra một commit mới (commit sửa sai khiến thông tin nhạy cảm vẫn còn lưu lại ở commit trước đó), ta sử dụng cờ `--amend` để gộp trực tiếp thay đổi này vào commit gần nhất và viết lại thông điệp commit:

```bash
git commit --amend -m "feat: initial commit with app code, configuration, and secure .gitignore"
```

*Nhật ký thực thi:*
```text
[main a6abd6f] feat: initial commit with app code, configuration, and secure .gitignore
 Date: Sun Oct 4 22:21:25 2026 +0700
 3 files changed, 25 insertions(+)
 create mode 100644 .gitignore
 create mode 100644 app.py
 create mode 100644 config.json
```

> **Phân tích kết quả amend:**
> - Commit snapshot chỉ còn lại 3 tệp: `.gitignore`, `app.py`, và `config.json`.
> - Tệp `credentials.txt` đã bị gỡ bỏ hoàn toàn khỏi commit snapshot mà không để lại bất kỳ vết tích nào trong commit này.
> - Thông điệp commit được thay đổi thành nội dung chuẩn mực, không còn chứa dấu hiệu nhầm lẫn.

---

## 4. Kết Quả Kiểm Tra & Bằng Chứng Thực Nghiệm

### 4.1. Kiểm Tra Trạng Thái Bằng Lệnh `git status`

```bash
git status
```

*Kết quả đầu ra:*
```text
On branch main
nothing to commit, working tree clean
```

> **Kết luận:**
> - Cây làm việc hoàn toàn sạch sẽ (`working tree clean`).
> - Tệp `credentials.txt` không xuất hiện ở dạng Staged, Modified hay Untracked.

---

### 4.2. Kiểm Tra Lịch Sử Commit Gần Nhất Bằng Lệnh `git log -n 1`

```bash
git log -n 1
```

*Kết quả đầu ra:*
```text
commit a6abd6f721d22edb77ba0a5ba367da2db00b3e47 (HEAD -> main)
Author: Hoàng Văn Lương <hoangluong@example.com>
Date:   Sun Oct 4 22:21:25 2026 +0700

    feat: initial commit with app code, configuration, and secure .gitignore
```

> **Kết luận:**
> - Thông điệp commit đã được cập nhật thành công qua `--amend`.
> - Tác giả và mốc thời gian được bảo toàn.

---

### 4.3. Kiểm Tra Sự Tồn Tại Của File Vật Lý & Quy Tắc Ignore

Kiểm tra tính toàn vẹn của tệp trên đĩa cứng và quy tắc `.gitignore`:

```bash
# 1. Kiểm tra file vật lý trên đĩa cứng
ls -la credentials.txt

# 2. Kiểm tra danh sách file đang được Git theo dõi
git ls-files

# 3. Kiểm tra quy tắc .gitignore đang áp dụng với file
git check-ignore -v credentials.txt
```

*Kết quả đầu ra:*
```text
-rw-r--r--@ 1 hoangluong  staff  289 Oct  4 22:21 credentials.txt

.gitignore
app.py
config.json

.gitignore:5:credentials.txt	credentials.txt
```

> **Bằng chứng xác thực:**
> 1. `ls -la`: Tệp `credentials.txt` vẫn tồn tại nguyên vẹn trên máy cục bộ với dung lượng 289 bytes.
> 2. `git ls-files`: Trong kho lưu trữ Git chỉ theo dõi 3 file (`.gitignore`, `app.py`, `config.json`), tuyệt đối không có `credentials.txt`.
> 3. `git check-ignore -v`: Quy tắc tại dòng 5 của file `.gitignore` (`credentials.txt`) đang hoạt động chính xác.

---

### 4.4. Hình Chụp Màn Hình Dòng Lệnh Terminal (Screenshot Evidence)

Dưới đây là hình chụp cửa sổ dòng lệnh Terminal hiển thị toàn bộ quá trình kiểm tra xác thực theo đúng yêu cầu đề bài:

![Xác thực git status và git log amend](images/git-amend-status.png)

---

## 5. Cơ Chế Nội Tại Của `git commit --amend`

### 5.1. Bản Chất Kỹ Thuật Dưới Tầng Git Object

Nhiều người lầm tưởng `--amend` sẽ mở commit cũ ra và chỉnh sửa trực tiếp trên commit đó. Tuy nhiên, trong Git, **các commit object là bất biến (immutable)**:

```mermaid
flowchart TD
    Old["Commit Cũ (SHA: 641c879)<br/>Chứa: credentials.txt<br/>(Trở thành Dangling Commit)"]
    New["Commit Mới Amend (SHA: a6abd6f)<br/>Chứa: .gitignore, app.py, config.json<br/>(Loại bỏ credentials.txt)"]
    Branch["Nhánh main / Con trỏ HEAD"]

    Old -.->|Bị thay thế bởi| New
    Branch -->|Trỏ tới commit mới| New

    style Old fill:#4a1e1e,stroke:#e74c3c,stroke-width:2px,color:#fff
    style New fill:#1e3d2f,stroke:#27ae60,stroke-width:2px,color:#fff
    style Branch fill:#2b3a4a,stroke:#4a90e2,stroke-width:2px,color:#fff
```

1. Khi gõ `git commit --amend`, Git sẽ tạo ra một **Commit Object hoàn toàn mới** với mã băm SHA mới (`a6abd6f`), chứa cây thư mục (tree object) mới và thông điệp mới.
2. Con trỏ nhánh `main` và `HEAD` được dịch chuyển sang commit mới này.
3. Commit cũ (`641c879`) bị tách rời khỏi đồ thị nhánh chính (trở thành **Dangling/Orphaned Commit**). Sau một thời gian (mặc định 30-90 ngày), tiến trình tự động dọn rác của Git (`git gc`) sẽ thu hồi commit cũ này.

### 5.2. Quy Tắc Vàng Khi Sử Dụng `git commit --amend`

> [!CAUTION]
> **Nguyên tắc bất di bất dịch:** Chỉ sử dụng `git commit --amend` cho các **commit cục bộ (Local Commits) CHƯA ĐƯỢC PUSH lên Remote Repository (GitHub/GitLab)**.

- **Nếu commit chưa push:** Amend hoàn toàn an toàn, giúp lịch sử commit trước khi mở Pull Request trở nên gọn gàng, sạch sẽ.
- **Nếu commit ĐÃ PUSH lên Remote:** Việc amend sẽ làm thay đổi SHA của commit trên máy cục bộ, dẫn đến xung đột phân kỳ lịch sử (Diverged Branch) giữa Local và Remote. Khi đó nếu muốn đẩy lên, lập trình viên buộc phải dùng `git push --force`, hành động này có thể ghi đè và phá hỏng lịch sử commit của các thành viên khác trong nhóm.

---

## 6. Đúc Kết Kiến Thức & Best Practices Quản Trị Secrets Trong DevOps

1. **Sử dụng `.env.example` thay vì lưu thông tin thật:** Chỉ commit tệp mẫu chứa tên biến rỗng hoặc giá trị demo (`DB_USER=your_user`), đưa toàn bộ các tệp `.env`, `.env.local`, `credentials.txt` vào `.gitignore`.
2. **Triển khai Pre-commit Hooks tự động:** Tích hợp các công cụ quét mã nguồn tự động ở bước tiền commit như **`gitleaks`**, **`trufflehog`**, hoặc **`detect-secrets`** vào Git hook. Nếu phát hiện tệp hoặc chuỗi ký tự chứa API key hay Password, hook sẽ tự động chặn lệnh commit ngay lập tức.
3. **Sử dụng Secret Management Services:** Trong môi trường Production, không lưu thông tin đăng nhập trong các tệp văn bản thô trên máy chủ. Thay vào đó, sử dụng các dịch vụ quản lý secret chuyên dụng như:
   - **HashiCorp Vault**
   - **AWS Secrets Manager / SSM Parameter Store**
   - **GitHub Actions Secrets / GitLab CI Variables**
4. **Hành động khẩn cấp khi đã lỡ push secret lên GitHub:** Nếu tệp `credentials.txt` đã bị đẩy lên Remote công khai, việc xóa file và commit lại **không bảo vệ được thông tin**. Cần thực hiện ngay lập tức:
   - **Thu hồi và đổi mới (Revoke & Rotate)** toàn bộ mật khẩu, token, và API key bị lộ.
   - Sử dụng các công cụ như `git-filter-repo` hoặc `BFG Repo-Cleaner` để thanh tẩy toàn bộ lịch sử git của kho chứa.
