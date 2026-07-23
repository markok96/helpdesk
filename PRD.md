# PRD: Multi-tenant support ticket helpdesk (v1)

## Problem Statement

Small support teams need a place to receive, track, and work customer requests without losing track of ones that have been waiting too long for a reply — and without one company's support data being visible to another's. Right now there's nothing here at all: no way for a customer to submit a request, no shared view for a team to see what's outstanding, and no way to know when something's gone quiet for too long.

## Solution

A multi-tenant helpdesk. Each Organization gets its own fully isolated space: its own Admins and Agents, its own Customers, its own Tickets, invisible to every other Organization. Customers reach a specific Organization through its own public support page, sign up, and submit Tickets. Agents work from a shared queue — claiming a Ticket assigns it to them and starts the conversation. A background sweep continuously watches for Tickets that have been waiting on an Agent's response too long and emails the right person before it's forgotten entirely.

## User Stories

**Organization & team management**

1. As a new user, I want to sign up and create a brand-new Organization in the same step, so that I immediately have my own isolated helpdesk.
2. As an Admin, I want to invite an Agent (or another Admin) by email, so that I can build my support team without them self-registering.
3. As an invited Agent or Admin, I want to set my password when accepting an invite, so that I can securely access my Organization's helpdesk.
4. As an Admin, I want to change a User's role between Agent and Admin, so that I can adjust my team's structure as it grows.
5. As an Admin, I want to remove a User from my Organization, so that former team members immediately lose access.

**Customer-facing**

6. As a prospective Customer, I want to visit an Organization's own public support page, so that I know exactly who I'm submitting my request to.
7. As a new Customer, I want to sign up directly from an Organization's support page, so that I can submit a Ticket without needing a prior invitation.
8. As a returning Customer, I want to log in and see all my previous Tickets, so that I can track their status over time.
9. As a Customer, I want to submit a new Ticket with a subject and an initial message, so that I can describe my issue.
10. As a Customer, I want to add follow-up messages to my Ticket, so that I can give more information or ask for an update.
11. As a Customer, I want to see my Ticket's current status (Open, In Progress, Resolved), so that I know whether anyone is working on it.
12. As a Customer, I want my reply to a Resolved Ticket to reopen it automatically, so that I don't have to start a brand-new Ticket for the same issue.
13. As a Customer, I want to see only my own Tickets, never another Customer's, so that my support history stays private.
14. As a Customer, I want to be notified by email when an Agent replies to my Ticket, so that I know to check back.

**Agent-facing**

15. As an Agent, I want to see a shared queue of unclaimed Tickets in my Organization, so that I know what needs attention.
16. As an Agent, I want to claim an unclaimed Ticket, so that it's assigned to me and moves to In Progress in the same step.
17. As an Agent, I want to reply to a Ticket assigned to me, so that I can help the Customer.
18. As an Agent, I want to mark a Ticket as Resolved once I've handled it, so that it's clear the conversation is done.
19. As an Agent, I want to release a Ticket back to the unclaimed queue, so that another Agent can pick it up if I can't handle it.
20. As an Agent, I want to see only Tickets belonging to my own Organization, so that I never see another company's support data.
21. As an Agent, I want a Ticket I resolved to come back to me automatically (not the shared queue) if the Customer replies again, so that I don't lose context I already have.
22. As an Agent, I want to be notified by email when a Customer replies to a Ticket assigned to me, so that I know to follow up.
23. As an Agent, I want to be notified by email when a Ticket assigned to me becomes Overdue, so that I don't accidentally leave a Customer waiting too long.

**Admin-facing (ticket work)**

24. As an Admin, I want to reassign a Ticket directly to a specific Agent, so that I can route work by expertise or availability.
25. As an Admin, I want all the same ticket-working abilities an Agent has, so that I can personally pitch in on support work.
26. As an Admin or any Agent, I want to be notified by email when an unclaimed Ticket becomes Overdue, so that someone claims it before it's ignored for too long.

**Isolation**

27. As an Organization, I want my Tickets, Customers, and Agents to be completely invisible to every other Organization, so that my support data and customer relationships stay private.

## Implementation Decisions

