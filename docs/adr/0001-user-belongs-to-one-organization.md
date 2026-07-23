# User belongs to exactly one Organization, no Membership join table

This is a multi-tenant app (Organizations are fully isolated from each other), which made a `User` ↔ `Organization` `Membership` join table — letting one person hold different roles across multiple Organizations — the default-seeming choice. We rejected it: nothing in this app's actual requirements needs a person to belong to more than one Organization, and the join table adds real cost (an extra model, "current organization" context-switching in every session/request, every policy and query indirecting through Membership instead of the User directly) for flexibility nobody asked for.

Instead, `User` has a direct `organization_id` and a `role` (Customer, Agent, or Admin) — one Organization, one role, no join table. If multi-organization membership becomes a real requirement later, it's a clean, isolated migration (introduce the join table then) rather than complexity paid for upfront on a maybe.
