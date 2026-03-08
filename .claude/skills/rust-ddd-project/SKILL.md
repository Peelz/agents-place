---
name: rust-ddd-project
description: |
  Domain-Driven Design project structure organizer for Rust projects.
  Creates modular, self-contained domains with shared common components.
  Use when creating new Rust projects, restructuring existing projects to DDD,
  adding domain modules, or setting up shared models/utilities.
  Trigger terms: DDD, domain-driven design, project structure, domain module,
  bounded context, module organization, restructure project.
---

# Rust Domain-Driven Design Project Practicle

Domain-Driven Design project structure organizer for Rust projects. Creates modular, self-contained domains with shared common components.

## When to Use

- Creating a new Rust project with DDD architecture
- Restructuring an existing project to follow DDD principles
- Adding a new domain module to a DDD project
- Setting up shared models or utilities across multiple domains
- Deciding whether something belongs in `common/` or a domain module

## Do Not Use When

- The project is a small CLI tool or library with a single concern
- The project has fewer than 2 distinct domains (flat structure is simpler)
- Working on non-Rust projects (this skill is Rust-specific)

## Instructions

When this skill is invoked, follow these steps:

1. **Understand the project scope**: Ask the user what domains/bounded contexts exist or are planned
2. **Identify shared concerns**: Determine which models, error types, or utilities are truly cross-domain
3. **Generate the structure**: Create directories and files following the structure below
4. **Wire up modules**: Update `main.rs` and `mod.rs` files to connect everything
5. **Apply dependency rules**: Validate that cross-domain communication follows the patterns in this document

## Structure

```
src/
├── common/
│   ├── mod.rs               # Re-exports shared components
│   ├── models/              # Shared data models
│   │   ├── mod.rs
│   │   ├── users.rs
│   │   └── ...
│   ├── errors.rs            # Shared error types
│   ├── http_util.rs         # Specific utility modules (no generic utils.rs)
│   ├── pubsub.rs            # Shared pubsub layer (if used across domains)
│   └── cache.rs             # Shared cache layer (if used across domains)
├── users/                   # Domain module
│   ├── mod.rs               # Module exports
│   ├── services.rs          # Business logic
│   ├── repository.rs        # Data access layer (trait + implementation)
│   ├── models.rs            # Domain-specific models
│   ├── routes.rs            # API endpoints
│   └── devices/             # Sub-module for nested resources
│       ├── mod.rs
│       ├── services.rs
│       ├── repository.rs
│       └── routes.rs        # Handles /users/{id}/devices
├── orders/                  # Another domain module
│   ├── mod.rs
│   ├── services.rs
│   ├── repository.rs
│   ├── models.rs
│   └── routes.rs
├── lib.rs                   # Module declarations and app-wide setup
└── main.rs                  # Entry point, minimal wiring
```

## Code Templates

### Domain `mod.rs`

Each domain module re-exports its public interface. Internal types stay private.

```rust
pub mod models;
pub mod repository;
pub mod routes;
pub mod services;

// Re-export key types for ergonomic access: `use crate::users::UserService`
pub use models::User;
pub use services::UserService;
```

### Domain `models.rs`

Domain models own their data. Shared models go in `common/models/`.

```rust
use serde::{Deserialize, Serialize};

/// Domain-specific model. Only this domain should create or mutate it.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct User {
    pub id: i64,
    pub email: String,
    pub name: String,
}

/// Input type for creating a new user. Keeps creation separate from the full model.
#[derive(Debug, Deserialize)]
pub struct CreateUser {
    pub email: String,
    pub name: String,
}
```

### Domain `repository.rs`

The repository is a trait-based abstraction over data access. This keeps services testable and decoupled from the storage backend.

```rust
use super::models::{CreateUser, User};

/// Data access trait. Implement for Postgres, SQLite, in-memory, etc.
#[async_trait::async_trait]
pub trait UserRepository: Send + Sync {
    async fn find_by_id(&self, id: i64) -> Result<Option<User>, anyhow::Error>;
    async fn find_all(&self) -> Result<Vec<User>, anyhow::Error>;
    async fn create(&self, input: CreateUser) -> Result<User, anyhow::Error>;
    async fn delete(&self, id: i64) -> Result<bool, anyhow::Error>;
}
```

### Domain `services.rs`

Services contain business logic. They depend on repository traits, not concrete implementations.

