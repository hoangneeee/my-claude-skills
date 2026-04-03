---
name: release-management
description: Git release management with semantic versioning, release branches, and hotfix workflow. Apply when creating releases, tagging versions, hotfixing, or managing multi-customer deployments.
allowed-tools: Read, Glob, Grep, Bash, Write, Edit
---

# Release Management

Hướng dẫn quản lý release cho mọi dự án với git tags, release branches, và semantic versioning.

## 1. Branching Model

```
main ──────────────────────────────────────────► (stable, production-ready)
  │
  └── develop ─────────────────────────────────► (tích hợp features)
        │
        ├── feat-xxx ──► merge vào develop
        │
        └── release/3.5.x ────────────────────► (hotfix cho khách hàng dùng 3.5.x)
              │    tags: v3.5.0, v3.5.1, v3.5.2
              │
            release/3.6.x ────────────────────► (hotfix cho khách hàng dùng 3.6.x)
                   tags: v3.6.0, v3.6.1
```

### Quy tắc

- `develop`: nhánh tích hợp chính, mọi feature merge vào đây
- `release/X.Y.x`: tạo khi release version mới, dùng để hotfix cho version đó
- Tag `vX.Y.Z`: đánh dấu mỗi release cụ thể
- Không commit trực tiếp vào `main` hoặc `release/*` mà không qua quy trình

## 2. Semantic Versioning

Format: `MAJOR.MINOR.PATCH` (ví dụ: `3.5.1`)

- **MAJOR** (3.x.x): breaking changes, không backward compatible
- **MINOR** (x.5.x): features mới, backward compatible
- **PATCH** (x.x.1): bug fixes, không thay đổi API

Tag format: `vX.Y.Z` (ví dụ: `v3.5.1`)
Release branch format: `release/X.Y.x` (ví dụ: `release/3.5.x`)

## 3. Version Detection (đa ngôn ngữ)

Khi user yêu cầu thao tác release, **tự động detect** loại project và đọc version:

### Rust

```
File: Cargo.toml (workspace root)
Read: version = "X.Y.Z"
Update: sed hoặc Edit tool thay đổi giá trị version
```

### Node.js

```
File: package.json
Read: "version": "X.Y.Z"
Update: Edit tool hoặc `npm version X.Y.Z --no-git-tag-version`
```

### Python

```
File: pyproject.toml
Read: version = "X.Y.Z"
Update: Edit tool thay đổi giá trị version
```

### Go

```
File: không có version file
Read: lấy version từ git tag mới nhất (`git describe --tags --abbrev=0`)
Update: chỉ cần tạo git tag
```

### Thứ tự detect

1. Kiểm tra `Cargo.toml` → Rust
2. Kiểm tra `package.json` → Node.js
3. Kiểm tra `pyproject.toml` → Python
4. Fallback → Go-style (git tags only)

## 4. Workflows

### 4.1 Create Release

Khi user yêu cầu: "tạo release X.Y.Z", "release version X.Y.Z"

Các bước thực hiện:

```
1. SAFETY CHECKS
   - Verify working tree sạch: `git status --porcelain`
   - Verify đang ở nhánh develop: `git rev-parse --abbrev-ref HEAD`
   - Verify tag chưa tồn tại: `git tag -l vX.Y.Z`

2. BUMP VERSION
   - Detect loại project (xem mục 3)
   - Đọc version hiện tại
   - Nếu version khác với target → update file version
   - `git add <version-file>`
   - `git commit -m "chore: bump version to X.Y.Z"`

3. CREATE TAG
   - `git tag -a vX.Y.Z -m "Release X.Y.Z"`

4. CREATE RELEASE BRANCH
   - Branch name: release/X.Y.x
   - Nếu branch chưa tồn tại: `git branch release/X.Y.x vX.Y.Z`
   - Nếu đã tồn tại: skip, thông báo cho user

5. REPORT
   - In tóm tắt: tag, branch đã tạo
   - Gợi ý lệnh push: `git push origin develop vX.Y.Z release/X.Y.x`
```

### 4.2 Hotfix

Khi user yêu cầu: "hotfix release/3.5.x", "tạo hotfix cho 3.5"

Điều kiện: user đã commit fixes trên release branch rồi.

```
1. SAFETY CHECKS
   - Verify đang ở đúng release branch
   - Verify không có uncommitted changes
   - Verify có commits mới kể từ tag gần nhất:
     latest_tag = `git tag --merged HEAD --sort=-v:refname | head -1`
     `git log $latest_tag..HEAD --oneline`

2. BUMP PATCH VERSION
   - Đọc version hiện tại
   - Tăng patch: X.Y.Z → X.Y.(Z+1)
   - Update file version
   - `git add <version-file>`
   - `git commit -m "chore: bump version to X.Y.(Z+1)"`

3. CREATE TAG
   - `git tag -a vX.Y.(Z+1) -m "Release X.Y.(Z+1) - hotfix"`

4. REPORT
   - In tóm tắt
   - Gợi ý push: `git push origin release/X.Y.x vX.Y.(Z+1)`
   - Gợi ý cherry-pick về develop nếu cần:
     `git checkout develop && git cherry-pick <commit-hash>`
```

### 4.3 List Releases

Khi user yêu cầu: "list releases", "xem các version"

```
1. RELEASE BRANCHES
   - `git branch --list "release/*"`
   - Với mỗi branch, tìm tag mới nhất: `git tag --merged <branch> --sort=-v:refname`

2. TAGS
   - `git tag -l "v*" --sort=-v:refname`
   - Với mỗi tag, lấy ngày: `git log -1 --format=%ai <tag>`

3. FORMAT
   - In bảng rõ ràng với branch, latest tag, và dates
```

### 4.4 Release Info

Khi user yêu cầu: "thông tin version 3.5.1", "release info v3.5.1"

```
1. LOOKUP
   - Normalize: thêm prefix "v" nếu chưa có
   - `git show <tag> --no-patch`

2. DETAILS
   - Tagger, date, message
   - Số commits kể từ tag trước: `git rev-list --count <prev-tag>..<tag>`
   - Files changed: `git diff --stat <prev-tag>..<tag>`
```

## 5. Safety Rules

Luôn tuân thủ:

- **KHÔNG** thao tác khi có uncommitted changes
- **KHÔNG** tạo tag trùng tên
- **KHÔNG** force push hoặc delete tags
- **KHÔNG** commit trực tiếp vào release branch mà không qua hotfix workflow
- **LUÔN** confirm với user trước khi push lên remote
- **LUÔN** tạo tag annotated (`-a`), không dùng lightweight tag
- **LUÔN** kiểm tra đúng branch trước khi thao tác

## 6. Multi-customer Deployment

Khi phân phối ứng dụng cho nhiều khách hàng ở các version khác nhau:

```
Khách A dùng v3.5.1 → hotfix trên release/3.5.x → tag v3.5.2 → build cho khách A
Khách B dùng v3.6.0 → hotfix trên release/3.6.x → tag v3.6.1 → build cho khách B
Khách C cần version mới → tạo release 3.7.0 từ develop → build cho khách C
```

Quy tắc:

- Mỗi khách hàng track qua tag version đang deploy
- Hotfix cho version cũ không ảnh hưởng version mới
- Cherry-pick fixes quan trọng ngược lên develop
- Build installer/artifact từ đúng tag: `git checkout vX.Y.Z && <build-command>`
