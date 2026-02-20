# PRD — Ticketing

<!--
=============================================================================
PRODUCT REQUIREMENTS DOCUMENT (PRD)
=============================================================================
PURPOSE: Define WHAT the ticketing module must do and WHY — business
requirements, functional capabilities, and quality attributes for a
domain-agnostic ticket management engine.

SCOPE:
  ✓ Business goals and success criteria
  ✓ Actors (users, systems) that interact with this module
  ✓ Functional requirements (WHAT, not HOW)
  ✓ Non-functional requirements (quality attributes, SLOs)
  ✓ Scope boundaries (in/out of scope)
  ✓ Assumptions, dependencies, risks

NOT IN THIS DOCUMENT (see other templates):
  ✗ Technical architecture, design decisions → DESIGN.md
  ✗ Why a specific technical approach was chosen → ADR/
  ✗ Detailed implementation flows, algorithms → features/
=============================================================================
-->

## 1. Overview

### 1.1 Purpose

The Ticketing module is a domain-agnostic engine for creating, tracking, and managing work items. It provides configurable workflows, extensible custom fields, and an event-driven architecture that allows any domain — IT helpdesk, warehouse, retail, school administration, housing management, HR, or others — to implement their specific ticket lifecycle without code changes.

### 1.2 Background / Problem Statement

Organizations across all industries need to track units of work from creation to resolution. IT teams manage incidents and service requests, warehouses track equipment repairs, schools handle student and facility issues, and housing managers process maintenance requests.

Existing solutions are either domain-locked (purpose-built for a single vertical) or overly generic (requiring extensive custom development for each domain). Both approaches result in high total cost of ownership and slow adaptation to new business processes.

The core challenge is that each domain has different status names, transition rules, custom data fields, and role hierarchies — yet the underlying lifecycle pattern (create, assign, track, resolve, close) is identical everywhere. A single configurable engine eliminates redundant implementations while preserving domain-specific flexibility.

### 1.3 Goals (Business Outcomes)

- Enable new domain onboarding through configuration alone, with zero code changes for workflow and field customization
- Provide a complete audit trail for all ticket operations, supporting compliance requirements across regulated industries
- Reduce mean time to resolution by surfacing relevant ticket data and enabling efficient assignment and tracking
- Support multi-tenant deployment where organizations share infrastructure but have fully isolated data
- Publish domain events that allow external systems to extend functionality without modifying the ticketing core

### 1.4 Success Criteria (Measurable)

- Core ticket operations (create/read/update/transition) achieve monthly availability of at least 99.9% excluding planned maintenance
- 100% of successful ticket state mutations produce immutable audit log entries with actor and timestamp
- 100% of required domain event types are emitted for successful corresponding operations
- p95 latency targets for query and write paths meet the NFR thresholds defined in this document under expected load
- New domain rollout can be completed through workflow and field configuration without ticketing module code changes

### 1.5 Glossary

| Term | Definition |
|------|------------|
| Ticket | A unit of work representing a request, issue, or task to be tracked and resolved |
| Project | A logical container for tickets that defines a specific workflow and set of custom fields |
| Organization | A top-level tenant entity representing a company or business unit; all data is scoped to an organization |
| Workflow | A configuration defining the allowed statuses and transitions for tickets within a project |
| Transition | A named action that moves a ticket from one status to another, governed by role guards and preconditions |
| Custom Field | A domain-specific data attribute defined per project and validated at runtime |
| Reporter | The user who created a ticket |
| Requester | The external party on whose behalf a ticket was created, when distinct from the reporter |
| Watcher | An actor who subscribes to updates on a ticket without being assigned to it |
| Ticket Link | A configured relationship between two tickets (e.g., related, duplicate, parent/child) |
| Agent | A user responsible for resolving tickets |
| Domain Event | A notification published when a significant state change occurs on a ticket |

## 2. Actors

### 2.1 Human Actors

#### Reporter

**ID**: `cpt-cf-ticketing-actor-reporter`

**Role**: End user who creates tickets to report issues or request work. Tracks the progress of their own tickets and communicates with agents.
**Needs**: Quick ticket creation, status visibility, communication channel with assigned agents.

#### Requester

**ID**: `cpt-cf-ticketing-actor-requester`

**Role**: The external party (person, customer, or entity) on whose behalf a ticket is created, when distinct from the reporter. In many domains, a reporter (agent or staff member) creates tickets on behalf of a requester (customer, student, tenant, etc.).
**Needs**: Status visibility for their tickets, communication channel with assigned agents, ability to add comments.

#### Agent

**ID**: `cpt-cf-ticketing-actor-agent`

**Role**: Operator who receives, investigates, and resolves tickets. Communicates with reporters and updates ticket state.
**Needs**: Efficient ticket queue management, access to ticket details and history, ability to transition tickets through workflow states.

#### Manager

**ID**: `cpt-cf-ticketing-actor-manager`

**Role**: Oversees agents, reviews ticket resolution quality, handles escalations, and has elevated permissions for sensitive operations.
**Needs**: Visibility into all tickets within the organization, ability to reassign and override, aggregate views for workload management.

#### Administrator

**ID**: `cpt-cf-ticketing-actor-administrator`

**Role**: Configures the ticketing system — defines workflows, custom fields, projects, and manages organizational settings.
**Needs**: Workflow and field configuration tools, project management, audit log access, user permission oversight.

> Actor labels in this section are reference personas. Deployments may use different role names; transition authorization is governed by configurable workflow role keys and role mapping (see `cpt-cf-ticketing-fr-role-mapping`).

### 2.2 System Actors

#### Notification Service

**ID**: `cpt-cf-ticketing-actor-notification-service`

**Role**: External service that subscribes to domain events and delivers notifications to users via configured channels (email, webhook, push, etc.).

#### File Storage Service

**ID**: `cpt-cf-ticketing-actor-file-storage`

**Role**: External storage backend that persists and retrieves file attachments uploaded to tickets.