```rust
use super::models::{CreateUser, User};
use super::repository::UserRepository;
use std::sync::Arc;

pub struct UserService {
    repo: Arc<dyn UserRepository>,
}

impl UserService {
    pub fn new(repo: Arc<dyn UserRepository>) -> Self {
        Self { repo }
    }

    pub async fn get_user(&self, id: i64) -> Result<Option<User>, anyhow::Error> {
        self.repo.find_by_id(id).await
    }

    pub async fn create_user(&self, input: CreateUser) -> Result<User, anyhow::Error> {
        // Business logic / validation goes here
        self.repo.create(input).await
    }
}
```

### Domain `routes.rs`

Routes are thin. They parse HTTP input, call the service, and return HTTP output.

```rust
use super::services::UserService;
use std::sync::Arc;

/// Register this domain's routes with your router.
/// The exact signature depends on your web framework.
pub fn register_routes(service: Arc<UserService>) {
    // Framework-specific route registration:
    //   GET  /users      -> service.list_users()
    //   GET  /users/{id} -> service.get_user(id)
    //   POST /users      -> service.create_user(input)
    todo!("Wire routes to your framework's router")
}
```

### `common/mod.rs`

```rust
pub mod errors;
pub mod models;

// Only add modules here that are genuinely used by 2+ domains.
// pub mod http_util;
// pub mod cache;
```

### `lib.rs`

```rust
pub mod common;
pub mod users;
pub mod orders;
// Declare each domain module here
```

### `main.rs`

```rust
use std::sync::Arc;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // 1. Initialize shared infrastructure (DB pool, cache, etc.)
    // 2. Create repositories
    // 3. Create services, injecting repositories
    // 4. Register routes from each domain
    // 5. Start the server

    Ok(())
}
```

## Cross-Domain Communication

### Dependency Rules

Domains must not reach directly into each other's repositories or models. All cross-domain communication goes through **services**.

```
 ALLOWED                          FORBIDDEN
 ────────                         ──────────
 orders::services                 orders::services
   → users::UserService             → users::repository::UserRepository  (bypasses business logic)
   → common::models::*              → users::models::User directly for mutation
```

| Pattern | Allowed | Why |
|---------|---------|-----|
| Domain A calls Domain B's **service** | Yes | Services enforce business rules |
| Domain A reads Domain B's **repository** directly | No | Bypasses validation, creates hidden coupling |
| Domain A uses `common/models` | Yes | Shared types are designed for cross-domain use |
| Domain A imports Domain B's **models** for read-only use | Acceptable | Sometimes needed for return types, but prefer shared models in `common/` if frequent |
| Domain A mutates Domain B's data directly | No | Always go through the owning domain's service |

### Cross-Domain Service Injection

When a service in one domain needs another domain's service, inject it via constructor:

```rust
// orders/services.rs
use crate::users::UserService;

pub struct OrderService {
    repo: Arc<dyn OrderRepository>,
    user_service: Arc<UserService>,  // Injected, not constructed here
}

impl OrderService {
    pub fn new(
        repo: Arc<dyn OrderRepository>,
        user_service: Arc<UserService>,
    ) -> Self {
        Self { repo, user_service }
    }

    pub async fn create_order(&self, user_id: i64) -> Result<Order, anyhow::Error> {
        // Validate user exists through the user service, not the user repo
        let user = self.user_service
            .get_user(user_id)
            .await?
            .ok_or_else(|| anyhow::anyhow!("user not found"))?;

        // ... create order
        todo!()
    }
}
```

### Event-Based Decoupling (Optional)

For domains that should not have direct dependencies, use events via a shared pubsub layer in `common/`:

```rust
// common/events.rs
#[derive(Debug, Clone)]
pub enum DomainEvent {
    UserCreated { user_id: i64 },
    OrderCompleted { order_id: i64, user_id: i64 },
}
```

Use this pattern when:

- Two domains react to each other's changes but shouldn't call each other directly
- You want to avoid circular dependencies between domain services
- The reaction is asynchronous or can be eventually consistent

## Decision Framework

### Domain vs. Sub-module vs. Common

```
Is this concept a top-level business capability with its own lifecycle?
  ├── Yes → Create a top-level domain module (e.g., users/, orders/, billing/)
  └── No
       ├── Does it belong to a parent resource? (e.g., devices belong to users)
       │    ├── Yes → Create a sub-module under the parent (users/devices/)
       │    └── No
       │         ├── Is it used by 2+ domains?
       │         │    ├── Yes → Place in common/
       │         │    └── No → Place in the single domain that uses it
       │         └── Is it infrastructure? (DB, cache, HTTP client)
       │              ├── Shared across domains → common/
       │              └── Single domain only → within that domain
```

