# Muhammad Haad
### Course: AI Seekho

# Assignment 7 — Capstone Report
## Project: EventFeast (Event Catering & Decor Estimator)

**Repo:** [add your GitHub repo link(s) here — backend & frontend]
**Live app:** https://eventfeast-frontend.vercel.app

---

### Database Choice: PostgreSQL

EventFeast uses **PostgreSQL**, hosted on Neon (a serverless Postgres provider), accessed asynchronously via SQLAlchemy + asyncpg from the FastAPI backend.

### Why PostgreSQL and not MongoDB

EventFeast's data is inherently **relational and structured**: a user owns one or more plans, each plan contains one or more events, and each event contains multiple menu items plus an optional decor package. These relationships have clear foreign keys and benefit directly from PostgreSQL's strengths:

- **Strong relational integrity**: `plan_events.plan_id` and `plan_event_items.plan_event_id` are enforced foreign keys — the database itself guarantees an event can never reference a plan that doesn't exist, and items can't be orphaned from an event. MongoDB has no native foreign key enforcement; that logic would have to be re-implemented and manually maintained in application code.
- **Joins for reporting**: features like the Dashboard/History screen and the combined multi-event cost summary require joining across `plans`, `plan_events`, and `plan_event_items` in a single query. SQL joins handle this naturally and efficiently; the equivalent in MongoDB would mean either denormalizing data (risking duplication/inconsistency) or performing multiple round-trip queries / aggregation pipelines in application code.
- **ACID transactions for calculations**: when a user saves a plan, multiple related rows (plan, its events, its items) are written together. Wrapping this in a single Postgres transaction guarantees it's all-or-nothing — if anything fails partway, nothing is left in a half-saved, inconsistent state. This matters for a cost/quantity estimator where partial data would be actively misleading.
- **Fixed, well-understood schema**: the shape of a "plan" doesn't vary per-user or evolve unpredictably — every plan has the same structure (events, items, guest counts, prices). This is exactly the case relational databases are optimized for; a flexible/schema-less document store like MongoDB is more valuable when different records genuinely need different shapes, which isn't the case here.
- **Ecosystem fit**: the project already uses FastAPI + SQLAlchemy + Pydantic, all of which are built with typed, relational data in mind, and Neon provides a free, serverless-friendly Postgres instance that fits a small full-stack project's needs without infrastructure overhead.

MongoDB would only have made sense if EventFeast needed to store loosely structured or highly variable documents (e.g. arbitrary user-generated content with unpredictable fields) or needed to scale horizontally across many nodes from day one — neither applies to this project's actual requirements.

### Database Schema

**users**
| Column | Type | Notes |
|---|---|---|
| id | UUID (PK) | |
| name | VARCHAR | |
| email | VARCHAR | unique |
| password_hash | VARCHAR | bcrypt hash |
| created_at | TIMESTAMP | |

**menu_catalog** (predefined reference items)
| Column | Type | Notes |
|---|---|---|
| id | UUID (PK) | |
| name | VARCHAR | e.g. "Chicken Biryani" |
| category | VARCHAR | e.g. "Rice & Biryani" |
| unit | VARCHAR | g / ml / pieces |
| per_person_quantity | NUMERIC | |
| per_person_price | NUMERIC | |

**plans**
| Column | Type | Notes |
|---|---|---|
| id | UUID (PK) | |
| user_id | UUID (FK → users.id) | |
| plan_type | VARCHAR | single / multi |
| created_at | TIMESTAMP | |

**plan_events**
| Column | Type | Notes |
|---|---|---|
| id | UUID (PK) | |
| plan_id | UUID (FK → plans.id) | |
| event_type | VARCHAR | Mehndi / Baraat / Walima / Birthday / etc. |
| guest_count | INTEGER | |
| decor_package | VARCHAR (nullable) | basic / standard / premium |
| decor_cost | NUMERIC (nullable) | |

**plan_event_items**
| Column | Type | Notes |
|---|---|---|
| id | UUID (PK) | |
| plan_event_id | UUID (FK → plan_events.id) | |
| catalog_item_id | UUID (FK → menu_catalog.id, nullable) | null if custom item |
| custom_name | VARCHAR (nullable) | |
| per_person_quantity | NUMERIC | |
| unit | VARCHAR | |
| per_person_price | NUMERIC | |

All foreign keys are indexed (`user_id` on `plans`, `plan_id` on `plan_events`, `plan_event_id` on `plan_event_items`) to keep dashboard/history queries fast as data grows. Every query that touches a user's plans is scoped by `user_id`, ensuring one user's data is never visible to another — this is the same relational structure that made a straightforward `JOIN`-based, foreign-key-enforced schema the right choice for this project.