## 3. Operational Concept & Environment

### 3.1 Module-Specific Environment Constraints

- All data operations are scoped to an organization; the module enforces tenant isolation at the query level
- The workflow engine loads configuration data at runtime; changes to workflow definitions take effect without redeployment
- Domain events are published through an abstract port; the delivery mechanism is determined by the adapter

## 4. Scope

### 4.1 In Scope

- Ticket lifecycle management (create, read, update, delete)
- Configurable workflow engine (statuses and transitions as data)
- Custom fields system (per-project field definitions with runtime validation)
- Comments with internal/public visibility and sender type tracking
- File attachments with external storage and scan integration
- Immutable audit log for all state changes
- Domain event publishing for all significant operations
- Multi-tenant data isolation via organization scoping
- Project-based ticket grouping with per-project workflow and field schema
- Ticket assignment and reassignment
- Filtering, search, and aggregation queries
- Optimistic concurrency control for conflict detection
- Tag-based categorization
- Optional requester identity distinct from reporter
- Human-readable sequential ticket identifiers
- Configurable intra-organization visibility rules
- Field-level access control
- Ticket watchers (subscription to updates)
- Ticket relationship linking
- Bulk operations (batch assign, transition, update)

### 4.2 Out of Scope

- User interface implementation — the module exposes programmatic interfaces; UI is a consumer
- Authentication and identity management — the module receives authenticated user context; authentication mechanism is external
- Authorization policy engine — the module defines role requirements on transitions; enforcement is delegated to the platform
- File storage backend implementation — abstracted behind a port
- Notification delivery mechanism — abstracted behind domain events
- AI-powered analysis, suggestions, or autonomous resolution — these are separate modules that consume domain events and provide their own interfaces
- SLA calculation and enforcement — domain events provide hooks; SLA logic is domain-specific
- Knowledge base or documentation management
- Real-time chat or messaging beyond ticket comments
- Reporting and analytics dashboards — consumers can build these from domain events and query interfaces
- Safety-critical operations — this is a pure information system with no physical interaction, equipment control, or environmental impact
- Accessibility compliance (WCAG) — this is a backend library module; accessibility requirements apply to the presentation layer that consumes this module
- Internationalization of user-facing content — the module stores and returns text as-is; localization is a presentation-layer concern; workflow status labels may include localization keys
- Offline operation — this is a server-side module; offline capability is a client-side concern
- Specific data protection regulation compliance (GDPR, HIPAA, etc.) — the module provides audit logging, data isolation, and deletion capabilities that support compliance; specific regulatory requirements are configured at the deployment level

## 5. Functional Requirements

### 5.1 Ticket Management

#### Create Ticket

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-create-ticket`

The system **MUST** allow creation of a ticket with a title, description, priority, and project reference. Custom field values provided at creation time are validated against the project's field definitions. The ticket receives the initial status defined by the project's workflow configuration.

**Rationale**: Core capability — all domains require work item creation as the entry point for tracking.
**Actors**: `cpt-cf-ticketing-actor-reporter`, `cpt-cf-ticketing-actor-agent`

#### Read Ticket

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-read-ticket`

The system **MUST** allow retrieval of a single ticket with all its associated data: current field values, comments, attachment metadata, and audit history.

**Rationale**: Agents and reporters need complete ticket context for effective resolution.
**Actors**: `cpt-cf-ticketing-actor-reporter`, `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`

#### Update Ticket

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-update-ticket`

The system **MUST** allow updating ticket fields including title, description, priority, tags, custom field values, and assignee. Updates are subject to optimistic concurrency control. Custom field changes are validated against the project's field definitions.

**Rationale**: Ticket information evolves as investigation and resolution progress.
**Actors**: `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`

#### Delete Ticket

- [ ] `p2` - **ID**: `cpt-cf-ticketing-fr-delete-ticket`

The system **MUST** allow deletion of a ticket. Deletion is restricted by configurable policy (default: manager or administrator roles). A deletion action is always recorded in the audit log, and deleted tickets are removed from default operational queries while lifecycle policy determines archival and final purge.

**Rationale**: Enables cleanup of duplicate, spam, or erroneously created tickets while maintaining audit trail.
**Actors**: `cpt-cf-ticketing-actor-manager`, `cpt-cf-ticketing-actor-administrator`

#### Requester Identity

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-requester-identity`

The system **MUST** support an optional requester identity on a ticket, distinct from the reporter. The reporter is the authenticated user who created the ticket in the system; the requester is the external party on whose behalf the ticket was created. When not specified, the reporter is implicitly the requester.

**Rationale**: In service-desk, MSP, school, and housing domains, the person entering the ticket (agent, secretary, manager) is typically different from the affected party (customer, student, tenant). This distinction is required for correct routing, communication, and reporting.
**Actors**: `cpt-cf-ticketing-actor-reporter`, `cpt-cf-ticketing-actor-agent`

#### Human-Readable Ticket Identifiers

- [ ] `p2` - **ID**: `cpt-cf-ticketing-fr-readable-ticket-id`

The system **MUST** generate a human-readable, sequential ticket identifier per project or organization, in addition to the internal unique identifier. The identifier format (prefix, separator, number length) is configurable per project. When a project does not configure a prefix, the system uses `TKT` as the default prefix.

**Rationale**: Users reference tickets verbally, in emails, and in external systems. Human-readable sequential identifiers (e.g., "HELP-1042", "MAINT-0057") are essential for usability and are standard across all major ticketing systems (Jira uses project keys, Freshservice uses type-based prefixes like INC/SR/CASE, ServiceNow uses INC/CHG/REQ).
**Actors**: `cpt-cf-ticketing-actor-reporter`, `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`