### Sizing Heuristics

| Signal | Action |
|--------|--------|
| A domain's `services.rs` exceeds ~300 lines | Split into multiple service files or extract a sub-module |
| A domain has 5+ model types | Consider if some are actually a separate bounded context |
| `common/` has domain-specific logic | Move it into the domain that owns it |
| Two domains always change together | They may be one domain, or need clearer boundaries |
| A sub-module has its own models + complex business rules | Consider promoting to a top-level domain |
| You need circular dependencies between domains | Introduce an event system or extract shared logic to `common/` |

## Complex Resources & Sub-modules

When a resource belongs to another resource (e.g., `devices` belongs to `users`), create a **sub-module** inside the parent domain.

**Example**: For nested routes like `/users/{id}/devices`, create:

```
users/
├── mod.rs
├── services.rs
├── repository.rs
├── models.rs
├── routes.rs
└── devices/
    ├── mod.rs
    ├── services.rs
    ├── repository.rs
    └── routes.rs
```

**Nesting Depth Guidelines:**

| Depth | Recommendation | Example |
|-------|----------------|---------|
| 1-2 levels | Keep in same module (optional sub-module) | `/users/{id}/devices` |
| 3+ levels | Create sub-module | `/users/{id}/devices/{device_id}/settings` |

**When to create sub-modules:**

- Parent-child relationships (e.g., `users -> devices`)
- Deep nesting (3+ levels)
- Complex business logic for the nested resource
- Shared lifecycle with parent (e.g., deleting user deletes their devices)

**Keep in same module when:**

- 1-2 levels of nesting with simple CRUD operations
- Resource can exist independently
- Only a few routes and minimal business logic

> **Note**: Sub-modules can use common components from their parent module. For example, `users/devices` can access shared types, utilities, or services defined in the `users` module.

## Key Principles

### Core Architecture

| Principle | Description |
|-----------|-------------|
| **Module Independence** | Each domain module should be self-contained with minimal dependencies on other modules |
| **Shared Components** | Place truly shared code in `common/` directory; domain-specific code stays in its module |
| **Clear Boundaries** | Define interfaces between domains using services, never direct repository access |
| **Domain Models** | Keep domain-specific models within their domain; share only through `common/models` or well-defined APIs |
| **Dependency Direction** | Domains depend on `common/`, never on each other's internals. Cross-domain calls go through services only. |

### File Naming

| Rule | Correct | Incorrect |
|------|---------|-----------|
| **No Generic Utils** | `http_util.rs`, `serde_util.rs`, `string_util.rs` | `utils.rs`, `util.rs` |
| **Layer Representation** | `pubsub.rs`, `cache.rs`, `queue.rs` | `infrastructure.rs` (too broad) |
| **Singular Nouns for Modules** | `user/`, `order/` or `users/`, `orders/` | Mixed conventions in the same project |

### Layer Placement

- **Shared layers** (used across multiple domains) -> place in `common/`
- **Domain-specific layers** -> place within the domain module
- **Sub-modules** -> can use common components from their parent module

## Anti-patterns

| Anti-pattern | Problem | Fix |
|-------------|---------|-----|
| `utils.rs` or `helpers.rs` | Becomes a dumping ground for unrelated code | Name by purpose: `http_util.rs`, `date_util.rs` |
| Domain A imports Domain B's repository | Bypasses business logic, creates tight coupling | Domain A calls Domain B's service instead |
| Giant `common/` module | Common becomes a second monolith | Only move code to common when used by 2+ domains |
| All models in `common/models/` | Domains lose ownership of their data | Keep domain models in the domain; share only cross-domain types |
| Circular domain dependencies | `users` depends on `orders` depends on `users` | Use events or extract shared logic to `common/` |
| Business logic in `routes.rs` | Routes become untestable, coupled to HTTP | Routes should only parse input, call service, return output |
| Business logic in `repository.rs` | Data access layer does validation/transformation | Repository does CRUD only; business rules live in `services.rs` |

## Usage

Ask me to:

- "Create a new DDD project structure for [project-name]"
- "Add a [domain-name] module following DDD structure"
- "Restructure my project to follow DDD principles"
- "Should [concept] be a domain, sub-module, or go in common?"
- "Set up cross-domain communication between [domain-a] and [domain-b]"
