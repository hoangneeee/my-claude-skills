---
name: mbfs-aiteam-be
description: Rust backend development guide for MBFS AI Team projects using Axum, SQLx + PostgreSQL, JWT auth, bcrypt, utoipa, validator, and tracing. Use when creating handlers, services, repositories, DTOs, routes, or managing auth/permissions.
allowed-tools: Read, Glob, Grep, Write, Edit, Bash
---

# MBFS AI Team Rust Backend Development Guide

## Stack

- **Axum 0.8** — web framework
- **Tokio** — async runtime (full features)
- **SQLx 0.8** — PostgreSQL, compile-time checked queries
- **serde / serde_json / serde_yaml** — serialization
- **utoipa 5** — OpenAPI documentation
- **validator 0.19** — request validation
- **jsonwebtoken 9** — JWT auth
- **bcrypt 0.15** — password hashing
- **tracing / tracing-subscriber** — structured logging
- **thiserror** — error types
- **tower / tower-http** — middleware (CORS, timeout, trace)

---

## 1. Layered Architecture

```
src/
├── api/          # HTTP handlers (axum Route handlers)
├── services/     # Business logic
├── repository/   # Data access (sqlx queries)
├── domain/       # Database models (structs mapping to DB rows)
├── models/       # Request/Response DTOs
├── middleware/   # JWT auth, permission checking
├── config/       # App configuration
├── error/        # AppError type
└── utils/        # Helpers
```

**Flow**: `Handler → Service → Repository → Database`

- Handlers: extract request, check permission, call service, return response
- Services: business rules, password hashing, error logic
- Repositories: SQL queries, return domain models
- Domain models are never returned directly — always map to response DTOs

---

## 2. Handler Pattern

```rust
// src/api/api_camera.rs
use axum::{extract::{Query, State}, Json};
use crate::{
    error::AppError,
    middleware::{auth::AuthClaims, permission::{check_permission, permissions}},
    models::{camera_model::{CameraCreateRequest, CameraItemResponse}, response::ApiResponse},
    services::service_camera,
    AppState,
};

#[utoipa::path(
    get,
    path = "/camera",
    tag = "Camera",
    responses((status = 200, description = "Camera list", body = Vec<CameraItemResponse>))
)]
pub async fn list_camera(
    AuthClaims(claims): AuthClaims,
    State(state): State<AppState>,
    Query(params): Query<CameraListParams>,
) -> Result<Json<ApiResponse<Vec<CameraItemResponse>>>, AppError> {
    check_permission(&claims, permissions::CAMERA_READ)?;

    let pool = state.db_pool.as_ref().ok_or_else(|| AppError::internal("DB not initialized"))?;
    let (items, total) = service_camera::list(pool, params.page, params.page_size).await?;

    Ok(Json(ApiResponse::success_with_pagination(
        items,
        params.page,
        params.page_size,
        total,
    )))
}

#[utoipa::path(post, path = "/camera/create", tag = "Camera")]
pub async fn create_camera(
    AuthClaims(claims): AuthClaims,
    State(state): State<AppState>,
    Json(req): Json<CameraCreateRequest>,
) -> Result<Json<ApiResponse<CameraItemResponse>>, AppError> {
    check_permission(&claims, permissions::CAMERA_MANAGE)?;

    let pool = state.db_pool.as_ref().ok_or_else(|| AppError::internal("DB not initialized"))?;
    let item = service_camera::create(pool, req).await?;

    Ok(Json(ApiResponse::success(item)))
}
```

**Rules:**
- Always start with `check_permission(&claims, permissions::XXX)?`
- Get pool from state with `.ok_or_else(|| AppError::internal(...))?`
- Return `Result<Json<ApiResponse<T>>, AppError>`
- Use `#[utoipa::path]` for OpenAPI docs on every handler

---

## 3. Repository Pattern

Repositories are modules with standalone async functions — no trait-based abstraction.

