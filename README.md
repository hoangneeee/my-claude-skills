# my-claude-skills

Kho lưu trữ Claude Code skills cho các dự án của MBFS AI Team.

## Cách dùng

Copy skill vào project cần dùng:

```bash
cp -r skills/<skill-name>/ <path-to-project>/.claude/skills/<skill-name>/
```

Sau đó Claude sẽ tự động load skill khi làm việc trong project đó.

---

## Danh sách Skills

### `mbfs-aiteam-fe` — Frontend React

Dành cho các project frontend với stack: React 19, Shadcn UI (new-york), Tailwind CSS, Zustand, React Hook Form, TanStack Query, Axios.

Bao gồm hướng dẫn về:
- Component & Form pattern
- Page/Feature structure
- API Service layer
- Nav Data Frontend (thêm menu item + permission)
- AuthGuard (route-level, component-level, hook)
- Zustand store
- Routing với React Router v7

```bash
cp -r skills/mbfs-aiteam-fe/ <project>/.claude/skills/mbfs-aiteam-fe/
```

---

### `mbfs-aiteam-be` — Backend Rust

Dành cho các project backend với stack: Axum, SQLx + PostgreSQL, JWT, bcrypt, utoipa, validator, tracing.

Bao gồm hướng dẫn về:
- Layered architecture (Handler → Service → Repository)
- Handler pattern (AuthClaims, check_permission, ApiResponse)
- Repository pattern (sqlx compile-time queries, soft delete)
- Service layer (business logic, error handling)
- Request/Response DTOs
- Auth & Permission middleware
- Route registration
- Config (YAML + env override)
- Database migration

```bash
cp -r skills/mbfs-aiteam-be/ <project>/.claude/skills/mbfs-aiteam-be/
```

---

### `henry-mindset-code` — Engineering Mindset

Mindset và nguyên tắc tư duy dùng chung cho mọi dự án. Áp dụng khi phân tích yêu cầu, ra quyết định kỹ thuật, debug, và tiếp cận bất kỳ task nào.

Bao gồm:
- Think Before Code
- Keep It Simple (KISS)
- Iterative Approach
- Understand the Context
- Trade-off Thinking
- Ownership & Craftsmanship
- Debug Systematically
- Communication First

```bash
cp -r skills/henry-mindset-code/ <project>/.claude/skills/henry-mindset-code/
```

---

### `release-management` — Release Management

Quản lý release cho mọi dự án với git tags, release branches, và semantic versioning. Hỗ trợ đa ngôn ngữ: Rust, Node.js, Python, Go.

Bao gồm:
- Branching model (develop → release/X.Y.x → tags)
- Semantic Versioning rules
- Auto-detect project type và version
- Create release workflow
- Hotfix workflow
- Multi-customer deployment guide
- Safety checks

```bash
cp -r skills/release-management/ <project>/.claude/skills/release-management/
```