#### List Tickets

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-list-tickets`

The system **MUST** allow listing tickets with filtering by status, priority, assignee, project, tags, and date range. Results are paginated and sortable by creation date, update date, and priority.

**Rationale**: Agents need filtered views to manage their workload; managers need organizational views for oversight.
**Actors**: `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`

#### Search Tickets

- [ ] `p2` - **ID**: `cpt-cf-ticketing-fr-search-tickets`

The system **MUST** support text-based search across ticket title and description fields within the current organization scope.

**Rationale**: Finding relevant tickets quickly reduces duplicate creation and supports knowledge reuse.
**Actors**: `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`

#### Aggregate Ticket Data

- [ ] `p2` - **ID**: `cpt-cf-ticketing-fr-aggregate-tickets`

The system **MUST** provide aggregation queries returning ticket counts grouped by status, priority, assignee, and project within the current organization scope.

**Rationale**: Managers need summary views for workload distribution, bottleneck identification, and reporting.
**Actors**: `cpt-cf-ticketing-actor-manager`

#### Bulk Operations

- [ ] `p2` - **ID**: `cpt-cf-ticketing-fr-bulk-operations`

The system **MUST** provide a dedicated batch interface for common bulk operations: bulk assign, bulk transition, and bulk field update. Each item in a batch is processed individually — producing its own audit log entry and domain event. Partial success is allowed; the batch response includes per-item status (success or failure with error detail).

**Rationale**: Agents and managers frequently need to act on multiple tickets simultaneously (e.g., reassign a departing agent's queue, bulk-close resolved tickets). A batch interface reduces network overhead while preserving per-ticket audit integrity. Industry practice (Jira bulk edit, ServiceNow bulk actions) confirms dedicated batch endpoints over generic wrappers.
**Actors**: `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`

### 5.2 Workflow Engine

#### Workflow Configuration

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-workflow-config`

The system **MUST** load workflow configuration defining statuses and transitions for each project. Workflow configuration specifies: status identifiers with display labels, terminal status flags, resolved status flags, and named transitions between statuses.

**Rationale**: Domain-agnostic core — different domains require different lifecycle definitions without code changes.
**Actors**: `cpt-cf-ticketing-actor-administrator`

#### Transition Ticket

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-transition-ticket`

The system **MUST** move a ticket from its current status to a new status when a valid transition is triggered. The system sets `resolved_at` when transitioning to a status marked as resolved, and `closed_at` when transitioning to a terminal status.

**Rationale**: Status transitions are the primary mechanism for tracking ticket lifecycle progress.
**Actors**: `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`, `cpt-cf-ticketing-actor-reporter`

#### Transition Role Guards

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-transition-guards`

The system **MUST** enforce role-based guards on transitions. Each transition specifies which roles are permitted to trigger it. A wildcard value permits any role. Role identifiers in transition definitions are configuration keys, not hardcoded role enums.

**Rationale**: Different domains require different permission models for status changes (e.g., only managers can reject, any agent can start work).
**Actors**: `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`

#### Role Mapping Configurability

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-role-mapping`

The system **MUST** evaluate transition permissions using configurable role keys defined in workflow configuration. Deployments **MUST** be able to map authenticated actor roles to these keys without changing ticketing module code.

**Rationale**: Prevents hardcoded role coupling and allows the same module to work across domains with different RBAC vocabularies.
**Actors**: `cpt-cf-ticketing-actor-administrator`

#### Transition Preconditions

- [ ] `p2` - **ID**: `cpt-cf-ticketing-fr-transition-conditions`

The system **MUST** evaluate preconditions before executing a transition. Standard condition types include: field not empty, field equals value, assignee is set, and actor is assignee or reporter. The condition system is extensible.

**Rationale**: Business rules often require specific data to be present before a status change is valid (e.g., resolution note required before resolving).
**Actors**: `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`

#### Initial Status Assignment

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-initial-status`

The system **MUST** assign the initial status defined in the project's workflow configuration when creating a new ticket.

**Rationale**: Ensures all tickets start in a consistent, workflow-defined state.
**Actors**: `cpt-cf-ticketing-actor-reporter`, `cpt-cf-ticketing-actor-agent`

### 5.3 Custom Fields

#### Define Custom Fields

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-define-fields`

The system **MUST** allow administrators to define custom fields per project. Each field definition specifies a field key, display label, data type, required flag, validation rules, default value, and display order.

**Rationale**: Domain-specific data (e.g., "equipment_id" for warehouse, "severity" for IT) must be captured without schema migration.
**Actors**: `cpt-cf-ticketing-actor-administrator`

#### Validate Custom Fields

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-validate-fields`

The system **MUST** validate custom field values against the project's field definitions when creating or updating a ticket. Required fields must be present, data types must match, and type-specific validation rules (e.g., regex, min/max, allowed options) must pass.

**Rationale**: Data integrity across domains requires runtime validation against configurable schemas.
**Actors**: `cpt-cf-ticketing-actor-reporter`, `cpt-cf-ticketing-actor-agent`

#### Standard Field Types

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-field-types`

The system **MUST** support the following custom field types: text, number, date, boolean, single-select, and multi-select.

**Rationale**: These types cover the majority of domain-specific data needs across verticals.
**Actors**: `cpt-cf-ticketing-actor-administrator`

#### Field-Level Access Control

- [ ] `p2` - **ID**: `cpt-cf-ticketing-fr-field-access-control`

The system **MUST** support configurable field-level access rules that restrict which roles can view or modify specific ticket fields or field values. Rules are defined per project and evaluated on create and update operations. Examples include restricting priority escalation to certain roles or making specific custom fields read-only for lower-level roles.

**Rationale**: Business processes commonly require field-level restrictions (e.g., only senior staff can set critical priority, only managers can modify certain classification fields). This complements transition-level role guards with finer-grained control.
**Actors**: `cpt-cf-ticketing-actor-administrator`

### 5.4 Comments & Communication

#### Add Comment

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-add-comment`

The system **MUST** allow adding a comment to a ticket with a text body and sender identification.

**Rationale**: Communication between reporters and agents is essential for ticket resolution.
**Actors**: `cpt-cf-ticketing-actor-reporter`, `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`