```rust
// src/repository/camera_repo.rs
use sqlx::{Pool, Postgres};
use crate::{domain::camera::Camera, error::AppError, models::camera_model::*};

pub async fn list_paginated(
    pool: &Pool<Postgres>,
    page: u32,
    page_size: u32,
) -> Result<(Vec<Camera>, u64), AppError> {
    let limit = page_size as i64;
    let offset = (page_size * page.saturating_sub(1)) as i64;

    let total: i64 = sqlx::query_scalar!(r#"SELECT COUNT(*) FROM camera WHERE is_delete = false"#)
        .fetch_one(pool)
        .await?
        .unwrap_or(0);

    let items = sqlx::query_as!(
        Camera,
        r#"SELECT id as "id!", name as "name!", location, is_active, created_at, updated_at
           FROM camera WHERE is_delete = false
           ORDER BY id DESC LIMIT $1 OFFSET $2"#,
        limit,
        offset,
    )
    .fetch_all(pool)
    .await?;

    Ok((items, total as u64))
}

pub async fn create(
    pool: &Pool<Postgres>,
    req: CameraCreateRequest,
) -> Result<Camera, AppError> {
    let item = sqlx::query_as!(
        Camera,
        r#"INSERT INTO camera (name, location, is_active)
           VALUES ($1, $2, $3)
           RETURNING id as "id!", name as "name!", location, is_active, created_at, updated_at"#,
        req.name,
        req.location,
        req.is_active.unwrap_or(true),
    )
    .fetch_one(pool)
    .await?;

    Ok(item)
}

pub async fn get_by_id(
    pool: &Pool<Postgres>,
    id: i32,
) -> Result<Option<Camera>, AppError> {
    let item = sqlx::query_as!(
        Camera,
        r#"SELECT id as "id!", name as "name!", location, is_active, created_at, updated_at
           FROM camera WHERE id = $1 AND is_delete = false"#,
        id,
    )
    .fetch_optional(pool)
    .await?;

    Ok(item)
}

// Always map domain model → response DTO in the repo module
pub fn map_to_response(c: Camera) -> CameraItemResponse {
    CameraItemResponse {
        id: c.id,
        name: c.name,
        location: c.location,
        is_active: c.is_active,
        created_at: c.created_at,
    }
}
```

**Rules:**
- Use `sqlx::query_as!` (compile-time checked) — never raw string queries
- Cast non-nullable columns with `as "col!"` syntax
- Use `RETURNING` for insert/update to avoid a second query
- Use `is_delete = false` for soft-delete filtering
- Provide a `map_to_response()` function per repo module

---

## 4. Service Layer

```rust
// src/services/service_camera.rs
use sqlx::{Pool, Postgres};
use crate::{
    error::AppError,
    models::camera_model::*,
    repository::camera_repo,
};

pub async fn list(
    pool: &Pool<Postgres>,
    page: u32,
    page_size: u32,
) -> Result<(Vec<CameraItemResponse>, u64), AppError> {
    let (items, total) = camera_repo::list_paginated(pool, page, page_size).await?;
    Ok((items.into_iter().map(camera_repo::map_to_response).collect(), total))
}

pub async fn create(
    pool: &Pool<Postgres>,
    req: CameraCreateRequest,
) -> Result<CameraItemResponse, AppError> {
    // Business rule validation
    if req.name.trim().is_empty() {
        return Err(AppError::bad_request("Camera name cannot be empty"));
    }

    let item = camera_repo::create(pool, req).await?;
    Ok(camera_repo::map_to_response(item))
}

pub async fn get_by_id(
    pool: &Pool<Postgres>,
    id: i32,
) -> Result<CameraItemResponse, AppError> {
    let item = camera_repo::get_by_id(pool, id)
        .await?
        .ok_or_else(|| AppError::not_found(format!("Camera {} not found", id)))?;

    Ok(camera_repo::map_to_response(item))
}
```

**Rules:**
- Services receive and return DTOs (not domain models)
- Business rule checks happen here, not in repositories
- Use `ok_or_else(|| AppError::not_found(...))` for 404 handling
- Services never know about HTTP — only domain logic