- **Data model**: `Organization` (name, slug — used for its public support page URL). `User` (Devise-backed; `organization_id`; `role` enum: customer / agent / admin; email unique **scoped to `organization_id`**, not globally unique — see ADR 0001). `Ticket` (`organization_id`, `customer_id`, `assigned_agent_id` nullable, `subject`, `status` enum: open / in_progress / resolved). `Message` (`ticket_id`, `sender_id`, `body`).
- **Multi-tenancy shape**: a `User` belongs to exactly one `Organization` with exactly one role — no `Membership` join table (ADR 0001, chosen over the more "flexible" multi-org-per-user pattern because nothing in this product's requirements needs a person to belong to more than one Organization). Every query and Action Policy scopes by `organization_id` first, role second.
- **Authorization** (Action Policy): Customer — create/read only their own Tickets and Messages on them. Agent — read/claim/release any Ticket in their own Organization, message on Tickets assigned to them. Admin — everything an Agent can, plus manage Users in their Organization (invite, remove, change role).
- **Ticket lifecycle**: Open (unclaimed) → In Progress (claimed) → Resolved. **Claim** is a single action/event: it assigns the Agent and moves Open → In Progress together, not two separate steps. **Release** is the reverse (In Progress → Open, unassigned). An Admin reassigning a Ticket to a specific Agent is a release + claim performed as one action.
- **Reopen behavior**: a Customer Message on a Resolved Ticket reopens it, reassigning automatically to whichever Agent resolved it (straight back to In Progress, skipping the unclaimed queue) — falling back to unclaimed Open only if that Agent is no longer part of the Organization.
- **Overdue** is a *computed* value, not a stored column: a Ticket is Overdue when it's unclaimed, or In Progress with the most recent Message from the Customer, for longer than a fixed SLA threshold. The threshold is a single global constant, not configurable per Organization (deliberately — see Out of Scope).
- **Background jobs** (Solid Queue): (1) a recurring sweep across non-Resolved Tickets that computes Overdue status and triggers the Overdue notification — needs an idempotency guard (e.g. a `last_overdue_notified_at` timestamp) so it doesn't re-notify on every sweep, and that guard needs to reset whenever a Ticket stops being Overdue so it can fire again later if it becomes overdue a second time; (2) a mailer job fired after a new Message is created, notifying whichever party didn't send it.
- **Sign-up paths**: (a) create a brand-new Organization and become its Admin, (b) accept an email invite to join an existing Organization as Agent/Admin, or (c) self-service sign-up as a Customer via a specific Organization's public page (`/o/:slug`).
- No internal/agent-only notes — every Message is visible to both Customer and Agent.
- No file attachments — text-only Messages for v1.
- Scaffolded from the `manufaktura-koda-shabloon` Rails application template: Rails 8, Postgres, Devise + Action Policy, Inertia.js + Vite with React/TypeScript, Typelizer/Alba for typed props, Solid Queue, RSpec.

## Testing Decisions

- **Primary seam: request specs.** Hit real routes/controllers end-to-end and assert on the Inertia response (rendered component + props), not on internal implementation. Cover the full flows: submit ticket, claim, reply, resolve, reopen-with-auto-reassignment, release, admin reassignment — and critically, cross-organization isolation (an Agent or Customer belonging to Organization A must never be able to read or act on Organization B's Ticket, regardless of role).
- **Unit specs for the Overdue computation** in isolation — given a Ticket's status, assignment, and the sender of its most recent Message, does it count as Overdue? This is a pure business rule and the kind of logic that's worth testing directly rather than only indirectly through a full request.
- **Job specs** for the two Solid Queue jobs (the overdue sweep, and the message-notification mailer job) — assert they enqueue/perform and invoke the right mailer for the right recipient.
- **What makes a good test here**: assert on behavior (given this Ticket state and Message history, is it Overdue? Can this User see this Ticket?), never on implementation details (which column stores the role, the exact scoping SQL) — the internal representation (Overdue being computed, not stored) should stay free to change without breaking tests.
- No prior test conventions exist yet (brand-new app) — follow whatever RSpec/FactoryBot/Faker/Shoulda Matchers setup the `manufaktura-koda-shabloon` template itself generates.

## Out of Scope

- Per-Organization configurable SLA thresholds (fixed global value only).
- Internal/agent-only notes on Tickets.
- File attachments on Messages.
- Ticket priority levels, categories, or tags.
- A single User belonging to more than one Organization (explicitly rejected — see ADR 0001).
- Automatic/round-robin ticket assignment (manual claim-from-queue only).
- Search, filtering, or sorting on the ticket queue.
- Organization branding/customization of the public support page.
- "Ticket submitted" confirmation emails and "Ticket resolved" notification emails — only the two notification triggers defined above (new Message, and newly-Overdue) exist.

## Further Notes

- This is a portfolio project (replacing outdated tutorial-tier projects on the author's existing portfolio site) explicitly meant to demonstrate real professional-stack skill (Rails 8, Inertia.js, React, TypeScript, background jobs, multi-tenant auth/authorization) rather than to become a real commercial product.
- Full domain glossary: `CONTEXT.md`. Architectural rationale for the account model: `docs/adr/0001-user-belongs-to-one-organization.md`.
- Deployment (not yet started): intended to run on a small shared VPS (Hetzner) alongside other portfolio demo projects, deployed via Kamal — already configured by the `manufaktura-koda-shabloon` template.
