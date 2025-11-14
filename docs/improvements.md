# Geni Improvement Roadmap

This document outlines potential improvements for the Geni database migration tool, organized by priority and impact.

## Table of Contents

- [Critical Improvements](#critical-improvements)
- [Major Improvements](#major-improvements)
- [Minor Improvements](#minor-improvements)
- [Quick Wins](#quick-wins)
- [Summary Table](#summary-table)

---

## Critical Improvements

### 1. Fix Error Handling (49 `unwrap()` calls)

**Priority**: 🔴 Critical
**Impact**: High - Prevents application crashes
**Effort**: Medium

#### Problem
The codebase contains 49 `unwrap()` and 2 `expect()` calls that can cause panics instead of graceful error handling.

#### Key Locations
- `src/lib/utils.rs`: 7 unwrap() calls in critical file I/O
  - Line 18: `entry.unwrap()` - no error context
  - Line 21: `file_name().to_str().unwrap()` - OS string conversion
  - Line 29-31: Multiple unwrap() in filename parsing
  - Line 43: `fs::read_to_string(path).unwrap()` - **Can panic on I/O error**

- `src/database_drivers/sqlite.rs`: 5 unwrap() calls
  - Line 27: `split_once("://").unwrap()` - Could fail with invalid URLs
  - Line 32, 43: `path.to_str().unwrap()` - Path encoding issues
  - Line 84: `result.next().await.unwrap()` - DB query panic
  - Line 182: `unwrap_or(None)` - redundant pattern

- `src/lib/migrate.rs`: Line 115
  - `s.into_boxed_str().parse::<i64>().unwrap()` - Migration ID parsing can panic

#### Example Fix
```rust
// Current (fragile):
let timestamp = filename.split_once('_').unwrap().0;
let timestamp = timestamp.parse::<i64>().unwrap();

// Should be:
let timestamp = filename
    .split_once('_')
    .ok_or_else(|| anyhow::anyhow!("Invalid migration filename: {}", filename))?
    .0
    .parse::<i64>()
    .context("Failed to parse migration timestamp")?;
```

#### Implementation Steps
1. Create a comprehensive list of all unwrap() locations
2. Replace with proper error propagation using `?` operator
3. Add meaningful error context using `.context()` from anyhow
4. Add tests for error cases
5. Update error messages to be user-friendly

---

### 2. Increase Test Coverage

**Priority**: 🔴 Critical
**Impact**: High - Improves reliability and maintainability
**Effort**: High

#### Current State
- **12 unit tests** for ~2,000 lines of code (0.6% coverage)
- **360-line integration test file** with multiple database tests
- Most modules have zero unit tests

#### Missing Test Coverage
- `utils.rs` functions (only 5 tests for `should_run_in_transaction`)
- `config.rs` parsing logic
- Database driver implementations
- Error path testing
- Edge cases (malformed migrations, permission issues)

#### Missing Test Types
```
✗ Unit tests for file I/O edge cases
✗ Error path testing (invalid migrations, permission issues)
✗ Concurrent migration attempts
✗ Malformed migration files
✗ Database connection failures
✗ Rollback ordering validation
✗ Transaction boundary testing
✗ Benchmark tests (despite performance claims)
```

#### Implementation Steps
1. Add unit tests for all utility functions
2. Mock database operations for isolated testing
3. Add error case tests for each public function
4. Test edge cases in filename parsing
5. Add property-based tests for migration ordering
6. Target minimum 70% code coverage
7. Consider adding benchmark tests

---

## Major Improvements

### 3. Refactor Database Driver Duplication

**Priority**: 🟡 Major
**Impact**: High - Reduces maintenance burden by 30%
**Effort**: High

#### Problem
PostgreSQL, MySQL, and MariaDB drivers are >95% identical (447-448 lines each = ~1,350 lines of duplicated code).

#### Current State
```rust
// postgres.rs, mysql.rs, and maria.rs all implement nearly identical:
let query = format!("INSERT INTO {} (id) VALUES ($1)", self.migrations_table);
sqlx::query(query.as_str()).bind(id).execute(&mut self.db).await?;
```

#### Proposed Solutions

**Option A: Macro-based Code Generation**
```rust
macro_rules! impl_sqlx_driver {
    ($name:ident, $pool_type:ty, $param:expr) => {
        // Generate common boilerplate
    };
}
```

**Option B: Generic SQL Driver with Trait Specialization**
```rust
trait SqlDialect {
    fn placeholder(&self, n: usize) -> String;
    fn dump_schema_command(&self) -> &str;
}

struct GenericSqlDriver<D: SqlDialect> {
    // Shared implementation
}
```

**Option C: Extract Common Functions**
```rust
mod common {
    async fn insert_migration_id<E: Executor>(
        executor: E,
        table: &str,
        id: i64,
        placeholder: &str
    ) -> Result<()> {
        // Shared logic
    }
}
```

#### Implementation Steps
1. Analyze differences between the three drivers
2. Extract common functionality into shared module
3. Use macros or traits to handle dialect-specific SQL
4. Migrate existing drivers to use shared implementation
5. Ensure all tests pass
6. Update documentation

#### Benefits
- Reduces codebase by ~30%
- Easier to add new database support
- Centralized bug fixes
- Improved maintainability

---

### 4. Fix API Design Issues

**Priority**: 🟡 Major
**Impact**: Medium - Improves developer experience
**Effort**: Medium

#### Issues

**A. Typo in Public API**
- `src/lib/lib.rs:32`: `migate_down()` should be `migrate_down()`
- Breaking change requiring version bump

**B. Parameter Bloat**
Every public function takes 7 parameters:
```rust
pub async fn migrate_database(
    database_url: String,           // 1
    database_token: Option<String>, // 2
    migration_table: String,        // 3
    migration_folder: String,       // 4
    schema_file: String,            // 5
    wait_timeout: Option<usize>,    // 6
    dump_schema: bool,              // 7
) -> anyhow::Result<()>
```

**C. Duplicated Configuration Logic**
- `bin/config.rs` and `lib/config.rs` contain identical code
- No type safety (all strings)

#### Proposed Solution

```rust
#[derive(Debug, Clone)]
pub struct MigrationConfig {
    pub database_url: String,
    pub database_token: Option<String>,
    pub migration_table: String,
    pub migration_folder: PathBuf,
    pub schema_file: PathBuf,
    pub wait_timeout: Option<Duration>,
    pub dump_schema: bool,
}

impl MigrationConfig {
    pub fn from_env() -> Result<Self> { /* ... */ }

    pub fn builder() -> MigrationConfigBuilder { /* ... */ }
}

// Simplified API:
pub async fn migrate_database(config: MigrationConfig) -> anyhow::Result<()>
```

#### Implementation Steps
1. Create `MigrationConfig` struct
2. Implement builder pattern for ergonomic construction
3. Consolidate config parsing logic
4. Update all public functions to accept config struct
5. Add deprecation warnings to old API
6. Update examples and documentation
7. Plan for v2.0.0 release with breaking changes

---

### 5. Add Comprehensive Documentation

**Priority**: 🟡 Major
**Impact**: High - Improves adoption and reduces support burden
**Effort**: Medium

#### Current State
- Good README with examples
- **Zero doc comments on public functions**
- No API documentation site
- Error cases not documented
- Complex logic (transaction handling) unexplained

#### Missing Documentation
```rust
// No documentation:
pub async fn migrate_database(...) -> anyhow::Result<()>

// Should have:
/// Runs pending database migrations.
///
/// # Arguments
/// * `database_url` - Connection string (e.g., "postgres://user:pass@host/db")
/// * `migration_table` - Table name for tracking migrations (default: "schema_migrations")
///
/// # Examples
/// ```no_run
/// use geni::migrate_database;
///
/// #[tokio::main]
/// async fn main() -> anyhow::Result<()> {
///     migrate_database(
///         "postgres://localhost/mydb".to_string(),
///         None,
///         "schema_migrations".to_string(),
///         // ...
///     ).await?;
///     Ok(())
/// }
/// ```
///
/// # Errors
/// Returns error if:
/// - Database connection fails
/// - Migration files are malformed
/// - SQL execution fails
pub async fn migrate_database(...) -> anyhow::Result<()>
```

#### Implementation Steps
1. Add `///` doc comments to all public functions
2. Add examples to each public API
3. Document error conditions
4. Create `docs.rs` documentation site
5. Add CHANGELOG.md
6. Improve inline comments for complex logic
7. Document transaction handling magic comment
8. Add architecture documentation

---

## Minor Improvements

### 6. Add Safety Features

**Priority**: 🟢 Minor
**Impact**: Medium - Prevents accidental data loss
**Effort**: Low

#### Missing Safety Features

**A. Interactive Confirmations**
```rust
// For destructive operations:
pub async fn drop_database(...) -> Result<()> {
    println!("⚠️  WARNING: This will permanently delete the database!");
    println!("Database: {}", database_url);
    print!("Type 'yes' to confirm: ");

    let mut input = String::new();
    std::io::stdin().read_line(&mut input)?;

    if input.trim() != "yes" {
        return Err(anyhow!("Operation cancelled"));
    }

    // Proceed with drop
}
```

**B. Dry-Run Mode**
```rust
pub async fn migrate_database(
    config: MigrationConfig,
    dry_run: bool,
) -> Result<()> {
    if dry_run {
        println!("🔍 Dry run - no changes will be made");
        // Show what would be executed
        return Ok(());
    }
    // Execute migrations
}
```

**C. Migration Validation**
- Basic SQL syntax checking before execution
- Verify migration files are valid UTF-8
- Check for common mistakes (missing semicolons, etc.)

#### Implementation Steps
1. Add `--yes` flag to skip confirmations in CI
2. Add `--dry-run` flag to preview changes
3. Implement basic SQL validation
4. Add warnings for potentially dangerous operations
5. Update documentation

---

### 7. Enhance CI/CD Pipeline

**Priority**: 🟢 Minor
**Impact**: Medium - Catches bugs earlier
**Effort**: Low

#### Current State
- Basic rust.yaml workflow (test + build)
- Docker and Nix workflows
- **No linting**
- **No security audits**
- **No formatting checks**

#### Missing CI Checks
```yaml
# Add to .github/workflows/rust.yaml:

- name: Run clippy
  run: cargo clippy --all-targets --all-features -- -D warnings

- name: Check formatting
  run: cargo fmt --all -- --check

- name: Security audit
  run: |
    cargo install cargo-audit
    cargo audit

- name: Check for outdated dependencies
  run: |
    cargo install cargo-outdated
    cargo outdated --exit-code 1

- name: Run MIRI (undefined behavior detection)
  run: |
    rustup toolchain install nightly --component miri
    cargo +nightly miri test
```

#### Implementation Steps
1. Add clippy step with deny warnings
2. Add rustfmt check
3. Add cargo-audit for security
4. Consider adding cargo-deny for dependency policies
5. Add MIRI for unsafe code checking (if any)
6. Add code coverage reporting (codecov/coveralls)

---

### 8. Performance Optimizations

**Priority**: 🟢 Minor
**Impact**: Low - Reduces binary size and build time
**Effort**: Low

#### Issues

**A. Tokio Dependency Bloat**
```toml
# Current (includes everything):
tokio = { version = "1.43.0", features = ["full"] }

# Better (only what's needed):
tokio = {
    version = "1.43.0",
    features = ["rt-multi-thread", "macros", "time", "sync"]
}
```

**B. No Connection Pooling Configuration**
- Each operation creates new connection
- Could be slow for large migration sets
- Consider exposing pool configuration

**C. Potential Regex Compilation Overhead**
- If regex is compiled per-migration check
- Use `lazy_static` or `once_cell` for regex compilation

#### Implementation Steps
1. Audit tokio feature usage
2. Remove unnecessary features
3. Measure binary size before/after
4. Add connection pooling options
5. Optimize regex compilation if applicable
6. Add benchmark tests

---

### 9. Configuration Enhancements

**Priority**: 🟢 Minor
**Impact**: Medium - Improves flexibility
**Effort**: Low

#### Missing Features

**A. Config File Support**
```toml
# .geni.toml
database_url = "postgres://localhost/mydb"
migration_table = "schema_migrations"
migration_folder = "./migrations"
schema_file = "./schema.sql"
wait_timeout = 30
dump_schema = true
```

**B. Better Transaction Control**
Current approach is fragile:
```sql
-- transaction:no
-- This relies on parsing comments
```

Better approach:
```toml
# In migration metadata or separate file:
[migrations."20230101000000_migration"]
transaction = false
```

**C. Migration Metadata**
```toml
# migrations/20230101000000_big_migration.toml
transaction = false
timeout = 600  # seconds
description = "Major schema refactor"
dependencies = ["20221231000000_previous"]
```

#### Implementation Steps
1. Add config file parsing (TOML/YAML)
2. Improve transaction control mechanism
3. Add optional migration metadata files
4. Maintain backward compatibility
5. Update documentation

---

### 10. Improve Error Messages

**Priority**: 🟢 Minor
**Impact**: Medium - Better user experience
**Effort**: Low

#### Current Issues
```rust
// Generic error:
"Couldn't read migration folder"

// Should include context:
"Couldn't read migration folder './migrations': No such file or directory"
```

#### Better Error Messages
```rust
use anyhow::Context;

std::fs::read_dir(&migration_folder)
    .context(format!(
        "Failed to read migration folder '{}'. Ensure the path exists and you have read permissions.",
        migration_folder
    ))?;
```

#### Implementation Steps
1. Audit all error messages
2. Add file paths, URLs, and operation details
3. Suggest fixes when possible
4. Use anyhow's `.context()` consistently
5. Add error codes for documentation reference

---

### 11. Migration Logic Improvements

**Priority**: 🟢 Minor
**Impact**: Medium - Prevents bugs
**Effort**: Low

#### Issues

**A. Rollback Ordering Bug**
```rust
// Line 118-120 in migrate.rs:
let migrations_to_run = migrations.into_iter().take(*rollback_amount as usize);

// This takes from START of vector, but should take from END for rollbacks
// Needs explicit .rev() or proper ordering
```

**B. Fragile Filename Parsing**
```rust
// Line 30 in utils.rs
let timestamp = filename.split_once('_').unwrap().0;

// Breaks if timestamp contains underscore
// Should use more robust parsing
```

**C. Silent Schema Dump Failures**
```rust
if let Err(err) = database.dump_database_schema().await {
    log::error!("Skipping dumping database schema: {:?}", err);
    // Continues silently - should at least warn user
}
```

#### Implementation Steps
1. Fix rollback ordering logic
2. Improve filename parsing robustness
3. Make schema dump failures more visible
4. Add validation for migration filenames
5. Add tests for edge cases

---

### 12. Add CHANGELOG.md

**Priority**: 🟢 Minor
**Impact**: Low - Improves transparency
**Effort**: Low

#### Problem
Currently no changelog to track version history. Users don't know what changed between versions.

#### Proposed Format
```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- New feature X
- Support for Y

### Changed
- Improved performance of Z

### Fixed
- Bug in migration rollback

## [1.1.6] - 2024-XX-XX

### Changed
- Bumped clap from 4.5.27 to 4.5.38
- Bumped which from 7.0.0 to 7.0.2
...
```

#### Implementation Steps
1. Create CHANGELOG.md
2. Document current version
3. Add unreleased section for ongoing work
4. Update with each release
5. Consider automated changelog generation

---

## Quick Wins

These improvements are easy to implement and provide immediate value:

| Improvement | Effort | Impact | Files |
|-------------|--------|--------|-------|
| Fix `migate_down()` typo | 5 min | Medium | `src/lib/lib.rs:32` |
| Add clippy to CI | 10 min | High | `.github/workflows/rust.yaml` |
| Add rustfmt check to CI | 5 min | Medium | `.github/workflows/rust.yaml` |
| Consolidate duplicate config code | 30 min | Medium | `bin/config.rs`, `lib/config.rs` |
| Optimize tokio features | 10 min | Low | `Cargo.toml` |
| Add doc comments to public API | 2 hours | High | All `src/lib/*.rs` files |
| Add interactive confirmation for `drop` | 30 min | High | `src/lib/management.rs` |
| Create CHANGELOG.md | 30 min | Low | Root directory |
| Fix rollback ordering bug | 20 min | High | `src/lib/migrate.rs:118-120` |
| Improve error messages | 1 hour | Medium | Multiple files |

---

## Summary Table

| Category | Priority | Impact | Effort | Est. Time |
|----------|----------|--------|--------|-----------|
| Error Handling | 🔴 Critical | High | Medium | 1-2 weeks |
| Test Coverage | 🔴 Critical | High | High | 2-3 weeks |
| Driver Refactoring | 🟡 Major | High | High | 2-3 weeks |
| API Design | 🟡 Major | Medium | Medium | 1 week |
| Documentation | 🟡 Major | High | Medium | 1 week |
| Safety Features | 🟢 Minor | Medium | Low | 2-3 days |
| CI/CD Enhancements | 🟢 Minor | Medium | Low | 1-2 days |
| Performance | 🟢 Minor | Low | Low | 1-2 days |
| Configuration | 🟢 Minor | Medium | Low | 3-4 days |
| Error Messages | 🟢 Minor | Medium | Low | 1-2 days |
| Migration Logic | 🟢 Minor | Medium | Low | 1-2 days |
| CHANGELOG | 🟢 Minor | Low | Low | 1 hour |

---

## Implementation Strategy

### Phase 1: Foundation (Weeks 1-2)
Focus on critical stability improvements:
- [ ] Fix all 49 unwrap() calls with proper error handling
- [ ] Fix typo in public API (`migate_down` → `migrate_down`)
- [ ] Add clippy and rustfmt to CI
- [ ] Fix rollback ordering bug

### Phase 2: Quality (Weeks 3-5)
Improve code quality and testing:
- [ ] Increase test coverage to 70%+
- [ ] Add comprehensive documentation
- [ ] Consolidate duplicate config code
- [ ] Improve error messages

### Phase 3: Refactoring (Weeks 6-8)
Major architectural improvements:
- [ ] Refactor database drivers to reduce duplication
- [ ] Redesign API with config struct
- [ ] Add config file support

### Phase 4: Enhancement (Weeks 9-10)
Add new features and polish:
- [ ] Add safety features (confirmations, dry-run)
- [ ] Performance optimizations
- [ ] Migration logic improvements
- [ ] CHANGELOG.md

---

## Version Planning

### v1.2.0 (Patch Release)
- Quick wins only
- No breaking changes
- Fix typo via deprecation

### v2.0.0 (Major Release)
- Breaking API changes
- New config struct
- Refactored drivers
- Comprehensive documentation

---

## Contributing

When implementing these improvements:

1. **Create issues** for each improvement on GitHub
2. **Reference this document** in issue descriptions
3. **Submit focused PRs** addressing single improvements
4. **Add tests** for all new functionality
5. **Update documentation** alongside code changes
6. **Follow conventional commits** for changelog generation

---

## References

- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
- [Anyhow Error Handling](https://docs.rs/anyhow/)
- [Clippy Lints](https://rust-lang.github.io/rust-clippy/master/)
- [Keep a Changelog](https://keepachangelog.com/)
- [Semantic Versioning](https://semver.org/)

---

**Last Updated**: 2025-11-14
**Status**: Draft
**Maintainer**: See CODEOWNERS