---

## 5. Error Handling

```rust
// src/error/app_error.rs
#[derive(Error, Debug)]
pub enum AppError {
    #[error("Bad request: {0}")]
    BadRequest(String),

    #[error("Unauthorized: {0}")]
    Unauthorized(String),

    #[error("Forbidden: {0}")]
    Forbidden(String),

    #[error("Not found: {0}")]
    NotFound(String),

    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("Validation error")]
    Validation(ValidationErrors),

    #[error("Internal server error: {0}")]
    Internal(String),
}

// Convenience constructors
impl AppError {
    pub fn bad_request(msg: impl Into<String>) -> Self { Self::BadRequest(msg.into()) }
    pub fn not_found(msg: impl Into<String>) -> Self { Self::NotFound(msg.into()) }
    pub fn internal(msg: impl Into<String>) -> Self { Self::Internal(msg.into()) }
    pub fn unauthorized(msg: impl Into<String>) -> Self { Self::Unauthorized(msg.into()) }
}
```

**Response format** for errors:
```json
{
  "code": "400",
  "message": "Invalid input",
  "data": {
    "errors": [
      { "field": "name", "message": "Name is required", "code": "required" }
    ]
  }
}
```

**Usage:**
```rust
// Propagate with ?
let item = camera_repo::get_by_id(pool, id).await?;

// Early return
if id <= 0 { return Err(AppError::bad_request("Invalid ID")); }

// NotFound pattern
let item = repo::get(pool, id)
    .await?
    .ok_or_else(|| AppError::not_found(format!("Item {} not found", id)))?;
```

---

## 6. Request / Response DTOs

```rust
// src/models/camera_model.rs
use serde::{Deserialize, Serialize};
use utoipa::ToSchema;
use chrono::{DateTime, Utc};

// Request DTO
#[derive(Debug, Deserialize, ToSchema)]
pub struct CameraCreateRequest {
    pub name: String,
    pub location: Option<String>,
    #[serde(default = "default_true")]
    pub is_active: Option<bool>,
}

fn default_true() -> Option<bool> { Some(true) }

// Update DTO (all fields optional for partial update)
#[derive(Debug, Deserialize, ToSchema)]
pub struct CameraUpdateRequest {
    pub name: Option<String>,
    pub location: Option<String>,
    pub is_active: Option<bool>,
}

// Response DTO (never include sensitive fields like passwords)
#[derive(Debug, Serialize, Deserialize, ToSchema, Clone)]
pub struct CameraItemResponse {
    pub id: i32,
    pub name: String,
    pub location: Option<String>,
    pub is_active: bool,
    pub created_at: Option<DateTime<Utc>>,
}

// Pagination query params
#[derive(Debug, Deserialize, ToSchema)]
pub struct CameraListParams {
    #[serde(default = "default_page")]
    pub page: u32,
    #[serde(default = "default_page_size")]
    pub page_size: u32,
}

fn default_page() -> u32 { 1 }
fn default_page_size() -> u32 { 20 }
```

**Standard response wrapper:**
```rust
// src/models/response.rs
ApiResponse::success(data)                                    // single item
ApiResponse::success_with_pagination(items, page, size, total) // paginated list
```

---

## 7. Auth & Permission

**JWT Claims extractor in handler:**
```rust
pub async fn my_handler(
    AuthClaims(claims): AuthClaims,  // Extracts JWT claims from request
    // ...
) -> Result<...> {
    check_permission(&claims, permissions::CAMERA_READ)?;
    // ...
}
```

**Permission constants** (`src/middleware/permission.rs`):
```rust
pub mod permissions {
    pub const CAMERA_READ:   &str = "camera.read";
    pub const CAMERA_MANAGE: &str = "camera.manage";
    pub const USER_READ:     &str = "user.read";
    pub const USER_MANAGE:   &str = "user.manage";
    // Add new permissions here following "resource.action" pattern
}
```