#### Internal Comments

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-internal-comments`

The system **MUST** support marking comments as internal. Internal comments are visible only to agents, managers, and administrators — not to reporters.

**Rationale**: Agents need a private channel for discussing sensitive details, escalation notes, or investigation findings.
**Actors**: `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`

#### Comment Sender Types

- [ ] `p2` - **ID**: `cpt-cf-ticketing-fr-comment-sender-types`

The system **MUST** track the sender type on each comment. Supported sender types are: reporter, agent, and system. System-type comments are generated by automated processes.

**Rationale**: Distinguishing comment origin enables proper display formatting, filtering, and analytics.
**Actors**: `cpt-cf-ticketing-actor-reporter`, `cpt-cf-ticketing-actor-agent`

#### Edit Comment

- [ ] `p3` - **ID**: `cpt-cf-ticketing-fr-edit-comment`

The system **MUST** allow the original author of a comment to edit its content within a configurable time window. Edits are tracked in the audit log with before/after values. The original creation timestamp is preserved; an `edited_at` timestamp is recorded. All actors who can view the comment see an "edited" indicator with the edit timestamp; full edit history (before/after diff) is available to managers and administrators via the audit log.

**Rationale**: Correcting typos, adding missing details, or clarifying information in recently posted comments avoids noise from correction follow-ups while maintaining audit integrity. Visibility of the "edited" indicator to all viewers follows industry standard practice (Jira, GitHub, Slack).
**Actors**: `cpt-cf-ticketing-actor-reporter`, `cpt-cf-ticketing-actor-agent`

### 5.5 Attachments

#### Upload Attachment

- [ ] `p2` - **ID**: `cpt-cf-ticketing-fr-upload-attachment`

The system **MUST** allow uploading file attachments to a ticket. File storage is delegated to an external storage port. Each attachment records the file name, size, MIME type, and uploader identity.

**Rationale**: Screenshots, logs, and documents are essential context for ticket resolution.
**Actors**: `cpt-cf-ticketing-actor-reporter`, `cpt-cf-ticketing-actor-agent`

#### Download Attachment

- [ ] `p2` - **ID**: `cpt-cf-ticketing-fr-download-attachment`

The system **MUST** allow downloading previously uploaded attachments for a ticket.

**Rationale**: All participants need access to attached files for investigation and resolution.
**Actors**: `cpt-cf-ticketing-actor-reporter`, `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`

#### Attachment Scanning

- [ ] `p3` - **ID**: `cpt-cf-ticketing-fr-attachment-scan`

The system **MUST** support delegating attachment scanning to an external scanning port. Scan status (pending, clean, flagged) is tracked per attachment.

**Rationale**: Uploaded files may contain malware; integration with security scanning protects users and infrastructure.
**Actors**: `cpt-cf-ticketing-actor-file-storage`

### 5.6 Audit & History

#### Audit Log

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-audit-log`

The system **MUST** record all ticket state changes as immutable audit log entries. Each entry captures the ticket ID, actor, action type, before/after values, and timestamp. Supported action types include: created, updated, status changed, assigned, comment added, and deleted.

**Rationale**: Complete audit trail is required for compliance, dispute resolution, and operational transparency.
**Actors**: `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`, `cpt-cf-ticketing-actor-administrator`

#### Query Audit Log

- [ ] `p2` - **ID**: `cpt-cf-ticketing-fr-audit-query`

The system **MUST** allow querying audit log entries for a specific ticket, ordered chronologically.

**Rationale**: Managers and administrators need to review the full history of changes for oversight and investigation.
**Actors**: `cpt-cf-ticketing-actor-manager`, `cpt-cf-ticketing-actor-administrator`

### 5.7 Domain Events

#### Publish Domain Events

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-domain-events`

The system **MUST** publish domain events through an abstract event bus port on all significant state changes. Each event includes the event type, ticket ID, actor, timestamp, and relevant change data.

**Rationale**: Event-driven architecture enables external systems (notifications, analytics, AI, SLA tracking) to extend functionality without modifying the ticketing core.
**Actors**: `cpt-cf-ticketing-actor-notification-service`

#### Standard Event Types

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-event-types`

The system **MUST** emit the following event types: `ticket.created`, `ticket.updated`, `ticket.transitioned`, `ticket.assigned`, `ticket.comment_added`, `ticket.comment_edited`, `ticket.attachment_added`, `ticket.deleted`, `ticket.watcher_added`, `ticket.watcher_removed`, and `ticket.linked`.

**Rationale**: Well-defined event types enable consumers to subscribe selectively to relevant changes.
**Actors**: `cpt-cf-ticketing-actor-notification-service`

### 5.8 Multi-Tenancy & Projects

#### Organization Isolation

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-org-isolation`

The system **MUST** scope all data operations to the current organization. No query or mutation may access data belonging to another organization.

**Rationale**: Multi-tenant deployments require strict data isolation for security and privacy.
**Actors**: `cpt-cf-ticketing-actor-administrator`

#### Intra-Organization Visibility

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-intra-org-visibility`

The system **MUST** support configurable visibility rules that restrict which tickets an actor can view and modify within their organization. Visibility rules are defined as part of project or role configuration and may restrict access based on assignment, project membership, requester identity, or other actor attributes. Default operational queries apply the actor's visibility scope automatically.

**Rationale**: Within an organization, different roles require different visibility — agents may see only their assigned tickets, managers see all tickets, and requesters see only their own. This is a universal business pattern across all domains.
**Actors**: `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`, `cpt-cf-ticketing-actor-administrator`

