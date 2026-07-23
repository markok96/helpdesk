# Build Plan

See `CONTEXT.md` for the domain glossary and `docs/adr/` for architectural decisions. This doc translates that into an actual build order.

## Stack

Scaffolded from the `manufaktura-koda-shabloon` Rails template: Rails 8, Postgres, Devise + Action Policy, Inertia.js + Vite with React/TypeScript, Solid Queue (background jobs), Typelizer/Alba (typed props), RSpec.

## Data model

- **Organization** — `name`, `slug` (for its public support page URL, e.g. `/o/acme`)
- **User** (Devise) — `organization_id`, `role` (enum: `customer` / `agent` / `admin`), email **unique scoped to `organization_id`** (not globally unique — the same email can belong to different Users in different Organizations, since Users don't span Organizations per ADR 0001)
- **Ticket** — `organization_id`, `customer_id` (User), `assigned_agent_id` (User, nullable), `subject`, `status` (enum: `open` / `in_progress` / `resolved`)
- **Message** — `ticket_id`, `sender_id` (User), `body`

Derived/computed, not stored columns:
- **Overdue** — computed from `status`, `assigned_agent_id`, and the sender of the most recent Message (see `CONTEXT.md`), not a stored flag. Recomputed each time the background job scans tickets.

## Authorization (Action Policy)

- **Customer**: create Tickets and Messages on their own Tickets; read only their own Tickets. No access to other Customers' Tickets, no Organization management.
- **Agent**: read/claim/release any Ticket in their Organization; create Messages on Tickets assigned to them.
- **Admin**: everything an Agent can, plus manage Users in their Organization (invite, remove, change role).
- All scoped by `organization_id` first, role second — a query should never be able to cross an Organization boundary regardless of role.

## Key flows

1. **Organization sign-up**: a new User signs up and creates a new Organization in the same step → becomes its first Admin.
2. **Team invites**: an Admin invites an Agent (or another Admin) by email → they set a password and join that Organization with the given role.
3. **Customer self-service**: anyone visits an Organization's public page (`/o/:slug`) → signs up or logs in as a Customer of that Organization → submits a Ticket (first Message).
4. **Claiming**: an Agent sees the unclaimed queue (`status: open`) → claims a Ticket → `status` becomes `in_progress`, `assigned_agent_id` set.
5. **Working a ticket**: Agent and Customer exchange Messages. Agent marks it `resolved` when done.
6. **Reopening**: a Customer Message on a `resolved` Ticket flips it back to `in_progress`, reassigned to whichever Agent resolved it (they already have context) — or to `open` (unclaimed) if that Agent is no longer part of the Organization.
7. **Release/reassignment**: an Agent releases a Ticket back to `open`; an Admin can directly reassign to a specific Agent (equivalent to release + claim in one step).
8. **Overdue sweep**: a recurring Solid Queue job scans all non-resolved Tickets, computes Overdue per Organization's fixed threshold, and triggers the Overdue notification for newly-overdue ones (needs a way to avoid re-notifying every sweep for the same overdue ticket — likely a `last_overdue_notified_at` timestamp).
9. **Message notifications**: after a Message is created, notify the other party (customer ↔ assigned agent) via a background mailer job.

## Suggested build order

1. Scaffold the Rails app from the template (`rails new -m template.rb`), choosing React for the Inertia frontend.
2. Organization + User models, Devise config (email scoped uniqueness), org sign-up + invite flow.
3. Ticket + Message models, Action Policies, the core submit → claim → reply → resolve loop end-to-end (no background jobs yet).
4. Reopen + release/reassignment logic.
5. Solid Queue: the overdue sweep job + the two mailer jobs (message notification, overdue notification).
6. Polish pass: empty states, the public per-org support page design, deploy.
