# This File

This file is the **standing prompt** — the global CLAUDE.md that governs Claude's behavior across all projects. It lives at `~/repos/claude-md/CLAUDE.md` and is symlinked to `~/.claude/CLAUDE.md`.

When asked to "modify" or "add to" the standing prompt, Claude should:
1. Edit this file (`~/repos/claude-md/CLAUDE.md`) directly.
2. Stage the change and prepare a commit (gitmoji style, co-author footer).
3. Show the proposed diff and commit message, then ask for confirmation before pushing.

# Interaction Style

- Be concise and factual.
- Skip filler phrases and value judgements ("Great idea!", "You're absolutely right!", "Certainly!", "Of course!", etc.). State facts and reasoning; let the user decide if something is a good idea.
- Lead with the answer or action.

# Code Style

- Prefer functional or object-functional programming style regardless of language (e.g., stream APIs over for loops in Java, map/filter/reduce over imperative loops, immutable data where practical).
- All public members must have a docstring.
- Do not implement `Default` unless clippy's `new_without_default` lint requires it, or there is an existing caller or bound that requires it. Do not add it speculatively.

# Database Query Practices

- No unbounded queries. Every query that can return more than one record must include a `limit` parameter and a pagination mechanism.
- Default to [keyset pagination](https://www.merge.dev/blog/keyset-pagination) for performance and scalability. Offset-based pagination may be acceptable when product requirements (e.g., random-access page jumps) make keyset impractical.

# Code Architecture

Follow [Domain-Driven Design](https://en.wikipedia.org/wiki/Domain-driven_design) and [Hexagonal Architecture](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)) principles.

## Module Structure

**`core`**
- Models business rules and domain objects.
- Defines repository/port interfaces that specify available database operations and transaction boundaries, with no reference to specific database technology.
- No I/O or side effects; pure domain logic only.
- Depends on no other internal modules.

**`db`** (one or more)
- Implements the repository interfaces defined in `core`.
- Contains all database-specific code.
- Depends only on `core`.

**`api`** (optional)
- Serialization/deserialization of domain objects (JSON, protobuf, etc.).
- Depends only on `core`.

**`service`**
- Wires up all other modules.
- Provides the external interface (typically HTTP).
- Hosts logging, metrics, and other telemetry.
- Depends on all other internal modules.

## Port Conventions

Within `core`/`domain`, data access ports follow a split read/write pattern:

- **`XxxProvider`** — read port (`&self`); implemented in `db`. Concurrent reads allowed; no `&mut self`.
- **`XxxPersistor`** — write port (`&self`); implemented in `db`. Concurrent writes allowed; interior mutability (connection pool, mutex, etc.) is the implementor's concern.
- **`XxxReader`** / **`XxxWriter`** — service-layer wrappers in `core`/`domain` that hold a generic `P: XxxProvider` / `P: XxxPersistor` and expose domain-facing methods. Prefer generic type parameters over `Box<dyn ...>` unless runtime type erasure is genuinely needed.
- **`XxxRepo`** — optional combined port (Provider + Persistor) for contexts where CQRS separation is not needed.
- **`XxxStore`** — combined Reader + Writer for contexts where separating reads and writes is not needed.

## Dependency Diagram

```
┌──────────────────────────────────────────────────┐
│                     service                      │
│   (wiring, HTTP, logging, metrics/telemetry)     │
└────────┬─────────────────┬──────────┬────────────┘
         │                 │          │
         ▼                 ▼          ▼
  ┌─────────────┐      ┌──────┐   ┌─────┐
  │ db          │      │  api │   │ ... │
  │             │      └──┬───┘   └──┬──┘
  │ implements  │         │          │
  │ XxxRepo /   │         │          │
  │ XxxProvider │         │          │
  │ XxxPersistor│         │          │
  └──────┬──────┘         │          │
         │                │          │
         └────────────────▼──────────┘
                    ┌──────────────────────┐
                    │ core                 │
                    │                      │
                    │ - domain objects     │
                    │ - business rules     │
                    │ - XxxProvider /      │
                    │   XxxPersistor /     │
                    │   XxxRepo ports      │
                    │ - XxxReader /        │
                    │   XxxWriter /        │
                    │   XxxStore wrappers  │
                    └──────────────────────┘
```

All arrows point toward `core`. `core` has no outward dependencies on internal modules.

# Essential vs. Accidental Complexity

Based on Moseley & Marks, ["Out of the Tar Pit"](https://curtclifton.net/papers/MoseleyMarks06a.pdf) (2006).

**Essential complexity** is inherent in the users' problem. If a user doesn't know what it is, it isn't essential.
**Accidental complexity** is everything else — complexity introduced by implementation choices, tools, or language.

The two governing design objectives are:
- **Avoid** — eliminate state and control wherever they are not strictly essential.
- **Separate** — when complexity cannot be avoided, isolate it from essential components.

## State

**Essential state** is input data the system must retain because the users' requirements say so. It is the only legitimate source of mutable state.

**Accidental state** is everything else:
- Derived data (can always be re-derived from essential state; do not store it)
- Caches and memoization results
- Data not present in the users' requirements at all

Prefer re-deriving over caching. Only retain derived data as an explicit performance optimization, declared separately from the logic that computes it.

## Control

All control — sequencing, branching, loop ordering, explicit concurrency — is accidental unless the users' requirements specifically demand it. Control is an implementation concern, not a problem-domain concern.

- Prefer declarative specifications (what, not how).
- Do not introduce ordering constraints beyond what correctness requires.
- Treat concurrency as accidental unless the requirements mention it.

# Testing Style

- Prefer asserting on whole objects over individual fields. Construct the expected value as a complete struct literal and use a single `assert_eq!` rather than multiple assertions on separate fields.
- Only assert on individual fields or accessor return values when a meaningful whole-object expected value cannot be expressed (e.g. types with private fields that lack a struct-literal form).

# Preferred Tools

- Every project has a unit test suite, a static linter, and a pretty printer.
- Favour low- or no-configuration tools (e.g., Prettier for JavaScript, Spotless for Java, Black for Python, gofmt for Go).
- Every project has a `Makefile` with at minimum:
  - `p` — run the pretty printer
  - `l` — run the linter
  - `t` — run the unit tests
- Before committing: run the pretty printer, then the linter, then the unit tests, in that order.
  - Scope the test run to relevant tests when possible. If running the full suite takes ~30 seconds or less, run it all.
  - The Makefile aliases can be used, but it's fine to invoke tools directly (e.g., to pass flags that restrict the test scope).

# Language-Specific Notes

## Rust

- Use `rustfmt` for formatting and `clippy` for linting.
- Name the core crate `domain` (not `core` — conflicts with Rust's standard library crate).
- `lib.rs` contains only module declarations (`pub mod foo;`). Actual code lives in dedicated module files.
- File naming within a module follows entity-based grouping:
  - `{entity}.rs` — the domain entity and closely related types
  - `{entity}_repo.rs` — the repo port definitions (`XxxProvider`, `XxxPersistor`, `XxxRepo`) for that entity
  - `{entity}_store.rs` — the store/service-layer wrapper (`XxxStore`) for that entity
- Makefile should include these additional aliases beyond the standard `p`/`l`/`t`:
  - `c check` — `cargo check --all`
  - `b build` — `cargo build --all`
  - `r run` — `cargo run`
  - `br build-release` — `cargo build --release --all`
- Pre-commit hook runs in this order: `cargo check` → `cargo fmt --check` → `cargo clippy`. Do **not** run tests in the pre-commit hook.

# Git Workflow

- Only create commits when explicitly asked. Do not commit proactively after completing a task.
- When initializing a new repo, make a skeleton initial commit with this exact message: `🌚 Creatio ex nihilo`

# Git Commit Style

Commit messages follow the [gitmoji](https://gitmoji.dev/) convention.

**Format:**
- Title: `<gitmoji> <subject>` — max 72 characters total, including the emoji and trailing space
- Body: additional context as needed
- Footer: co-author line

**Co-author footer:**
```
Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

**Gitmoji reference:**

| Emoji | Code | Description |
|-------|------|-------------|
| 🎨 | `:art:` | Improve structure / format of the code |
| ⚡️ | `:zap:` | Improve performance |
| 🔥 | `:fire:` | Remove code or files |
| 🐛 | `:bug:` | Fix a bug |
| 🚑️ | `:ambulance:` | Critical hotfix |
| ✨ | `:sparkles:` | Introduce new features |
| 📝 | `:memo:` | Add or update documentation |
| 🚀 | `:rocket:` | Deploy stuff |
| 💄 | `:lipstick:` | Add or update the UI and style files |
| 🎉 | `:tada:` | Begin a project |
| ✅ | `:white_check_mark:` | Add, update, or pass tests |
| 🔒️ | `:lock:` | Fix security or privacy issues |
| 🔐 | `:closed_lock_with_key:` | Add or update secrets |
| 🔖 | `:bookmark:` | Release / Version tags |
| 🚨 | `:rotating_light:` | Fix compiler / linter warnings |
| 🚧 | `:construction:` | Work in progress |
| 💚 | `:green_heart:` | Fix CI build |
| ⬇️ | `:arrow_down:` | Downgrade dependencies |
| ⬆️ | `:arrow_up:` | Upgrade dependencies |
| 📌 | `:pushpin:` | Pin dependencies to specific versions |
| 👷 | `:construction_worker:` | Add or update CI build system |
| 📈 | `:chart_with_upwards_trend:` | Add or update analytics or tracking code |
| ♻️ | `:recycle:` | Refactor code |
| ➕ | `:heavy_plus_sign:` | Add a dependency |
| ➖ | `:heavy_minus_sign:` | Remove a dependency |
| 🔧 | `:wrench:` | Add or update configuration files |
| 🔨 | `:hammer:` | Add or update development scripts |
| 🌐 | `:globe_with_meridians:` | Internationalization and localization |
| ✏️ | `:pencil2:` | Fix typos |
| 💩 | `:poop:` | Write bad code that needs improvement |
| ⏪️ | `:rewind:` | Revert changes |
| 🔀 | `:twisted_rightwards_arrows:` | Merge branches |
| 📦️ | `:package:` | Add or update compiled files or packages |
| 👽️ | `:alien:` | Update code due to external API changes |
| 🚚 | `:truck:` | Move or rename resources |
| 📄 | `:page_facing_up:` | Add or update license |
| 💥 | `:boom:` | Introduce breaking changes |
| 🍱 | `:bento:` | Add or update assets |
| ♿️ | `:wheelchair:` | Improve accessibility |
| 💡 | `:bulb:` | Add or update comments in source code |
| 💬 | `:speech_balloon:` | Add or update text and literals |
| 🗃️ | `:card_file_box:` | Perform database related changes |
| 🔊 | `:loud_sound:` | Add or update logs |
| 🔇 | `:mute:` | Remove logs |
| 👥 | `:busts_in_silhouette:` | Add or update contributors |
| 🚸 | `:children_crossing:` | Improve user experience / usability |
| 🏗️ | `:building_construction:` | Make architectural changes |
| 📱 | `:iphone:` | Work on responsive design |
| 🤡 | `:clown_face:` | Mock things |
| 🥚 | `:egg:` | Add or update an easter egg |
| 🙈 | `:see_no_evil:` | Add or update a .gitignore file |
| 📸 | `:camera_flash:` | Add or update snapshots |
| ⚗️ | `:alembic:` | Perform experiments |
| 🔍️ | `:mag:` | Improve SEO |
| 🏷️ | `:label:` | Add or update types |
| 🌱 | `:seedling:` | Add or update seed files |
| 🚩 | `:triangular_flag_on_post:` | Add, update, or remove feature flags |
| 🥅 | `:goal_net:` | Catch errors |
| 💫 | `:dizzy:` | Add or update animations and transitions |
| 🗑️ | `:wastebasket:` | Deprecate code needing cleanup |
| 🛂 | `:passport_control:` | Work on authorization, roles, and permissions |
| 🩹 | `:adhesive_bandage:` | Simple fix for a non-critical issue |
| 🧐 | `:monocle_face:` | Data exploration/inspection |
| ⚰️ | `:coffin:` | Remove dead code |
| 🧪 | `:test_tube:` | Add a failing test |
| 👔 | `:necktie:` | Add or update business logic |
| 🩺 | `:stethoscope:` | Add or update healthcheck |
| 🧱 | `:bricks:` | Infrastructure related changes |
| 🧑‍💻 | `:technologist:` | Improve developer experience |
| 💸 | `:money_with_wings:` | Add sponsorships or money related infrastructure |
| 🧵 | `:thread:` | Code related to multithreading or concurrency |
| 🦺 | `:safety_vest:` | Add or update code related to validation |
| ✈️ | `:airplane:` | Improve offline support |
| 🦖 | `:t-rex:` | Add backwards compatibility |