#### Project Scoping

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-project-scope`

The system **MUST** associate each ticket with exactly one project. The project determines the workflow configuration and custom field definitions applicable to its tickets.

**Rationale**: Different teams or domains within an organization need independent workflows and field schemas.
**Actors**: `cpt-cf-ticketing-actor-administrator`

#### Project Management

- [ ] `p2` - **ID**: `cpt-cf-ticketing-fr-project-management`

The system **MUST** allow creating, updating, and deactivating projects within an organization. Deactivated projects prevent new ticket creation but preserve existing data.

**Rationale**: Organizations need to manage multiple concurrent projects and retire completed ones.
**Actors**: `cpt-cf-ticketing-actor-administrator`

### 5.9 Assignment

#### Assign Ticket

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-assign-ticket`

The system **MUST** allow assigning a ticket to an agent. Assignment is recorded in the audit log and triggers a domain event.

**Rationale**: Assigning responsibility is fundamental to ticket resolution workflow.
**Actors**: `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`

#### Reassign Ticket

- [ ] `p2` - **ID**: `cpt-cf-ticketing-fr-reassign-ticket`

The system **MUST** allow reassigning a ticket to a different agent. Reassignment is restricted by role (configurable per workflow transition or by manager/administrator role).

**Rationale**: Workload balancing and escalation require the ability to transfer ticket ownership.
**Actors**: `cpt-cf-ticketing-actor-manager`, `cpt-cf-ticketing-actor-administrator`

### 5.10 Concurrency Control

#### Optimistic Concurrency

- [ ] `p1` - **ID**: `cpt-cf-ticketing-fr-optimistic-concurrency`

The system **MUST** implement optimistic concurrency control using a version counter on the ticket entity. Concurrent updates targeting the same version are rejected with a conflict error. The conflict response includes the current ticket state to enable client-side retry. Field-level merge for non-conflicting concurrent changes may be considered as a future enhancement but is not required for initial implementation.

**Rationale**: Multiple agents may access the same ticket simultaneously; version-based conflict detection prevents silent data loss. Strict reject is the industry standard default (AWS AppSync, EF Core, standard OCC) and is simpler to reason about than field-level merge.
**Actors**: `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`

### 5.11 Data Lifecycle & Retention

#### Retention and Archival Policy

- [ ] `p2` - **ID**: `cpt-cf-ticketing-fr-retention-policy`

The system **MUST** support configurable retention rules for tickets and audit records by organization and/or project. Retention rules define active retention duration, archival eligibility, and purge eligibility windows. Default retention: 12 months active (hot storage, queryable), then archive for up to 7 years (cold storage, retrievable on request). Deployments can override these defaults per organization.

**Rationale**: Different domains and regulators require different retention periods and archival behavior. Default periods are derived from compliance requirements: SOC 2/SOX require 7 years, HIPAA requires 6 years, PCI DSS 4.0 requires 12 months (3 months readily available), ISO 27001 recommends 12 months.
**Actors**: `cpt-cf-ticketing-actor-administrator`

#### Lifecycle Visibility in Queries

- [ ] `p2` - **ID**: `cpt-cf-ticketing-fr-lifecycle-visibility`

The system **MUST** distinguish active, archived, and deleted records in query behavior. Default operational queries exclude archived and deleted records unless explicitly requested by authorized actors.

**Rationale**: Keeps operational views focused while preserving historical discoverability for governance and investigations.
**Actors**: `cpt-cf-ticketing-actor-manager`, `cpt-cf-ticketing-actor-administrator`

### 5.12 Watchers

#### Watch Ticket

- [ ] `p3` - **ID**: `cpt-cf-ticketing-fr-watch-ticket`

The system **MUST** allow actors to subscribe to (watch) a ticket. Watched tickets include the subscriber's identity in domain event routing metadata for the relevant ticket events. Actors can unsubscribe at any time.

**Rationale**: Managers, team leads, and interested parties need to follow specific tickets without being assigned. Watchers are a standard pattern in ticketing systems for stakeholder notification.
**Actors**: `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`

### 5.13 Ticket Relationships

#### Link Tickets

- [ ] `p3` - **ID**: `cpt-cf-ticketing-fr-link-tickets`

The system **MUST** support configurable relationships between tickets. Relationship types (e.g., related, duplicate, parent/child, blocks/blocked-by) are defined as configuration. Each link is bidirectional and recorded in the audit log. For hierarchical (parent/child) relationships, the maximum nesting depth is configurable per project with a default of 3 levels and an upper bound of 5 levels.

**Rationale**: Duplicate detection, hierarchical work breakdown, and dependency tracking are common requirements across ticketing domains. Configurable relationship types allow each domain to define the link semantics it needs. Depth limits follow industry practice: Jira defaults to 3 levels (Epic → Story → Subtask), with Premium allowing up to ~9. Most organizations effectively use 2–4 levels; deeper hierarchies degrade recursive query performance and UX clarity.
**Actors**: `cpt-cf-ticketing-actor-agent`, `cpt-cf-ticketing-actor-manager`

## 6. Non-Functional Requirements

### 6.1 Module-Specific NFRs

#### Audit Immutability

- [ ] `p1` - **ID**: `cpt-cf-ticketing-nfr-audit-immutability`

Audit log entries **MUST** be append-only. No update or delete operations are permitted on audit records.

**Threshold**: Zero audit record modifications or deletions under any operation path
**Rationale**: Compliance and trust require tamper-proof audit trails; this is a module-specific invariant beyond project defaults
**Architecture Allocation**: See DESIGN.md § NFR Allocation

#### Tenant Data Isolation

- [ ] `p1` - **ID**: `cpt-cf-ticketing-nfr-tenant-isolation`

Data from one organization **MUST** never be accessible to users or queries from another organization.

**Threshold**: Zero cross-tenant data leakage under all query and mutation paths
**Rationale**: Multi-tenant architecture requires strict isolation guarantees
**Architecture Allocation**: See DESIGN.md § NFR Allocation

#### Query Latency

- [ ] `p1` - **ID**: `cpt-cf-ticketing-nfr-query-latency`

Ticket list queries with standard filters **MUST** complete within 200ms at p95 for datasets up to 100,000 tickets per organization.

