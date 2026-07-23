# Helpdesk

A multi-tenant support ticket system. Small support teams need a place to receive, track, and work customer requests without losing track of ones that have been waiting too long for a reply — and without one company's support data being visible to another's.

Each Organization gets its own fully isolated space: its own Admins and Agents, its own Customers, its own Tickets. Customers reach a specific Organization through its own public support page, sign up, and submit Tickets. Agents work from a shared queue — claiming a Ticket assigns it to them and starts the conversation. A background sweep continuously watches for Tickets that have been waiting on an Agent's response too long and emails the right person before it's forgotten entirely.

Built with Ruby on Rails 8, Inertia.js, and React/TypeScript, scaffolded from the [manufaktura-koda-shabloon](https://github.com/JovicaSusa/manufaktura-koda-shabloon) application template.

Part of Marko Korolija's portfolio: https://markok96.github.io/portfolio/

## Documentation

- [`PRD.md`](./PRD.md) — full product requirements: problem statement, user stories, implementation and testing decisions, and what's explicitly out of scope.
- [`CONTEXT.md`](./CONTEXT.md) — the domain glossary (Organization, User, Ticket, Claim, Overdue, etc.).
- [`PLAN.md`](./PLAN.md) — data model, authorization rules, key flows, and suggested build order.
- [`docs/adr/`](./docs/adr/) — architectural decisions, e.g. why a User belongs to exactly one Organization.

Work is tracked as GitHub issues, broken into vertical slices; see the repo's [issue tracker](https://github.com/markok96/helpdesk/issues).
