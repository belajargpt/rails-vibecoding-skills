---
name: schema-design-flow
description: Use when designing database schema, tables, columns, associations, or migrations in Rails before writing code. Walks the user through a Socratic interrogation (who owns this? who sees it? what lifecycle? what's unique?) before generating any migration. Applies to new models, schema changes, adding columns, multi-tenancy / permission boundaries, state modeling (timestamps vs booleans), associations (belongs_to / has_many / has_many :through), indexes, foreign keys, NOT NULL discipline, default values, and reversible migration safety. Platform-agnostic — SQLite, MySQL, PostgreSQL. Triggers when user mentions schema, migration, database design, add column, new table, data model, `bin/rails g model`, `bin/rails g migration`, belongs_to, has_many, foreign key, index, or "think about the data before I code". Inspired by 37signals Fizzy state-as-records patterns.
---

# Schema Design Flow — Think in Questions, Not Tables

> Non-programmers plan apps in stories. Developers think in tables. This skill bridges both: Claude asks the story questions, the student answers, Claude drafts the tables. Never generate migrations without running the interrogation first.

## Philosophy

> Schema is the hardest thing to change later. 20 minutes of questions upfront saves 2 hours of migrations and broken deploys. Always interrogate before you migrate.

**The rule:** Before writing a single `bin/rails generate migration`, run the Socratic flow. Capture answers in words (or update the PRD). Only then draft the schema.

## The Socratic Interrogation Flow

When the user says *"add a Post model"* or *"I need a team todo list"*, **do not** immediately scaffold. Ask these questions first — one batch, in plain language:

### Universal questions (every new table)

1. **Who creates this record?** (one user? any authenticated user? admin only? system-generated?)
2. **Who can see it?** (public? only the creator? their team? their tenant/account?)
3. **Who can edit or delete it?** (creator only? admin? never?)
4. **Does it belong to something bigger?** (a user, a team, an account, a project?)
5. **What's the lifecycle?** (draft → published? open → closed? created → deleted? or just "exists"?)
6. **What makes it unique?** (is there a name / code / slug that can't repeat within a scope?)
7. **What happens when the parent is deleted?** (delete children? orphan them? forbid delete?)

### Context-specific (add these when relevant)

8. **Multi-tenant app?** → *Does every table need an `account_id` / `workspace_id` for isolation?*
9. **Soft vs hard delete?** → *Do we need a trash/undo, or is gone forever OK?*
10. **Audit trail needed?** → *Do we care who changed what and when?*
11. **Counts to display fast?** → *Will we show "50 posts" in a list? (counter_cache)*
12. **Attachments?** → *Files / images? (Active Storage — no `image_url` column)*
13. **Associations to other models?** → *What else does this record point to or own?*

**Rule:** Ask all applicable questions in one message. Then wait for answers. Don't write code in the middle of the interrogation.

## Example Interrogation (Team Todo App)

User prompt:
> *"Bikin table untuk team todo list."*

**Bad** (skipping the flow):
```
Sure! Here's the migration:
bin/rails generate model Todo title:string completed:boolean user:references
```

**Good** (interrogate first):
```
Sebelum bikin migration, saya perlu tanya 7 hal dulu — supaya schema-nya
pas sekali jadi, tidak perlu diulang:

1. Siapa yang bikin todo — anggota team, atau admin team saja?
2. Siapa yang bisa lihat — semua anggota team, atau yang di-assign saja?
3. Siapa yang bisa edit / delete — yang bikin, admin, atau yang di-assign?
4. Todo ini milik "team" ya? Berarti ada model Team juga — sudah ada?
5. Apa lifecycle-nya — cuma "done / not done", atau ada "in progress" juga?
6. Kalau team dihapus, todo-nya ikut hilang atau disimpan?
7. Perlu assign ke orang tertentu, atau todo bebas dikerjakan siapa saja?

Jawab dalam 1-2 kalimat per pertanyaan, saya tunggu dulu baru bikin schema.
```

Once the user answers, draft the schema in the **schema summary format** below — still no code — and confirm before generating the migration.

## Schema Summary Format (Draft Before Migrating)

After interrogation, present the draft as a plain-language table, not code. This is the student's last checkpoint:

```
Table: todos
- id (primary key — Rails default)
- team_id (references teams, NOT NULL)  ← multi-tenant anchor
- creator_id (references users, NOT NULL) ← who made this
- assignee_id (references users, nullable) ← who's doing it (optional)
- title (string, NOT NULL, max 255)
- description (text, nullable)
- status (string, NOT NULL, default: "open")  ← "open" | "doing" | "done"
- due_on (date, nullable)
- completed_at (datetime, nullable)  ← timestamp, not boolean
- timestamps (created_at, updated_at)

Indexes:
- team_id (every query scoped by team)
- [team_id, status] (list "open" todos per team fast)
- [assignee_id, completed_at] (my-todos page)

Associations:
- belongs_to :team
- belongs_to :creator, class_name: "User"
- belongs_to :assignee, class_name: "User", optional: true

Cascade:
- team.destroy → todos deleted (dependent: :destroy)
- user.destroy → blocked if they created any todos (foreign_key)
```

Then ask: *"Oke ga schema-nya? Kalau setuju, saya bikin migration. Kalau ada yang mau diubah, sekarang waktunya."*

## Migration Patterns (Rails 8, Platform-Agnostic)

### Generate via Rails generator

```bash
bin/rails generate migration CreateTodos \
  team:references creator:references assignee:references{null} \
  title:string description:text status:string due_on:date completed_at:datetime

bin/rails db:migrate
```

Open the generated file and tighten it before migrating.

### The reversible migration skeleton

```ruby
class CreateTodos < ActiveRecord::Migration[8.1]
  def change
    create_table :todos do |t|
      t.references :team,     null: false, foreign_key: true
      t.references :creator,  null: false, foreign_key: { to_table: :users }
      t.references :assignee, null: true,  foreign_key: { to_table: :users }

      t.string   :title,       null: false, limit: 255
      t.text     :description
      t.string   :status,      null: false, default: "open"
      t.date     :due_on
      t.datetime :completed_at

      t.timestamps
    end

    add_index :todos, [:team_id, :status]
    add_index :todos, [:assignee_id, :completed_at]
  end
end
```

**Why these choices:**
- `null: false` on everything required. Defaults at DB level, not app level.
- `limit: 255` on strings — explicit, works identically on SQLite / MySQL / Postgres.
- `status` as `string`, not `boolean` or integer enum. Readable in DB, swappable.
- `completed_at` is a timestamp, not `completed boolean`. One column carries state + when.
- Composite indexes match the real query shapes.
- `foreign_key: true` adds DB-level integrity (survives application bugs).

## Column Naming — The Golden Rules

### Timestamps over booleans

| Boolean (avoid) | Timestamp (prefer) | What it captures |
|---|---|---|
| `published` | `published_at` | state + when it happened |
| `completed` | `completed_at` | state + when |
| `verified` | `verified_at` | state + when |
| `archived` | `archived_at` | state + when |
| `read` | `read_at` | state + when |

Query is just `where.not(published_at: nil)` — same ergonomics, more info.

### `*_at` suffix for moments in time

`created_at`, `published_at`, `last_signed_in_at`. Always `datetime`, never `date` (unless truly a date — birthdays, due dates).

### Reserved singular names — avoid `type` (Rails STI collision) and `class`

### Counts

`posts_count`, `comments_count` — integer, default 0, NOT NULL. Managed via `counter_cache: true` on `belongs_to` or manually via `increment!`.

## State Modeling — The Fizzy Pattern

37signals Fizzy (kanban app) teaches this: **separate records beat boolean flags** when state has history, metadata, or optional context.

### Example: "this card is closed"

**Naive (avoid):**
```ruby
# cards table
t.boolean :closed, default: false
t.datetime :closed_at
t.references :closed_by_user, null: true
```

**Fizzy pattern (prefer):**
```ruby
# separate closures table
create_table :closures do |t|
  t.references :card, null: false, foreign_key: true
  t.references :user, null: true,  foreign_key: true  # who closed it
  t.text :reason
  t.timestamps
end
# cards has_one :closure
# card.closed? = card.closure.present?
```

**When to reach for it:**
- State has **who / when / why** metadata
- State can be **undone** (closure can be deleted → card is open again)
- State is **rare** relative to the parent (don't bloat every card with `closed_*` columns)

**When a column suffices:**
- State is binary, no metadata, no undo → single timestamp column (`completed_at`)
- State is always-present → enum column (`status`)

### Example: status as enum

```ruby
# migration
t.string :status, null: false, default: "drafted"

# model
class Post < ApplicationRecord
  enum :status, { drafted: "drafted", published: "published", archived: "archived" }
end
```

String-backed enums are portable across SQLite/MySQL/Postgres and readable in the DB.

## Associations

### belongs_to defaults

```ruby
belongs_to :team
belongs_to :creator, class_name: "User"
belongs_to :assignee, class_name: "User", optional: true  # explicit when nullable
```

Rails 5+ makes `belongs_to` required by default. Use `optional: true` when nullable is intentional.

### has_many + dependent

```ruby
class Team < ApplicationRecord
  has_many :todos, dependent: :destroy       # parent gone → children gone (with callbacks)
  has_many :events, dependent: :delete_all   # bulk delete, no callbacks (faster)
  has_many :memberships, dependent: :restrict_with_error  # prevent delete if children exist
end
```

| Option | When |
|---|---|
| `:destroy` | Most common. Runs child callbacks (e.g., Active Storage purge). |
| `:delete_all` | Child has no callbacks worth running. Faster. |
| `:restrict_with_error` | Refuse to delete parent while children exist. |
| `:nullify` | Set child's FK to NULL instead of deleting. |
| (nothing) | Rarely. DB-level foreign key decides. |

### has_many :through for many-to-many

```ruby
class Team < ApplicationRecord
  has_many :memberships
  has_many :users, through: :memberships
end

class User < ApplicationRecord
  has_many :memberships
  has_many :teams, through: :memberships
end

class Membership < ApplicationRecord
  belongs_to :team
  belongs_to :user
end
```

Always prefer `has_many :through` over `has_and_belongs_to_many` — the join model can grow (roles, invited_at, etc.).

## Indexes — Match Your Queries

### The three types you'll actually use

```ruby
add_index :todos, :team_id                        # single column, matches scope queries
add_index :todos, [:team_id, :status]             # composite, matches list-filtered queries
add_index :users, :email, unique: true            # unique constraint at DB level
```

### Composite index rule

**Leading column must match your `WHERE`.** Index `[:team_id, :status]` supports:
- `WHERE team_id = ?` ✅
- `WHERE team_id = ? AND status = ?` ✅
- `WHERE status = ?` ❌ (index not used — leading column missing)

### When to add an index

- Every foreign key (`t.references` adds it automatically)
- Every column you `WHERE`, `ORDER BY`, or `JOIN` on
- Unique business keys (email, slug, code)

**Don't** index every column "just in case" — indexes cost write speed + disk.

## Multi-Tenancy — Always Scope by Account

If the app has multiple customers / teams / workspaces, **every user-owned table gets the tenant FK**.

```ruby
create_table :todos do |t|
  t.references :account, null: false, foreign_key: true  # tenant anchor
  t.references :team,    null: false, foreign_key: true
  # ...
end

add_index :todos, [:account_id, :team_id, :status]  # leading column = tenant
```

In models:
```ruby
class Todo < ApplicationRecord
  belongs_to :account
  belongs_to :team

  # Convenience: infer account from team (avoid passing it manually)
  # belongs_to :account, default: -> { team.account }
end
```

Enforce in controller via `Current.account`:
```ruby
# All queries scoped automatically
Current.account.todos.where(team_id: params[:team_id])
```

## Platform-Agnostic Column Types

Stick to these for SQLite ↔ MySQL ↔ PostgreSQL portability:

| Use | Type | Notes |
|---|---|---|
| Short text | `:string` | Default limit 255 in MySQL; explicit `limit: 255` is safest |
| Long text | `:text` | Unlimited on all three |
| Integer | `:integer` | 4-byte |
| Big integer | `:bigint` | 8-byte, for IDs / large counts |
| Decimal money | `:decimal, precision: 10, scale: 2` | Never `:float` for money |
| Date only | `:date` | Birthdays, due dates |
| Date + time | `:datetime` | `*_at` columns |
| Boolean | `:boolean, default: false, null: false` | Rare — prefer timestamps |
| Structured | `:json` | Works on all three (Postgres stores as `json`, MySQL as `json`, SQLite as text) |
| UUID | `:string, limit: 36` | Generate via `SecureRandom.uuid`; portable |

**Avoid for portability:**
- `:jsonb` (Postgres-only — use `:json` if you need it; index support is reduced)
- `:citext` (Postgres-only — use `:string` + normalize via `.downcase` before save)
- `:inet`, `:array`, `:hstore` (Postgres-only)
- `t.uuid` native type (Postgres-only; use string UUID above for portability)
- `algorithm: :concurrently` on indexes (Postgres-only)
- Partial indexes `where: "..."` (Postgres + SQLite only — MySQL ignores)

When Postgres-specific features are worth it, **commit to Postgres for the project** and document the decision. Don't mix portability claims with Postgres-only columns.

## Common Migration Operations

### Add a column
```ruby
class AddDueOnToTodos < ActiveRecord::Migration[8.1]
  def change
    add_column :todos, :due_on, :date
    add_index  :todos, :due_on
  end
end
```

### Add a NOT NULL column to an existing table (safe 3-step)

Large tables need this pattern to avoid long locks:

```ruby
# Migration 1 — add as nullable
add_column :todos, :priority, :integer

# Deploy + backfill in a rake task or Active Job
Todo.in_batches.update_all(priority: 0)

# Migration 2 — enforce NOT NULL + default
change_column_null    :todos, :priority, false
change_column_default :todos, :priority, from: nil, to: 0
```

Small tables (< 10K rows) — one migration is fine.

### Rename a column (two-deploy dance)

```ruby
# Deploy 1 — add new column, keep old, dual-write in app
add_column :todos, :title, :string
# (backfill + update app to write to both)

# Deploy 2 — drop old
remove_column :todos, :name, :string
```

Never rename in one shot on a production app — older web containers will 500 on the missing column during deploy.

### Drop a column

```ruby
class RemoveCompletedFromTodos < ActiveRecord::Migration[8.1]
  def change
    remove_column :todos, :completed, :boolean, default: false, null: false
  end
end
```

Include the full original definition as the 3rd+ argument so `down` is reversible.

## NOT NULL Discipline

**Default to NOT NULL.** Every column should be NOT NULL unless nullability is a real business fact.

| Column | NOT NULL? |
|---|---|
| `title` on Post | Yes — every post must have a title |
| `published_at` on Post | No — `nil` means draft |
| `user_id` on Post | Yes — every post has an author |
| `description` on Post | Debatable — if "optional", NULL is fine; if "empty string OK", use `default: ""` NOT NULL |

Rule: if you'd say "the column *has* to have a value," add `null: false, default: X`.

## Anti-Patterns

- ❌ Generating migrations before interrogation (defeats the skill's purpose)
- ❌ Boolean flags where a timestamp carries more info (`published` vs `published_at`)
- ❌ Nullable columns without thinking (`null: true` by default is lazy)
- ❌ Missing foreign key constraints (app-only integrity breaks in raw SQL / console bugs)
- ❌ Missing indexes on FK + query columns (slow at 10K rows)
- ❌ Postgres-only features (`jsonb`, `citext`, `gen_random_uuid()`) in a "portable" schema
- ❌ STI `type` column when a regular polymorphic or separate table would be cleaner
- ❌ `has_and_belongs_to_many` — always use `has_many :through` so the join can grow
- ❌ Renaming columns in one deploy (causes 500s during rollout)
- ❌ Changing a migration that already ran in production (create a new migration instead)

## How to Use This Skill

1. **User asks for a new model / table / schema change.**
2. **Run the interrogation** — universal questions + context-specific. One batch.
3. **Wait for answers.** Don't write code.
4. **Draft the schema summary in plain language** (table above). Get confirmation.
5. **Generate the migration.** Open the file, tighten defaults / nulls / indexes.
6. **Run `bin/rails db:migrate`.**
7. **Roll back once as a sanity check:** `bin/rails db:rollback && bin/rails db:migrate`. Both succeed → schema is reversible.
8. **Commit.** Migrations are forever — git history proves intent.

## Related Skills

- **prd-writing** — PRD informs "who owns / who sees / what lifecycle" (feeds the interrogation)
- **rails-conventions** — models, validations, associations (the code after the schema)
- **auth-setup** — permission boundaries shape tenant/account schema decisions
- **active-storage** — file columns (don't add `image_url` strings — use `has_one_attached`)
- **rails-debug-helper** — diagnosing migration / schema drift issues