**Threshold**: p95 latency ≤ 200ms for filtered list queries
**Rationale**: Agent productivity depends on fast ticket queue rendering; stricter than generic project default
**Architecture Allocation**: See DESIGN.md § NFR Allocation

#### Write Latency

- [ ] `p1` - **ID**: `cpt-cf-ticketing-nfr-write-latency`

Ticket create and update operations **MUST** complete within 100ms at p95 excluding external port calls (file storage, event delivery).

**Threshold**: p95 latency ≤ 100ms for core write path
**Rationale**: Fast write acknowledgment is critical for responsive user experience
**Architecture Allocation**: See DESIGN.md § NFR Allocation

#### Event Delivery Reliability

- [ ] `p2` - **ID**: `cpt-cf-ticketing-nfr-event-delivery`

Domain events **MUST** be delivered with at-least-once semantics. Events include idempotency metadata to support deduplication by consumers.

**Threshold**: Zero event loss under normal operation; consumers receive all events at least once
**Rationale**: Downstream systems (notifications, SLA, analytics) depend on complete event streams
**Architecture Allocation**: See DESIGN.md § NFR Allocation

#### Concurrent Operations

- [ ] `p2` - **ID**: `cpt-cf-ticketing-nfr-concurrent-ops`

The system **MUST** support at least 50 concurrent ticket write operations per organization without degradation.

**Threshold**: 50 concurrent writes per org, p95 latency within write latency SLO
**Rationale**: Peak operational periods generate bursts of concurrent ticket activity
**Architecture Allocation**: See DESIGN.md § NFR Allocation

#### Availability

- [ ] `p1` - **ID**: `cpt-cf-ticketing-nfr-availability`

Core ticket operations **MUST** achieve monthly availability of at least 99.9%, excluding planned maintenance windows.

**Threshold**: Monthly availability ≥ 99.9%
**Rationale**: Ticketing is an operational system used continuously by support and operations teams
**Architecture Allocation**: See DESIGN.md § NFR Allocation

#### Recovery Objectives

- [ ] `p2` - **ID**: `cpt-cf-ticketing-nfr-recovery-objectives`

After a major service disruption, the system **MUST** restore core ticket operations within an RTO of 4 hours and with an RPO no worse than 15 minutes.

**Threshold**: RTO ≤ 4h; RPO ≤ 15m
**Rationale**: Operational continuity requires bounded downtime and bounded recent data loss in disaster scenarios
**Architecture Allocation**: See DESIGN.md § NFR Allocation

#### Optional Extension Profiles

- [ ] `p2` - **ID**: `cpt-cf-ticketing-nfr-optional-profiles`

The module **MUST** support a core profile that operates without SLA enforcement or inbound asynchronous command processing. Optional profiles can add SLA logic and async intake without changing core ticket lifecycle semantics.

**Threshold**: Core acceptance criteria remain satisfiable with optional profiles disabled
**Rationale**: Not all vendors or domains require SLA or async command ingestion
**Architecture Allocation**: See DESIGN.md § NFR Allocation

#### Async Command Idempotency (Optional Profile)

- [ ] `p2` - **ID**: `cpt-cf-ticketing-nfr-async-idempotency`

When asynchronous command intake is enabled, repeated delivery of the same command **MUST NOT** produce duplicate effective ticket mutations.

**Threshold**: Duplicate command deliveries result in at most one effective state mutation
**Rationale**: Message transports are commonly at-least-once and require idempotent handling for correctness
**Architecture Allocation**: See DESIGN.md § NFR Allocation

#### Sensitive Data Handling

- [ ] `p2` - **ID**: `cpt-cf-ticketing-nfr-sensitive-data`

Ticket fields, comments, and audit payloads **MUST** support marking as PII-sensitive through field definitions. Domain event and audit log payloads **MUST** respect deployment-level redaction policies for sensitive data, omitting or masking PII fields when configured.

**Threshold**: Zero unredacted PII fields in event/audit payloads when redaction policy is active for those fields
**Rationale**: Ticket data frequently contains personally identifiable information; downstream consumers (analytics, logging) may operate in contexts where PII must be redacted
**Architecture Allocation**: See DESIGN.md § NFR Allocation

### 6.2 NFR Exclusions

- **Offline support**: Not applicable — this is a server-side module; offline capability is a client-side concern
- **Real-time sync**: Not applicable — real-time UI updates are a presentation-layer concern; domain events provide the data foundation

## 7. Public Library Interfaces

### 7.1 Public API Surface

#### Ticket Management Interface

- [ ] `p1` - **ID**: `cpt-cf-ticketing-interface-ticket-mgmt`

**Type**: Rust module (traits and types)
**Stability**: stable
**Description**: Core interface for ticket CRUD operations, workflow transitions, assignment, and queries. Exposes use case handlers for all ticket management operations.
**Breaking Change Policy**: Major version bump required for trait signature changes or removal of operations

#### Workflow Configuration Interface

- [ ] `p1` - **ID**: `cpt-cf-ticketing-interface-workflow-config`

**Type**: Rust module (traits and types)
**Stability**: stable
**Description**: Interface for loading and managing workflow configurations. Provides the workflow engine contract for status validation and transition evaluation.
**Breaking Change Policy**: Major version bump for changes to workflow configuration structure or engine contract

#### Custom Field Schema Interface

- [ ] `p1` - **ID**: `cpt-cf-ticketing-interface-field-schema`

**Type**: Rust module (traits and types)
**Stability**: stable
**Description**: Interface for managing custom field definitions per project and validating field values against definitions.
**Breaking Change Policy**: Major version bump for field type changes; new field types are minor changes

#### Domain Event Bus Interface

- [ ] `p1` - **ID**: `cpt-cf-ticketing-interface-event-bus`

**Type**: Rust trait
**Stability**: stable
**Description**: Abstract port for publishing domain events. Consumers implement this trait to receive event notifications. Events are typed with a standard envelope containing event type, ticket ID, actor, timestamp, and payload.
**Breaking Change Policy**: New event types are minor changes; changes to event envelope are major

