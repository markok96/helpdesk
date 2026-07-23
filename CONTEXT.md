# Helpdesk

A multi-tenant support ticket system. Customers submit tickets to an organization; agents within that organization pick them up, work them, and resolve them.

## Language

**Organization**:
The top-level tenant boundary. Owns its own Agents, Customers, and Tickets, fully isolated from every other Organization's data.
_Avoid_: Tenant, company, account, team

**User**:
A person with an account in the system, belonging to exactly one Organization with exactly one role (Customer, Agent, or Admin).
_Avoid_: Account, person, member

**Customer** (role):
A User who can submit Tickets to their Organization and track their own Tickets. Cannot see other Customers' Tickets or manage the Organization.

**Agent** (role):
A User who works Tickets within their Organization — claims Tickets, replies, and updates their status.

**Admin** (role):
A User who can do everything an Agent can, plus manage the Organization itself (its Users and settings).
_Avoid_: Owner, manager

**Ticket**:
A request for help submitted by a Customer to an Organization. Moves through three states: Open, In Progress, Resolved. A Customer reply to a Resolved Ticket reopens it, reassigned to whichever Agent resolved it (straight back to In Progress) — or to the unclaimed Open queue if that Agent is no longer part of the Organization.
_Avoid_: Case, issue, request

**Claim**:
The action an Agent takes to take ownership of an unclaimed Ticket from the Organization's shared queue, which moves it from Open to In Progress in the same step.
_Avoid_: Assign (assignment is a side effect of claiming, not a separate action an Agent performs on themselves; an Admin assigning a Ticket to a specific Agent is still "claiming" on that Agent's behalf)

**Release**:
The action an Agent takes to return a claimed Ticket to the Organization's unclaimed queue (In Progress → Open, no assignee). An Admin reassigning a Ticket directly to a different Agent has the same effect, immediately followed by that Agent claiming it.
_Avoid_: Unassign, unclaim

**Message**:
A single reply within a Ticket's conversation thread, sent by either the Customer or the Agent working it.
_Avoid_: Comment, reply, note

**Overdue**:
A Ticket is Overdue when it has been waiting on an Agent's response for longer than a fixed SLA threshold (the same for every Organization) — i.e. it's unclaimed, or it's In Progress and the most recent Message was from the Customer. Time spent waiting on the Customer does not count toward this.
_Avoid_: Late, breached, expired
