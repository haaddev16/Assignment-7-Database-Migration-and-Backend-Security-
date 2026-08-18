# Muhammad Haad
### Course: AI Seekho

# Muhammad Haad
### Course: AI Seekho

# Assignment 7 — Databases, Migrations & Backend Security

**Name:** Muhammad Haad
**Course:** AI Seekho

---

## 1. PostgreSQL

```sql
CREATE TABLE plan_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id UUID NOT NULL REFERENCES plans(id) ON DELETE CASCADE,
    event_type VARCHAR(50) NOT NULL,
    guest_count INTEGER NOT NULL CHECK (guest_count > 0),
    created_at TIMESTAMP DEFAULT now()
);

SELECT plans.id AS plan_id, plans.plan_type, plan_events.event_type, plan_events.guest_count
FROM plans
JOIN plan_events ON plan_events.plan_id = plans.id;
```

---

## 2. MongoDB

```javascript
db.plan_events.insertOne({
  plan_id: "8f14e45f-ea9b-4c2a-8b3a-1f2c9d3e4a5b",
  event_type: "Walima",
  guest_count: 300,
  created_at: new Date()
});

db.plan_events.find({ event_type: "Walima" });
```

---

## 3. Alembic

```python
"""add decor_package column to plan_events

Revision ID: a1b2c3d4e5f6
Revises: 
Create Date: 2026-08-16
"""
from alembic import op
import sqlalchemy as sa

revision = 'a1b2c3d4e5f6'
down_revision = None

def upgrade():
    op.add_column('plan_events', sa.Column('decor_package', sa.String(20), nullable=True))

def downgrade():
    op.drop_column('plan_events', 'decor_package')
```

---

## 4. Data Migration

To split `full_name` into `first_name` and `last_name` without downtime: first add both new columns as nullable so the deploy doesn't lock or break existing reads/writes. Next, run a backfill script that parses `full_name` for existing rows and populates `first_name`/`last_name` in small batches (to avoid long locks), while the application keeps writing to `full_name` as before. Then deploy an application update that writes to all three columns simultaneously (dual-write) for a transition period, and switch reads over to the new columns once backfill is verified complete. Finally, once confident, drop the `full_name` column and remove the dual-write code in a later, separate deployment.

---

## 5. Order of Command (Precedence) in DB

```sql
-- Actual execution order: FROM -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY
SELECT event_type, COUNT(*) AS total_plans
FROM plan_events
WHERE guest_count > 50
GROUP BY event_type
HAVING COUNT(*) > 2
ORDER BY total_plans DESC;
```

---

## 6. Idempotency

An operation is idempotent if performing it multiple times produces the same result as performing it once — repeating the request doesn't change the outcome beyond the first application. For example, `PUT /users/5` with a full replacement body is idempotent (setting the same data repeatedly leaves the resource in the same state), while `POST /orders` is non-idempotent because each call typically creates a brand new order, so calling it twice creates two separate orders.

---

## 7. Session Hijacking — 3 Mitigations

- **HttpOnly + Secure cookie flags**: `HttpOnly` prevents JavaScript (and therefore XSS attacks) from reading the session/JWT cookie; `Secure` ensures the cookie is only ever sent over HTTPS, preventing it from being exposed on plain HTTP.
- **Short-lived tokens with refresh rotation**: keeping access tokens short-lived (e.g. 15-60 minutes) and rotating refresh tokens on each use limits how long a stolen token remains useful to an attacker.
- **Enforced HTTPS/TLS everywhere**: encrypting all traffic prevents an attacker on the network (e.g. public Wi-Fi) from intercepting the session cookie/token in transit in the first place.

---

## 8. ACID Properties

- **Atomicity**: a transaction's operations either all succeed together or all fail together — there's no partial completion.
- **Consistency**: a transaction moves the database from one valid state to another, never violating defined rules/constraints.
- **Isolation**: concurrent transactions don't see each other's uncommitted intermediate changes.
- **Durability**: once a transaction is committed, its changes persist even if the system crashes immediately after.

```sql
BEGIN;
UPDATE plans SET status = 'confirmed' WHERE id = '8f14e45f-ea9b-4c2a-8b3a-1f2c9d3e4a5b';
INSERT INTO plan_events (plan_id, event_type, guest_count) VALUES ('8f14e45f-ea9b-4c2a-8b3a-1f2c9d3e4a5b', 'Walima', 300);
COMMIT;
-- If either statement fails, the whole transaction rolls back — neither change is applied (atomicity).
```

---

## 9. Decoupling Identity from Data

Application tables should reference users by a stable `user_id` (UUID) rather than duplicating email or name on every table, because personal details like email or name can change over time — if they're copied into many tables, every change requires updating all of them, risking inconsistency. A UUID foreign key stays constant for the life of the account regardless of profile edits, so all related records (plans, orders, estimates) remain correctly linked without any update cascade. It also keeps sensitive personal data centralized in one place (the `users` table), making it easier to secure, audit, and comply with privacy requirements, rather than scattered across the schema.