### 7.2 External Integration Contracts

#### File Storage Contract

- [ ] `p2` - **ID**: `cpt-cf-ticketing-contract-file-storage`

**Direction**: required from client (external storage backend)
**Protocol/Format**: Programmatic interface — store, retrieve, delete operations with byte streams
**Compatibility**: Adapter-specific; interface contract is stable

#### Notification Delivery Contract

- [ ] `p2` - **ID**: `cpt-cf-ticketing-contract-notification`

**Direction**: provided by library (via domain events)
**Protocol/Format**: Domain events published through event bus interface; consumers subscribe and deliver notifications via their preferred channel
**Compatibility**: Event schema versioned; backward-compatible additions are minor changes

#### SLA Policy Contract (Optional Profile)

- [ ] `p3` - **ID**: `cpt-cf-ticketing-contract-sla-policy`

**Direction**: optional integration — library provides events and query/update interfaces; client provides SLA policy enforcement logic
**Protocol/Format**: SLA consumer subscribes to ticket events and can invoke ticket operations for due-date updates, breach marking, and escalation actions
**Compatibility**: Optional profile; absence of SLA consumer must not affect core ticket lifecycle operations

#### Async Command Intake Contract (Optional Profile)

- [ ] `p3` - **ID**: `cpt-cf-ticketing-contract-async-intake`

**Direction**: optional integration — client provides message transport delivering ticketing commands
**Protocol/Format**: Commands carry idempotency identity and actor context required for authorization and audit attribution
**Compatibility**: Optional profile; synchronous ticket interfaces remain canonical regardless of async intake availability

#### Attachment Scan Contract

- [ ] `p3` - **ID**: `cpt-cf-ticketing-contract-attachment-scan`

**Direction**: required from client (external scanning service)
**Protocol/Format**: Programmatic interface — submit file for scan, receive scan result (clean/flagged)
**Compatibility**: Interface contract is stable; scan result types may be extended

## 8. Use Cases

#### Create a New Ticket

- [ ] `p1` - **ID**: `cpt-cf-ticketing-usecase-create-ticket`

**Actor**: `cpt-cf-ticketing-actor-reporter`

**Preconditions**:
- Actor is authenticated and authorized within an organization
- Target project exists and is active

**Main Flow**:
1. Actor submits ticket data: title, description, priority, project, optional requester identity, and optional custom field values
2. System validates required fields and custom field values against the project's field definitions
3. System assigns the initial status from the project's workflow configuration
4. System persists the ticket and creates an audit log entry
5. System publishes a `ticket.created` domain event
6. System returns the created ticket with its assigned ID and initial status

**Postconditions**:
- Ticket exists with the project's initial status
- Audit log contains a `ticket_created` entry
- Domain event `ticket.created` has been published

**Alternative Flows**:
- **Validation fails**: System returns validation errors listing each invalid field; no ticket is created
- **Project is inactive**: System rejects creation with an appropriate error

#### Transition Ticket Status

- [ ] `p1` - **ID**: `cpt-cf-ticketing-usecase-transition-ticket`

**Actor**: `cpt-cf-ticketing-actor-agent`

**Preconditions**:
- Ticket exists and is accessible to the actor
- Actor's role is included in the transition's allowed roles

**Main Flow**:
1. Actor requests a named transition on a ticket
2. System loads the project's workflow configuration
3. System verifies the transition is valid from the current status
4. System checks the actor's role against the transition's role guard
5. System evaluates all transition preconditions
6. System updates the ticket status, sets resolved/closed timestamps if applicable
7. System records the status change in the audit log
8. System publishes a `ticket.transitioned` domain event

**Postconditions**:
- Ticket status is updated to the transition's target status
- Audit log contains a `status_changed` entry
- Domain event `ticket.transitioned` has been published

**Alternative Flows**:
- **Invalid transition**: System rejects with an error indicating the transition is not valid from the current status
- **Role not permitted**: System rejects with a forbidden error
- **Preconditions not met**: System returns the list of unmet preconditions

#### Assign Ticket to Agent

- [ ] `p1` - **ID**: `cpt-cf-ticketing-usecase-assign-ticket`

**Actor**: `cpt-cf-ticketing-actor-manager`

**Preconditions**:
- Ticket exists and is not in a terminal status
- Target agent belongs to the same organization

**Main Flow**:
1. Actor selects a ticket and specifies the target agent
2. System updates the ticket's assignee
3. System records the assignment in the audit log
4. System publishes a `ticket.assigned` domain event

**Postconditions**:
- Ticket assignee is updated
- Audit log contains an `assigned` entry
- Domain event `ticket.assigned` has been published

**Alternative Flows**:
- **Ticket in terminal status**: System rejects the assignment
- **Concurrency conflict**: System rejects with a conflict error; actor must reload the ticket

#### Add Comment to Ticket

- [ ] `p1` - **ID**: `cpt-cf-ticketing-usecase-add-comment`

**Actor**: `cpt-cf-ticketing-actor-agent`

**Preconditions**:
- Ticket exists and is accessible to the actor

**Main Flow**:
1. Actor submits a comment body and internal/public flag
2. System records the comment with sender type, timestamp, and visibility flag
3. System updates the ticket's last modified timestamp
4. System records the comment in the audit log
5. System publishes a `ticket.comment_added` domain event

**Postconditions**:
- Comment is persisted and associated with the ticket
- Audit log contains a `comment_added` entry
- Domain event `ticket.comment_added` has been published

**Alternative Flows**:
- **Empty body**: System rejects the comment with a validation error
- **Reporter submitting internal comment**: System rejects — reporters cannot create internal comments

#### Search and Filter Tickets

- [ ] `p2` - **ID**: `cpt-cf-ticketing-usecase-search-tickets`

**Actor**: `cpt-cf-ticketing-actor-agent`

