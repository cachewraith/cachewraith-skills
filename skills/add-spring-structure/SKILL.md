---
name: add-spring-structure
description: "Lay out a Java Spring Boot project as a versioned, module-per-feature package tree (config, common, security, modules/<feature>/{controller,service,repository,entity,dto,mapper,validator}, integration, scheduler) and record that layout plus its class-naming manifest in CLAUDE.md so every later file lands in the right package under the right name. Invoke when the user asks to add, apply, or scaffold the Spring structure, set up a new Spring Boot project's packages, or add a new feature module. Skip for non-Spring projects and for renaming or moving a single existing class."
---

# Add Spring Structure

Create the package tree, then write the layout contract into `CLAUDE.md` so it survives
the session. Directories are cheap; write class bodies only when the work needs them.

## 1. Read the project

```bash
ls pom.xml build.gradle* 2>/dev/null; find src/main/java -name '*Application.java'
```

The `*Application.java` path gives the base package — use it, never `com.company.project`.
No Spring Boot project here? Say so and stop.

Ask only what you cannot infer: which feature modules (`user`, `auth`, `order`, …) and
which API versions (default `v1`). Existing modules on disk answer the first question.

## 2. Create the tree

```bash
B=src/main/java/com/company/project          # the real base package path
mkdir -p $B/{config,scheduler} \
  $B/common/{response,exception,base,util,constant,annotation} \
  $B/security/{jwt,userdetails,handler} \
  $B/integration/{payment,email,storage} \
  src/main/resources/db/migration src/main/resources/static
```

Per feature module, per version:

```bash
M=user; V=v1
mkdir -p $B/modules/$M/{controller/$V,service/impl,repository,entity,mapper/$V,validator} \
  $B/modules/$M/dto/$V/{request,response}
```

Add a second version by re-running with `V=v2` — it only adds `controller/v2`,
`dto/v2/{request,response}`, `mapper/v2`.

Drop a `.gitkeep` in any directory you leave empty, or git will not track it.

Resources: `application.yml` plus `application-dev.yml` / `application-prod.yml`, and
`db/migration/V1__init.sql` if the project uses Flyway. Create only what is missing.

## 3. Write the contract into CLAUDE.md

This is the part that makes the structure stick. Replace the managed block in place so
re-running never duplicates it, and never touch the rest of the file:

```bash
F=CLAUDE.md
[ -f $F ] && sed -i '/<!-- spring-structure:start -->/,/<!-- spring-structure:end -->/d' $F
cat >> $F <<'MD'
<!-- spring-structure:start -->
## Project structure (Spring Boot)

Base package `com.company.project`, entry point `ProjectApplication`. Every new file goes
at its address below, under the name below; do not invent sibling packages.

| Package | Classes |
|---|---|
| `config` | `SecurityConfig`, `SwaggerConfig`, `WebConfig`, `RedisConfig`, `AsyncConfig`, `ModelMapperConfig` |
| `common.response` | `ApiResponse`, `ApiErrorResponse`, `PageResponse` |
| `common.exception` | `GlobalExceptionHandler`, `BusinessException`, `ResourceNotFoundException`, `ValidationException`, `ErrorCode` |
| `common.base` | `BaseEntity`, `BaseRepository`, `BaseService`, `BaseController` |
| `common.util` | `DateUtil`, `StringUtil`, `FileUtil` |
| `common.constant` | `AppConstants`, `ApiVersions` |
| `common.annotation` | `CurrentUser`, `RateLimited` |
| `security.jwt` | `JwtProvider`, `JwtFilter`, `JwtProperties` |
| `security.userdetails` | `CustomUserDetailsService` |
| `security.handler` | `AccessDeniedHandlerImpl`, `AuthEntryPointImpl` |
| `integration.<provider>` | third-party clients — `payment`, `email`, `storage` |
| `scheduler` | one class per job — `CleanupScheduler` |

Inside `modules.<feature>`, for feature `User`:

| Package | Classes |
|---|---|
| `controller.v<N>` | `UserController` — same simple name in every version; the package disambiguates |
| `service` | `UserService` (interface) |
| `service.impl` | `UserServiceImpl` |
| `repository` | `UserRepository` |
| `entity` | `User` |
| `dto.v<N>.request` | `UserCreateRequest`, `UserUpdateRequest` |
| `dto.v<N>.response` | `UserResponse` |
| `mapper.v<N>` | `UserMapper` |
| `validator` | `UserValidator` |

Rules
- Controllers, DTOs and mappers are versioned (`v1`, `v2`); services, repositories and
  entities are not — one service backs every version.
- A module carries only the packages it needs: `auth` is controller + service + dto, no
  entity or repository of its own.
- Controllers accept and return DTOs only. Entities never cross the controller boundary.
- Endpoints return `ApiResponse` / `PageResponse`; failures throw a `BusinessException`
  subclass carrying an `ErrorCode` and surface through `GlobalExceptionHandler`.
- Cross-feature access is service → service, never into another feature's repository.
- New feature → new `modules/<name>` with the full layout. New breaking API shape → new
  `v<N>` of controller + dto + mapper only.
<!-- spring-structure:end -->
MD
```

Substitute the real base package and application class before writing. If `CLAUDE.md`
does not exist, this creates it; if it does, the block lands at the end and the user's own
sections are untouched.

## 4. Existing projects

Do not mass-move classes. Create the tree, write the contract, and report which existing
packages already fit and which are off-layout — then let the user pick what to migrate.
Moving a package means updating its `package`/`import` lines and any `@ComponentScan`,
so treat each move as its own change.

The manifest names the classes the layout expects, not classes that must exist today.
Create one when the work needs it; do not generate empty stubs for the whole table.

## 5. Report

List what you created (modules and versions), whether `CLAUDE.md` was created or updated,
and anything you deliberately left empty. Then stop — no build, no commit.
