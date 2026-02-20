# Ticketing

Domain-agnostic engine for creating, tracking, and managing work items with configurable workflows and extensible fields.

## Capabilities

### P1 — Core

- [ ] Ticket CRUD (create, read, update, delete)
- [ ] Configurable workflow engine (statuses and transitions as data)
- [ ] Transition role guards and preconditions
- [ ] Configurable role mapping (domain-specific role keys)
- [ ] Custom fields per project (text, number, date, boolean, select, multiselect)
- [ ] Custom field runtime validation
- [ ] Optional requester identity (distinct from reporter)
- [ ] Comments (internal/public, sender types: reporter/agent/system)
- [ ] Immutable audit log
- [ ] Domain event publishing (created, updated, transitioned, assigned, commented, deleted, linked, watched)
- [ ] Organization-scoped multi-tenancy
- [ ] Intra-organization visibility rules (role/assignment-based access)
- [ ] Project-based ticket grouping
- [ ] Ticket assignment
- [ ] Optimistic concurrency control

### P2 — Extended

- [ ] Human-readable sequential ticket IDs (configurable prefix per project)
- [ ] Field-level access control (view/edit restrictions by role)
- [ ] Text search across tickets
- [ ] Aggregation queries (counts by status, priority, assignee)
- [ ] Bulk operations (batch assign, transition, update)
- [ ] Ticket reassignment (role-restricted)
- [ ] Project management (create, update, deactivate)
- [ ] File attachments (upload, download)
- [ ] Extensible transition conditions
- [ ] Data retention and archival policies
- [ ] PII field marking with redaction support

### P3 — Optional

- [ ] Ticket watchers (subscribe to updates)
- [ ] Ticket linking (configurable relationship types)
- [ ] Comment editing (time-windowed, audited)
- [ ] Attachment scanning (external scan port)
- [ ] SLA policy integration (optional profile)
- [ ] Async command intake (optional profile)

## Module Structure

```plaintext
modules/ticketing/
├── docs/                    # Documentation
│   ├── PRD.md               # Product requirements
│   ├── DESIGN.md            (planned)
│   ├── ADR/                 (planned)
│   └── features/            (planned)
├── DB_SCHEMA.md             # Logical database schema
├── ticketing-sdk/           # Public API traits, models, errors (planned)
└── ticketing/               # Core module implementation (planned)
```

## Documentation

| Document | Description | Status |
|----------|-------------|--------|
| [PRD.md](docs/PRD.md) | Product requirements, use cases, acceptance criteria | ✓ Complete |
| [DB_SCHEMA.md](DB_SCHEMA.md) | Logical relational schema, ER diagrams | ✓ Complete |
| DESIGN.md | Technical architecture, NFR allocation, ports/adapters | Planned |
| ADR/ | Architecture decision records | Planned |

## Dependencies

| Dependency | Role | Criticality |
|------------|------|-------------|
| Authentication Service | Authenticated user context (user ID, org ID, roles) | P1 |
| Platform Runtime | Module lifecycle and service discovery | P1 |
| Persistent Storage | Ticket, audit, and configuration persistence | P1 |
| File Storage Service | Ticket attachment storage | P2 |
| Event Delivery Infrastructure | Domain event transport | P2 |
| Archival Storage | Long-term retention (optional) | P3 |

## Consumers

| Module | Usage |
|--------|-------|
| CyberDesk Service Desk | MSP ticket management with AI-powered analysis |

## Key Design Principles

- **Domain-agnostic**: Workflow and fields are configuration, not code
- **Multi-tenant**: Strict organization-level data isolation
- **Event-driven**: All state changes emit domain events for extensibility
- **Audit-first**: Immutable audit log for compliance and transparency
- **Configurable access**: Role guards, field ACL, visibility rules — all as data