**Preconditions**:
- Actor is authenticated within an organization

**Main Flow**:
1. Actor specifies filter criteria: status, priority, assignee, project, tags, date range, and/or text search query
2. System executes the query within the actor's organization scope
3. System returns paginated, sorted results with ticket summary data

**Postconditions**:
- Actor receives a filtered list of tickets matching the criteria

**Alternative Flows**:
- **No results**: System returns an empty result set
- **Invalid filter values**: System returns validation errors

#### Configure Project Workflow

- [ ] `p2` - **ID**: `cpt-cf-ticketing-usecase-configure-workflow`

**Actor**: `cpt-cf-ticketing-actor-administrator`

**Preconditions**:
- Actor has administrator role within the organization
- Project exists

**Main Flow**:
1. Administrator defines or updates workflow configuration: statuses, transitions, role guards, and preconditions
2. System validates the workflow configuration (no orphan statuses, no missing transition targets, initial status exists)
3. System persists the workflow configuration
4. System applies the updated workflow to future ticket operations within the project

**Postconditions**:
- Project's workflow configuration is updated
- Existing tickets retain their current status; new transitions are governed by the updated workflow

**Alternative Flows**:
- **Invalid configuration**: System returns validation errors (e.g., transition references non-existent status)
- **Active tickets in removed status**: System rejects the configuration change and lists affected tickets

## 9. Acceptance Criteria

- [ ] A ticket can be created with all required fields and receives the project's initial workflow status
- [ ] Custom field values are validated against the project's field definitions on create and update
- [ ] Workflow transitions enforce role guards and preconditions; invalid transitions are rejected
- [ ] Comments support internal/public visibility with correct access control
- [ ] All ticket state changes produce immutable audit log entries
- [ ] Domain events are published for all significant operations (created, updated, transitioned, assigned, commented, deleted)
- [ ] Organization isolation is enforced — no cross-tenant data access under any operation
- [ ] Concurrent updates to the same ticket are detected and rejected via optimistic concurrency control
- [ ] Workflow configurations can be modified without code changes or redeployment
- [ ] New domains can be supported by providing workflow configuration and field definitions alone
- [ ] Transition permissions can be configured with domain-specific role keys without hardcoding module roles
- [ ] Retention policy controls active visibility, archival eligibility, and purge eligibility for ticket data
- [ ] Core lifecycle remains fully functional when optional SLA and async command profiles are disabled
- [ ] Availability and recovery objectives (99.9% monthly availability, RTO/RPO targets) are defined and verifiable for core operations
- [ ] An optional requester identity can be set on a ticket, distinct from the reporter who created it
- [ ] Human-readable sequential ticket identifiers are generated per project with configurable format
- [ ] Intra-organization visibility rules restrict ticket access based on actor role, assignment, and configuration
- [ ] Field-level access rules can restrict which roles may view or modify specific fields or values
- [ ] Domain event and audit payloads respect PII redaction policies when configured
- [ ] Actors can watch/unwatch tickets and receive relevant event routing metadata
- [ ] Tickets can be linked with configurable relationship types; links are bidirectional and audited
- [ ] Bulk operations process each item individually with per-item audit entries, domain events, and error reporting

## 10. Dependencies

| Dependency | Description | Criticality |
|------------|-------------|-------------|
| Authentication Service | Provides authenticated user context (user ID, organization ID, roles) | p1 |
| Platform Runtime | CyberFabric module lifecycle and service discovery | p1 |
| Persistent Storage | Database for ticket, audit, and configuration persistence | p1 |
| File Storage Service | External storage for ticket attachments | p2 |
| Event Delivery Infrastructure | Transport for domain events (in-process or distributed) | p2 |
| Archival Storage (optional) | Stores archived ticket and audit data for long-term retention policies | p3 |
| SLA Policy Consumer (optional) | Consumes ticket events and applies SLA deadlines/escalations outside core lifecycle | p3 |
| Async Command Transport (optional) | Delivers asynchronous ticket commands when optional async profile is enabled | p3 |

## 11. Assumptions

- The hosting platform provides authenticated user context including user ID, organization ID, and role assignments
- A persistent storage backend is available and supports transactional operations with optimistic locking
- The module operates within a single deployment region; cross-region replication is handled at the infrastructure level
- Workflow configurations are relatively stable and change infrequently (not on a per-request basis)
- Custom field definitions per project are bounded at a supported maximum of 100 per project (soft warning at 50). This is conservative relative to industry limits — Jira Cloud enforces 700 fields per space — and ensures predictable query performance at scale. Performance testing should validate at 100 fields × 100k tickets per organization
- The hosting platform handles data protection regulation compliance (GDPR, HIPAA, etc.); this module provides building blocks (audit log, data isolation, deletion) but does not implement regulatory workflows
- If optional SLA or async command profiles are enabled, external consumers provide policy logic and transport reliability guarantees

## 12. Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| Workflow misconfiguration leaves tickets in unreachable states | Tickets cannot progress to resolution | Validate workflow completeness at configuration time; check all non-terminal statuses have at least one outbound transition |
| Custom field schema evolution breaks existing ticket data | Historical data fails validation or becomes unreadable | Field definitions support deactivation (hidden but data preserved); new required fields include default values |
| Optimistic concurrency failures frustrate users during high-contention periods | Repeated conflict errors degrade user experience | Strict reject with current state in conflict response for client retry; field-level merge may be considered in future phases for non-conflicting changes |
| Event bus port saturation under high load | Downstream consumers miss events or back-pressure slows ticket operations | Design event publishing as non-blocking; document at-least-once semantics; provide replay capability in future phases |
| Multi-tenant isolation bypass through query injection | Cross-organization data exposure | Enforce organization scoping at the repository port level; automated testing for tenant isolation |

## 13. Traceability

- **Design**: [DESIGN.md](./DESIGN.md)
- **ADRs**: [ADR/](./ADR/)
- **Features**: [features/](./features/)