**Adding a new permission:**
1. Add constant to `permissions` mod
2. Assign it to roles in seed data (`seed_data.yml`) or via admin API
3. Use `check_permission(&claims, permissions::MY_NEW_PERM)?` in handler

**Public routes** (no JWT required):
```rust
// src/api/mod.rs
let public_routes = Router::new()
    .route("/health", get(health::health_check))
    .route("/login", post(api_login::login));
    // Add public routes here — no JWT middleware layer
```

---

## 8. Route Registration

```rust
// src/api/mod.rs
pub fn create_router(state: AppState, jwt_config: Arc<JwtConfig>) -> Router {
    let config = get_config();

    // Public routes (no auth)
    let public_routes = Router::new()
        .route("/health", get(health::health_check))
        .route("/login", post(api_login::login));

    // Protected routes — grouped by domain, each gets JWT middleware
    let camera_routes = Router::new()
        .route("/camera",           get(api_camera::list_camera))
        .route("/camera/create",    post(api_camera::create_camera))
        .route("/camera/update",    put(api_camera::update_camera))
        .route("/camera/delete",    delete(api_camera::delete_camera))
        .layer(middleware::from_fn_with_state(jwt_config.clone(), jwt_auth));

    let protected_routes = Router::new()
        .merge(camera_routes)
        // .merge(other_routes)
        .with_state(state.clone());

    Router::new()
        .merge(public_routes)
        .nest(&config.server.api_prefix, protected_routes)  // e.g. /camera-manager/api
        .layer(CorsLayer::permissive())
        .layer(TimeoutLayer::new(Duration::from_secs(30)))
        .layer(TraceLayer::new_for_http())
        .with_state(state)
}
```

---

## 9. Configuration

```rust
// Access config anywhere (after init in main.rs)
use crate::config::get_config;

let config = get_config();
let port = config.server.port;
let db_url = &config.database.url;
```

**config.example.yml structure:**
```yaml
general:
  rust_env: development           # development | production
  project_name: "MyProject"
  secret_key: "change-me-32chars+"
  api_prefix: "/myproject/api"
  port: 8000

database:
  engine: postgresql
  url: "postgresql://user:pass@localhost:5432/dbname"

jwt:
  secret: "change-me-jwt-secret-32chars+"
  issuer: "myproject"
  expire_hours: 24

cors:
  backend_cors_origins:
    - "http://localhost:3001"

admin:
  username: "admin"
  password: "Admin@123"
  reset_on_startup: false
```

**Environment variable overrides** take priority over YAML values.

---

## 10. Database Migration

Migrations are manual SQL files — no automatic runner.

```sql
-- migrations/migration_camera.sql
CREATE TABLE IF NOT EXISTS camera (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    location    TEXT,
    is_active   BOOLEAN NOT NULL DEFAULT true,
    is_delete   BOOLEAN NOT NULL DEFAULT false,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_camera_is_delete ON camera(is_delete);
```

**Rules:**
- Always include `is_delete BOOLEAN NOT NULL DEFAULT false` for soft deletes
- Always include `created_at` and `updated_at` with `DEFAULT NOW()`
- Add indexes on frequently filtered columns
- Apply migrations manually: `psql $DATABASE_URL -f migrations/migration_xxx.sql`

---

## 11. Logging

```rust
use tracing::{info, warn, error, debug};

// In handlers/services:
info!("Creating camera: name={}", req.name);
warn!("Camera not found: id={}", id);
error!("Failed to connect to DB: {}", e);
debug!("Query params: {:?}", params);
```

---

## 12. Conventions

- **Commit format**: `type(scope): description`
  - Types: `feat`, `fix`, `refactor`, `docs`, `chore`, `perf`, `test`
  - Example: `feat(camera): add bulk delete endpoint`
- **No co-author** in commit messages
- Run `cargo fmt` before committing
- Run `cargo clippy` to catch common mistakes
- File naming: `snake_case` for all Rust files
- Handler file: `api_<domain>.rs`, service: `service_<domain>.rs`, repo: `<domain>_repo.rs`
