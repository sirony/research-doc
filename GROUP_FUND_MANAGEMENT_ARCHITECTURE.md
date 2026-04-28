# Group Fund Management — Full Architecture Document

**Project:** PayToMe API (Laravel 10, PHP 8.2+)
**Feature:** Collaborative Group Fund & Shared Investment Management
**Author:** Architecture Design
**Date:** 2026-04-20
**Status:** Design Phase

---

## Table of Contents

1. [Overview](#1-overview)
2. [Goals & Scope](#2-goals--scope)
3. [How It Fits the Existing System](#3-how-it-fits-the-existing-system)
4. [Module Boundaries & Layers](#4-module-boundaries--layers)
5. [Domain Model & Entities](#5-domain-model--entities)
6. [Database Schema](#6-database-schema)
7. [Business Rules](#7-business-rules)
8. [Service Layer Design](#8-service-layer-design)
9. [API Endpoints](#9-api-endpoints)
10. [Event Flow & Jobs](#10-event-flow--jobs)
11. [Permissions & Roles](#11-permissions--roles)
12. [File & Directory Structure](#12-file--directory-structure)
13. [Migration Order](#13-migration-order)
14. [Key Workflows](#14-key-workflows)
15. [Edge Cases & Error Handling](#15-edge-cases--error-handling)
16. [Future Extension Points](#16-future-extension-points)

---

## 1. Overview

This document describes the architecture for adding a **Group Fund Management** layer to the PayToMe platform.

The feature allows a company or a set of contacts to form a **funding group**, define share-based contribution obligations, pool collected money into a **group fund ledger**, and **allocate** those funds to one or more **projects or assets** (land, vehicle, business expansion, etc.).

**Architectural boundary:** PayToMe already has standalone `Invoice` and `Payment` modules that handle billing issuance and money collection. This new module does **not** duplicate those concerns. Instead it acts as the **business rules / contribution obligation layer** that feeds the existing Invoice and Payment modules.

```
Group Fund Module           →    Invoice Module    →    Payment Module
(who owes what & when)           (bill issuance)        (money collection)
```

This module is layered on top of the existing:

- `Contact` — identity of participants
- `CompanyGroups` — organizational grouping (adapted/extended)
- `User` / `UserHasCompany` — authentication and tenancy
- `Invoice` — bill generation and tracking (existing, reused)
- `Payment` / `PaymentGateway` — payment collection (existing, reused)

---

## 2. Goals & Scope

### In Scope (this module)

| Capability | Description |
|---|---|
| Fund Group creation | Create a named group with contribution rules |
| Member management | Add/invite contacts as members with share counts |
| Contribution plan | Define share rate, frequency, grace period, penalty |
| Billing cycles | Auto-generate monthly contribution obligations per member |
| Invoice generation | Create invoices (via existing Invoice module) from contribution obligations |
| Ledger crediting | Credit group fund ledger when Invoice/Payment module records a payment |
| Group ledger | Double-entry-style fund ledger per group |
| Project creation | Create projects under a group with a funding goal |
| Fund allocation | Move money from group ledger to a project |
| Ownership tracking | Calculate each member's ownership % per project |
| Approval workflow | Governance for allocations and project creation |
| Audit trail | Every financial action is logged |

### Out of Scope (handled by existing modules)

| Concern | Handled by |
|---|---|
| Invoice issuance, due-date tracking, invoice status | Existing `Invoice` module |
| Payment collection (manual, Stripe, bKash, etc.) | Existing `Payment` module |
| Payment gateway configuration | Existing `PaymentGateway` module |

### Out of Scope (v1, future)

- Profit/dividend distribution engine — modelled but not automated
- Voting/quorum engine — approval is sequential, not vote-counted
- Multi-currency per group — single currency per group

---

## 3. How It Fits the Existing System

```
┌──────────────────────────────────────────────────────────────────────┐
│                          PayToMe Platform                             │
│                                                                      │
│  ┌──────────────┐   ┌───────────────┐                                │
│  │   Contact    │   │ CompanyGroups │                                │
│  │  (identity)  │   │  (grouping)   │                                │
│  └──────┬───────┘   └───────┬───────┘                                │
│         └──────────┬────────┘                                        │
│                    │                                                 │
│      ┌─────────────▼───────────────────────────────────┐            │
│      │        GROUP FUND MANAGEMENT MODULE (NEW)         │            │
│      │                                                   │            │
│      │  FundGroup → Membership → ContributionPlan        │            │
│      │  BillingCycle → ContributionDue                   │            │
│      │  GroupLedger → FundProject → FundAllocation       │            │
│      │  ProjectOwnership → ApprovalRequest               │            │
│      └──────────┬──────────────────────┬────────────────┘            │
│                 │  generates invoices  │  credits ledger              │
│                 │  via Invoice module  │  on payment events           │
│                 ▼                      ▼                              │
│      ┌──────────────────┐   ┌──────────────────────────┐             │
│      │  Invoice Module  │──▶│     Payment Module        │             │
│      │  (EXISTING)      │   │     (EXISTING)            │             │
│      │  bill issuance   │   │  collection / gateways   │             │
│      └──────────────────┘   └──────────────────────────┘             │
└──────────────────────────────────────────────────────────────────────┘
```

### Existing Assets Reused

| Existing | Role in New Feature |
|---|---|
| `Contact` | Fund group member identity |
| `CompanyGroups` | Optional: map a fund group to an existing company group |
| `User` / company tenancy | Multi-tenant isolation — each fund group belongs to a company |
| `Invoice` model | Invoices generated from `ContributionDue` records; `contribution_dues.invoice_id` FK |
| `Payment` model | Payments collected against invoices; payment events credit the group ledger |
| `spatie/laravel-permission` | Role & permission enforcement |
| `owen-it/laravel-auditing` | Audit trail on all financial models |
| Queue / Horizon | Billing cycle generation and invoice generation jobs |
| Notification system | Due reminders, approval requests |

---

## 4. Module Boundaries & Layers

```
app/
└── Domains/
    └── FundGroup/
        ├── Entities/           ← Pure domain objects (no Eloquent)
        ├── Models/             ← Eloquent models
        ├── Services/           ← Business logic
        ├── Repositories/       ← Data access interfaces + implementations
        ├── DTOs/               ← Data Transfer Objects
        ├── Events/             ← Domain events
        ├── Listeners/          ← Event listeners
        ├── Jobs/               ← Queue jobs (billing cycle generation)
        ├── Policies/           ← Authorization policies
        ├── Enums/              ← Status, type enumerations
        └── Http/
            ├── Controllers/    ← API controllers (registered in routes/)
            ├── Requests/       ← Form request validation
            └── Resources/      ← API response transformers
```

---

## 5. Domain Model & Entities

### Entity Relationship (logical)

```
Company
  └── FundGroup (1..n)
        ├── ContributionPlan (1)
        ├── FundGroupMember (1..n)
        │     └── MemberShareHolding (1)
        ├── BillingCycle (1..n per month)
        │     └── ContributionDue (1 per member per cycle)
        │           └── Invoice (1) ← existing Invoice module
        │                 └── Payment (0..n) ← existing Payment module
        ├── GroupLedgerEntry (1..n)  ← credited on payment events
        ├── FundProject (0..n)
        │     ├── FundAllocation (0..n) ← from GroupLedger
        │     ├── ProjectOwnership (1 per active member)
        │     └── ApprovalRequest (0..n)
        └── ApprovalRequest (0..n) ← for group-level decisions
```

### Core Entities Described

#### FundGroup
The top-level collaborative unit. One company can have many fund groups. Each group has its own fund ledger, contribution plan, and projects.

#### FundGroupMember
Links a `Contact` (and optionally a `User`) to a `FundGroup` with a role, status, and governance flags.

#### ContributionPlan
Defines the financial rules for a group: how often to collect, how much per share, grace period, late penalty calculation method.

#### MemberShareHolding
How many shares a specific member holds within a group at a given time. Supports history (effective_from / effective_to) for share count changes.

#### BillingCycle
A time-bounded collection period (e.g., April 2026). Generated automatically by a job or manually. Contains the expected totals for the group.

#### ContributionDue
One record per member per billing cycle. The obligation — what that member owes for that cycle including any carry-forward arrears. When generated, an `Invoice` is created in the existing Invoice module and its ID is stored on this record. Payment status is tracked via that invoice.

#### GroupLedgerEntry
Double-entry style accounting line for the group's fund. Every financial movement creates at least one ledger entry. Credits are triggered by payment events from the existing Payment module (via event listener). Debits are triggered by fund allocations.

#### FundProject
A named goal (land, vehicle, business) under a group. Has a funding target, status, and ownership rule.

#### FundAllocation
Transfer of money from the group ledger to a specific project. Requires approval.

#### ProjectOwnership
Calculated or manually set ownership percentage per member per project. Used for future settlement or distribution.

#### ApprovalRequest
A governance record — requests approval for creating a project, allocating funds, adding a member, or removing a member. Tracks approvers and their decisions.

---

## 6. Database Schema

### 6.1 `fund_groups`

```sql
fund_groups
  id                    BIGINT UNSIGNED PK
  uuid                  CHAR(36) UNIQUE
  company_id            BIGINT UNSIGNED FK→companies
  created_by            BIGINT UNSIGNED FK→users
  name                  VARCHAR(191)
  description           TEXT NULL
  currency              CHAR(3) DEFAULT 'BDT'
  status                ENUM('active','paused','closed') DEFAULT 'active'
  requires_approval     BOOLEAN DEFAULT FALSE
  logo                  VARCHAR(500) NULL
  settings              JSON NULL        -- extensible config
  created_at, updated_at, deleted_at
```

**Indexes:** `company_id`, `uuid`, `status`

---

### 6.2 `fund_group_members`

```sql
fund_group_members
  id                    BIGINT UNSIGNED PK
  uuid                  CHAR(36) UNIQUE
  fund_group_id         BIGINT UNSIGNED FK→fund_groups
  contact_id            BIGINT UNSIGNED FK→contacts NULL   -- may not have contact
  user_id               BIGINT UNSIGNED FK→users NULL      -- if registered user
  invited_by            BIGINT UNSIGNED FK→users NULL
  role                  ENUM('owner','admin','treasurer','auditor','member','viewer')
  status                ENUM('invited','active','suspended','exited') DEFAULT 'invited'
  joined_at             TIMESTAMP NULL
  exited_at             TIMESTAMP NULL
  can_vote              BOOLEAN DEFAULT FALSE
  can_manage_funds      BOOLEAN DEFAULT FALSE
  can_approve           BOOLEAN DEFAULT FALSE
  notes                 TEXT NULL
  created_at, updated_at, deleted_at
```

**Indexes:** `fund_group_id`, `contact_id`, `user_id`, `status`
**Unique:** `(fund_group_id, contact_id)` where contact_id not null

---

### 6.3 `contribution_plans`

```sql
contribution_plans
  id                    BIGINT UNSIGNED PK
  uuid                  CHAR(36) UNIQUE
  fund_group_id         BIGINT UNSIGNED FK→fund_groups UNIQUE
  contribution_type     ENUM('share_based','fixed','percentage','flexible') DEFAULT 'share_based'
  share_unit_amount     DECIMAL(15,2) DEFAULT 0      -- amount per share per cycle
  fixed_amount          DECIMAL(15,2) DEFAULT 0      -- used when type=fixed
  frequency             ENUM('monthly','quarterly','yearly') DEFAULT 'monthly'
  start_date            DATE
  grace_days            TINYINT UNSIGNED DEFAULT 0
  penalty_type          ENUM('none','fixed','percentage') DEFAULT 'none'
  penalty_value         DECIMAL(10,2) DEFAULT 0
  auto_generate_cycles  BOOLEAN DEFAULT TRUE
  notes                 TEXT NULL
  created_at, updated_at
```

---

### 6.4 `member_share_holdings`

```sql
member_share_holdings
  id                    BIGINT UNSIGNED PK
  fund_group_member_id  BIGINT UNSIGNED FK→fund_group_members
  number_of_shares      DECIMAL(8,2) DEFAULT 1       -- supports fractional shares
  monthly_expected      DECIMAL(15,2)                -- computed: shares * unit_amount
  effective_from        DATE
  effective_to          DATE NULL                    -- null = currently active
  changed_by            BIGINT UNSIGNED FK→users NULL
  reason                TEXT NULL
  created_at, updated_at
```

**Index:** `(fund_group_member_id, effective_to)`

---

### 6.5 `billing_cycles`

```sql
billing_cycles
  id                    BIGINT UNSIGNED PK
  uuid                  CHAR(36) UNIQUE
  fund_group_id         BIGINT UNSIGNED FK→fund_groups
  cycle_year            SMALLINT UNSIGNED
  cycle_month           TINYINT UNSIGNED             -- 1–12
  period_label          VARCHAR(50)                  -- e.g. "April 2026"
  due_date              DATE
  expected_total        DECIMAL(15,2) DEFAULT 0      -- sum of all member dues
  collected_total       DECIMAL(15,2) DEFAULT 0      -- updated by payments
  status                ENUM('upcoming','open','closed','partially_collected') DEFAULT 'upcoming'
  generated_at          TIMESTAMP NULL
  closed_at             TIMESTAMP NULL
  created_at, updated_at
```

**Unique:** `(fund_group_id, cycle_year, cycle_month)`

---

### 6.6 `contribution_dues`

```sql
contribution_dues
  id                    BIGINT UNSIGNED PK
  uuid                  CHAR(36) UNIQUE
  billing_cycle_id      BIGINT UNSIGNED FK→billing_cycles
  fund_group_member_id  BIGINT UNSIGNED FK→fund_group_members
  share_snapshot        DECIMAL(8,2)                 -- shares at time of generation
  expected_amount       DECIMAL(15,2)
  paid_amount           DECIMAL(15,2) DEFAULT 0
  arrear_carried        DECIMAL(15,2) DEFAULT 0      -- unpaid from previous cycle
  advance_used          DECIMAL(15,2) DEFAULT 0      -- from member advance balance
  penalty_amount        DECIMAL(15,2) DEFAULT 0
  due_amount            DECIMAL(15,2)                -- computed: expected - paid + penalty - advance
  status                ENUM('unpaid','partial','paid','overdue','waived') DEFAULT 'unpaid'
  waived_by             BIGINT UNSIGNED FK→users NULL
  waived_reason         TEXT NULL
  created_at, updated_at
```

**Unique:** `(billing_cycle_id, fund_group_member_id)`

---

### 6.7 `contribution_payments`

```sql
contribution_payments
  id                    BIGINT UNSIGNED PK
  uuid                  CHAR(36) UNIQUE
  fund_group_id         BIGINT UNSIGNED FK→fund_groups
  fund_group_member_id  BIGINT UNSIGNED FK→fund_group_members
  contribution_due_id   BIGINT UNSIGNED FK→contribution_dues NULL  -- null if advance
  amount                DECIMAL(15,2)
  payment_type          ENUM('regular','advance','penalty','refund','adjustment')
  payment_method        VARCHAR(100) NULL            -- cash, bank_transfer, mobile_banking, etc.
  reference_number      VARCHAR(191) NULL
  paid_at               DATE
  collected_by          BIGINT UNSIGNED FK→users NULL
  notes                 TEXT NULL
  payment_id            BIGINT UNSIGNED FK→payments NULL  -- link to existing Payment model if used
  created_at, updated_at, deleted_at
```

---

### 6.8 `group_ledger_entries`

```sql
group_ledger_entries
  id                    BIGINT UNSIGNED PK
  uuid                  CHAR(36) UNIQUE
  fund_group_id         BIGINT UNSIGNED FK→fund_groups
  entry_type            ENUM('credit','debit')
  amount                DECIMAL(15,2)
  running_balance       DECIMAL(15,2)               -- maintained by service layer
  source_type           VARCHAR(100)                -- contribution_payment / allocation / refund / adjustment
  source_id             BIGINT UNSIGNED NULL
  description           VARCHAR(500)
  entry_date            DATE
  recorded_by           BIGINT UNSIGNED FK→users
  created_at, updated_at
```

**Indexes:** `fund_group_id`, `entry_date`, `source_type`

---

### 6.9 `fund_projects`

```sql
fund_projects
  id                    BIGINT UNSIGNED PK
  uuid                  CHAR(36) UNIQUE
  fund_group_id         BIGINT UNSIGNED FK→fund_groups
  created_by            BIGINT UNSIGNED FK→users
  approved_by           BIGINT UNSIGNED FK→users NULL
  title                 VARCHAR(191)
  description           TEXT NULL
  project_type          ENUM('land','vehicle','business','construction','general','other')
  funding_goal          DECIMAL(15,2) DEFAULT 0
  allocated_amount      DECIMAL(15,2) DEFAULT 0     -- updated by allocations
  status                ENUM('proposed','approved','active','completed','cancelled') DEFAULT 'proposed'
  ownership_rule        ENUM('equal_shares','contribution_ratio','custom','manual') DEFAULT 'equal_shares'
  start_date            DATE NULL
  expected_end_date     DATE NULL
  completed_at          TIMESTAMP NULL
  notes                 TEXT NULL
  created_at, updated_at, deleted_at
```

---

### 6.10 `fund_allocations`

```sql
fund_allocations
  id                    BIGINT UNSIGNED PK
  uuid                  CHAR(36) UNIQUE
  fund_group_id         BIGINT UNSIGNED FK→fund_groups
  fund_project_id       BIGINT UNSIGNED FK→fund_projects
  amount                DECIMAL(15,2)
  allocation_date       DATE
  requested_by          BIGINT UNSIGNED FK→users
  approved_by           BIGINT UNSIGNED FK→users NULL
  approval_status       ENUM('pending','approved','rejected') DEFAULT 'pending'
  notes                 TEXT NULL
  created_at, updated_at
```

---

### 6.11 `project_ownerships`

```sql
project_ownerships
  id                    BIGINT UNSIGNED PK
  fund_project_id       BIGINT UNSIGNED FK→fund_projects
  fund_group_member_id  BIGINT UNSIGNED FK→fund_group_members
  ownership_percentage  DECIMAL(8,4)               -- e.g. 33.3333
  share_count_snapshot  DECIMAL(8,2) NULL
  contribution_snapshot DECIMAL(15,2) NULL
  is_manual_override    BOOLEAN DEFAULT FALSE
  effective_at          TIMESTAMP
  calculated_by         VARCHAR(100)               -- which rule was applied
  created_at, updated_at
```

---

### 6.12 `approval_requests`

```sql
approval_requests
  id                    BIGINT UNSIGNED PK
  uuid                  CHAR(36) UNIQUE
  fund_group_id         BIGINT UNSIGNED FK→fund_groups
  approvable_type       VARCHAR(191)               -- polymorphic: FundAllocation, FundProject, etc.
  approvable_id         BIGINT UNSIGNED
  requested_by          BIGINT UNSIGNED FK→users
  status                ENUM('pending','approved','rejected','withdrawn') DEFAULT 'pending'
  required_approvers    JSON NULL                  -- list of user_ids who must approve
  notes                 TEXT NULL
  resolved_at           TIMESTAMP NULL
  created_at, updated_at
```

### 6.13 `approval_decisions`

```sql
approval_decisions
  id                    BIGINT UNSIGNED PK
  approval_request_id   BIGINT UNSIGNED FK→approval_requests
  decided_by            BIGINT UNSIGNED FK→users
  decision              ENUM('approved','rejected')
  comment               TEXT NULL
  decided_at            TIMESTAMP
  created_at
```

---

## 7. Business Rules

### 7.1 Membership Rules

- A contact or user can belong to only one active membership per fund group.
- Share count can be changed by owner/admin; a new `MemberShareHolding` record is created with `effective_from = next cycle start date`. The old record gets `effective_to` set.
- A member can be suspended (no dues generated) or exited (historical data preserved).
- A new member joining mid-year gets dues from their `joined_at` date's cycle onwards.
- An exited member's unpaid dues remain; exit does not auto-waive arrears.

### 7.2 Contribution Rules

- Dues are generated per billing cycle per active (non-suspended, non-exited) member.
- `contribution_dues.expected_amount` = active shares × `share_unit_amount` at cycle start.
- Partial payment is allowed; `paid_amount` accumulates per due record.
- Advance payment: a payment with no linked due. Stored in `contribution_payments` with `payment_type = advance`. Advance balance is applied to next cycle's due during due generation.
- Penalty is calculated at `billing_cycle.due_date + grace_days`. Applied as a new `contribution_payments` record with `payment_type = penalty`.
- Arrears carry forward: unpaid amount from cycle N is added to `arrear_carried` in cycle N+1.
- A due can be manually waived by a user with `can_manage_funds = true`. Requires a reason.

### 7.3 Ledger Rules

- Every `ContributionPayment` creates a **credit** `GroupLedgerEntry`.
- Every `FundAllocation` creates a **debit** `GroupLedgerEntry`.
- Every refund creates a **debit** `GroupLedgerEntry`.
- `running_balance` is maintained by the `GroupLedgerService` and must never go negative (service-level guard).
- Ledger entries are append-only (no updates, no deletes). Corrections via reversal entries only.

### 7.4 Allocation Rules

- Allocation amount must not exceed `group_ledger.current_balance` at time of approval.
- If `fund_group.requires_approval = true`, all allocations require an approved `ApprovalRequest` before the ledger debit is posted.
- `fund_project.allocated_amount` is updated atomically when the ledger debit is posted.
- One project can receive multiple allocations.

### 7.5 Ownership Rules

| Rule | Calculation |
|---|---|
| `equal_shares` | Each active member's total shares / group total shares |
| `contribution_ratio` | Member's total lifetime contributions / group total contributions |
| `custom` | Admin-defined percentage, must sum to 100% |
| `manual` | Entered directly per member |

Ownership is recalculated whenever: a new allocation is posted, a member's share count changes, or admin triggers recalculation manually.

### 7.6 Approval Rules

- Approval can be required for: fund allocation, project creation, new member addition, member exit, share change.
- `required_approvers` list is set at the time of request creation.
- All listed approvers must approve for status to become `approved` (AND logic, v1).
- Any one rejection makes the request `rejected`.

---

## 8. Service Layer Design

All services live in `app/Domains/FundGroup/Services/`.

### 8.1 `FundGroupService`

```php
// Responsibilities
createGroup(CreateFundGroupDTO $dto): FundGroup
updateGroup(FundGroup $group, UpdateFundGroupDTO $dto): FundGroup
pauseGroup(FundGroup $group): void
closeGroup(FundGroup $group): void
getGroupSummary(FundGroup $group): FundGroupSummaryDTO
```

### 8.2 `MembershipService`

```php
inviteMember(FundGroup $group, InviteMemberDTO $dto): FundGroupMember
acceptInvitation(FundGroupMember $member, User $user): void
updateMemberRole(FundGroupMember $member, string $role): void
suspendMember(FundGroupMember $member, string $reason): void
exitMember(FundGroupMember $member, ExitMemberDTO $dto): void
updateShares(FundGroupMember $member, UpdateSharesDTO $dto): MemberShareHolding
getActiveSharesAt(FundGroupMember $member, Carbon $date): float
```

### 8.3 `ContributionPlanService`

```php
createPlan(FundGroup $group, CreatePlanDTO $dto): ContributionPlan
updatePlan(ContributionPlan $plan, UpdatePlanDTO $dto): ContributionPlan
computeMemberExpectedAmount(FundGroupMember $member, BillingCycle $cycle): float
```

### 8.4 `BillingCycleService`

```php
generateCycleForGroup(FundGroup $group, int $year, int $month): BillingCycle
generateDuesForCycle(BillingCycle $cycle): Collection  // ContributionDue[]
applyAdvanceBalances(BillingCycle $cycle): void
applyArrears(BillingCycle $cycle): void
closeCycle(BillingCycle $cycle): void
getMemberDueSummary(FundGroupMember $member): Collection
```

### 8.5 `ContributionPaymentService`

```php
recordPayment(RecordPaymentDTO $dto): ContributionPayment
recordAdvancePayment(RecordAdvanceDTO $dto): ContributionPayment
applyPaymentToDue(ContributionPayment $payment, ContributionDue $due): void
waivedDue(ContributionDue $due, User $waivedBy, string $reason): void
calculatePenalty(ContributionDue $due): float
getMemberBalance(FundGroupMember $member): MemberBalanceDTO  // paid, due, advance, arrear
```

### 8.6 `GroupLedgerService`

```php
credit(FundGroup $group, CreditDTO $dto): GroupLedgerEntry
debit(FundGroup $group, DebitDTO $dto): GroupLedgerEntry    // throws if balance insufficient
getCurrentBalance(FundGroup $group): float
getLedgerStatement(FundGroup $group, DateRange $range): Collection
reverseEntry(GroupLedgerEntry $entry, User $by, string $reason): GroupLedgerEntry
```

### 8.7 `FundProjectService`

```php
proposeProject(FundGroup $group, ProposeProjectDTO $dto): FundProject
approveProject(FundProject $project, User $approver): void
rejectProject(FundProject $project, User $approver, string $reason): void
completeProject(FundProject $project): void
getProjectFundingStatus(FundProject $project): ProjectFundingStatusDTO
```

### 8.8 `FundAllocationService`

```php
requestAllocation(FundProject $project, RequestAllocationDTO $dto): FundAllocation
approveAllocation(FundAllocation $allocation, User $approver): void
rejectAllocation(FundAllocation $allocation, User $approver, string $reason): void
postAllocation(FundAllocation $allocation): void  // writes ledger debit
getGroupFundSummary(FundGroup $group): GroupFundSummaryDTO
```

### 8.9 `OwnershipService`

```php
calculateOwnership(FundProject $project): Collection  // ProjectOwnership[]
setManualOwnership(FundProject $project, array $ownerships): void  // must sum to 100
recalculateAll(FundProject $project): void
getMemberOwnershipAcrossProjects(FundGroupMember $member): Collection
```

### 8.10 `ApprovalService`

```php
createRequest(CreateApprovalDTO $dto): ApprovalRequest
recordDecision(ApprovalRequest $request, User $decider, string $decision, ?string $comment): void
resolveRequest(ApprovalRequest $request): void  // called after all decisions
isFullyApproved(ApprovalRequest $request): bool
```

---

## 9. API Endpoints

All routes registered in `routes/fund-group.php`, versioned under `/api/v1/`.

### Fund Groups

```
GET    /api/v1/fund-groups                        List groups for company
POST   /api/v1/fund-groups                        Create group
GET    /api/v1/fund-groups/{uuid}                 Get group detail + summary
PUT    /api/v1/fund-groups/{uuid}                 Update group
PATCH  /api/v1/fund-groups/{uuid}/pause           Pause group
PATCH  /api/v1/fund-groups/{uuid}/close           Close group
```

### Members

```
GET    /api/v1/fund-groups/{uuid}/members                  List members
POST   /api/v1/fund-groups/{uuid}/members                  Invite member
GET    /api/v1/fund-groups/{uuid}/members/{memberUuid}     Get member detail
PUT    /api/v1/fund-groups/{uuid}/members/{memberUuid}     Update role/flags
PATCH  /api/v1/fund-groups/{uuid}/members/{memberUuid}/shares    Update share count
PATCH  /api/v1/fund-groups/{uuid}/members/{memberUuid}/suspend   Suspend
PATCH  /api/v1/fund-groups/{uuid}/members/{memberUuid}/exit      Exit member
GET    /api/v1/fund-groups/{uuid}/members/{memberUuid}/balance    Member balance
```

### Contribution Plan

```
GET    /api/v1/fund-groups/{uuid}/contribution-plan         Get plan
POST   /api/v1/fund-groups/{uuid}/contribution-plan         Create plan
PUT    /api/v1/fund-groups/{uuid}/contribution-plan         Update plan
```

### Billing Cycles

```
GET    /api/v1/fund-groups/{uuid}/billing-cycles             List cycles
POST   /api/v1/fund-groups/{uuid}/billing-cycles             Generate cycle (manual trigger)
GET    /api/v1/fund-groups/{uuid}/billing-cycles/{id}        Cycle detail with dues
PATCH  /api/v1/fund-groups/{uuid}/billing-cycles/{id}/close  Close cycle
```

### Contribution Dues

```
GET    /api/v1/billing-cycles/{cycleId}/dues               List dues for cycle
GET    /api/v1/contribution-dues/{uuid}                    Due detail
PATCH  /api/v1/contribution-dues/{uuid}/waive              Waive due
```

### Payments

```
GET    /api/v1/fund-groups/{uuid}/payments                 List all payments
POST   /api/v1/fund-groups/{uuid}/payments                 Record payment
POST   /api/v1/fund-groups/{uuid}/payments/advance         Record advance payment
GET    /api/v1/contribution-payments/{uuid}                Payment detail
DELETE /api/v1/contribution-payments/{uuid}                Reverse payment (soft)
```

### Ledger

```
GET    /api/v1/fund-groups/{uuid}/ledger                   Ledger statement (filterable by date)
GET    /api/v1/fund-groups/{uuid}/ledger/balance           Current balance
```

### Projects

```
GET    /api/v1/fund-groups/{uuid}/projects                 List projects
POST   /api/v1/fund-groups/{uuid}/projects                 Propose project
GET    /api/v1/fund-groups/{uuid}/projects/{projUuid}      Project detail
PUT    /api/v1/fund-groups/{uuid}/projects/{projUuid}      Update project
PATCH  /api/v1/fund-groups/{uuid}/projects/{projUuid}/approve   Approve
PATCH  /api/v1/fund-groups/{uuid}/projects/{projUuid}/complete  Complete
```

### Fund Allocations

```
GET    /api/v1/fund-projects/{projUuid}/allocations        List allocations
POST   /api/v1/fund-projects/{projUuid}/allocations        Request allocation
GET    /api/v1/fund-allocations/{uuid}                     Allocation detail
PATCH  /api/v1/fund-allocations/{uuid}/approve             Approve allocation
PATCH  /api/v1/fund-allocations/{uuid}/reject              Reject allocation
```

### Ownership

```
GET    /api/v1/fund-projects/{projUuid}/ownership          Get ownership breakdown
POST   /api/v1/fund-projects/{projUuid}/ownership/recalculate  Trigger recalculation
PUT    /api/v1/fund-projects/{projUuid}/ownership          Set manual overrides
```

### Approval Requests

```
GET    /api/v1/fund-groups/{uuid}/approvals                List approval requests
GET    /api/v1/approval-requests/{uuid}                    Request detail
POST   /api/v1/approval-requests/{uuid}/decide             Submit decision
PATCH  /api/v1/approval-requests/{uuid}/withdraw           Withdraw request
```

---

## 10. Event Flow & Jobs

### Events

```php
// app/Domains/FundGroup/Events/

FundGroupCreated
MemberInvited
MemberJoined
MemberExited
SharesUpdated
BillingCycleGenerated
ContributionDueGenerated
PaymentRecorded
DueOverdue                  // fired by scheduled job
LedgerCredited
LedgerDebited
ProjectProposed
ProjectApproved
AllocationRequested
AllocationApproved
AllocationPosted
OwnershipRecalculated
ApprovalRequestCreated
ApprovalDecisionRecorded
```

### Listeners

```php
// On PaymentRecorded:
UpdateContributionDueStatus
PostLedgerCreditEntry
UpdateBillingCycleTotals
NotifyTreasurer

// On AllocationApproved:
PostAllocationToLedger
UpdateProjectAllocatedAmount
RecalculateProjectOwnership

// On DueOverdue:
ApplyPenalty
NotifyMemberOfOverdue
```

### Jobs

```php
// app/Domains/FundGroup/Jobs/

GenerateMonthlyBillingCycles   // runs on 1st of every month via scheduler
ApplyOverduePenalties          // runs daily, checks grace period
SendDueReminders               // runs weekly for unpaid dues
RecalculateGroupBalances       // data integrity check, runs nightly
```

### Scheduler (in `Console/Kernel.php`)

```php
$schedule->job(new GenerateMonthlyBillingCycles)->monthlyOn(1, '00:00');
$schedule->job(new ApplyOverduePenalties)->daily();
$schedule->job(new SendDueReminders)->weekly();
```

---

## 11. Permissions & Roles

### Fund Group Roles

| Role | Description |
|---|---|
| `owner` | Created the group; full control |
| `admin` | Can manage members, plans, and projects |
| `treasurer` | Can record payments, view ledger, manage dues |
| `auditor` | Read-only access to full ledger and reports |
| `member` | Can view their own dues and payment history |
| `viewer` | Read-only, no financial data |

### Capability Matrix

| Action | owner | admin | treasurer | auditor | member | viewer |
|---|---|---|---|---|---|---|
| Create/edit group | ✓ | ✓ | | | | |
| Invite members | ✓ | ✓ | | | | |
| Change member role | ✓ | ✓ | | | | |
| Update shares | ✓ | ✓ | | | | |
| Exit member | ✓ | ✓ | | | | |
| Create contribution plan | ✓ | ✓ | | | | |
| Generate billing cycle | ✓ | ✓ | ✓ | | | |
| Record payment | ✓ | ✓ | ✓ | | | |
| Waive due | ✓ | ✓ | | | | |
| View ledger | ✓ | ✓ | ✓ | ✓ | | |
| View own dues | ✓ | ✓ | ✓ | ✓ | ✓ | |
| Propose project | ✓ | ✓ | | | | |
| Approve project | ✓ | ✓ | | | | |
| Request allocation | ✓ | ✓ | ✓ | | | |
| Approve allocation | ✓ | ✓ | | | | |
| Set manual ownership | ✓ | ✓ | | | | |
| Approve requests | ✓ | ✓ | | | | |

### Implementation

Use existing `spatie/laravel-permission`. Create a `FundGroupPolicy` per action. Gate checks in controllers. Role stored in `fund_group_members.role`.

```php
// app/Domains/FundGroup/Policies/FundGroupPolicy.php
class FundGroupPolicy
{
    public function recordPayment(User $user, FundGroup $group): bool
    {
        return $group->getMemberRole($user)?->in(['owner','admin','treasurer']);
    }
    // ...
}
```

---

## 12. File & Directory Structure

```
app/
└── Domains/
    └── FundGroup/
        ├── Enums/
        │   ├── MemberRole.php
        │   ├── MemberStatus.php
        │   ├── ContributionType.php
        │   ├── BillingCycleStatus.php
        │   ├── DueStatus.php
        │   ├── PaymentType.php
        │   ├── LedgerEntryType.php
        │   ├── ProjectStatus.php
        │   ├── ProjectType.php
        │   ├── OwnershipRule.php
        │   └── ApprovalStatus.php
        │
        ├── Models/
        │   ├── FundGroup.php
        │   ├── FundGroupMember.php
        │   ├── ContributionPlan.php
        │   ├── MemberShareHolding.php
        │   ├── BillingCycle.php
        │   ├── ContributionDue.php
        │   ├── ContributionPayment.php
        │   ├── GroupLedgerEntry.php
        │   ├── FundProject.php
        │   ├── FundAllocation.php
        │   ├── ProjectOwnership.php
        │   ├── ApprovalRequest.php
        │   └── ApprovalDecision.php
        │
        ├── Services/
        │   ├── FundGroupService.php
        │   ├── MembershipService.php
        │   ├── ContributionPlanService.php
        │   ├── BillingCycleService.php
        │   ├── ContributionPaymentService.php
        │   ├── GroupLedgerService.php
        │   ├── FundProjectService.php
        │   ├── FundAllocationService.php
        │   ├── OwnershipService.php
        │   └── ApprovalService.php
        │
        ├── Repositories/
        │   ├── Contracts/
        │   │   ├── FundGroupRepositoryInterface.php
        │   │   ├── BillingCycleRepositoryInterface.php
        │   │   └── LedgerRepositoryInterface.php
        │   └── Eloquent/
        │       ├── FundGroupRepository.php
        │       ├── BillingCycleRepository.php
        │       └── LedgerRepository.php
        │
        ├── DTOs/
        │   ├── CreateFundGroupDTO.php
        │   ├── InviteMemberDTO.php
        │   ├── UpdateSharesDTO.php
        │   ├── CreatePlanDTO.php
        │   ├── RecordPaymentDTO.php
        │   ├── CreditDTO.php
        │   ├── DebitDTO.php
        │   ├── ProposeProjectDTO.php
        │   ├── RequestAllocationDTO.php
        │   ├── MemberBalanceDTO.php
        │   ├── GroupFundSummaryDTO.php
        │   └── ProjectFundingStatusDTO.php
        │
        ├── Events/
        │   ├── FundGroupCreated.php
        │   ├── MemberInvited.php
        │   ├── PaymentRecorded.php
        │   ├── LedgerDebited.php
        │   ├── AllocationPosted.php
        │   └── ... (all events listed in §10)
        │
        ├── Listeners/
        │   ├── UpdateContributionDueStatus.php
        │   ├── PostLedgerCreditEntry.php
        │   ├── PostAllocationToLedger.php
        │   ├── RecalculateProjectOwnership.php
        │   └── NotifyMemberOfOverdue.php
        │
        ├── Jobs/
        │   ├── GenerateMonthlyBillingCycles.php
        │   ├── ApplyOverduePenalties.php
        │   ├── SendDueReminders.php
        │   └── RecalculateGroupBalances.php
        │
        ├── Policies/
        │   ├── FundGroupPolicy.php
        │   ├── FundProjectPolicy.php
        │   └── FundAllocationPolicy.php
        │
        └── Http/
            ├── Controllers/
            │   ├── FundGroupController.php
            │   ├── FundGroupMemberController.php
            │   ├── ContributionPlanController.php
            │   ├── BillingCycleController.php
            │   ├── ContributionDueController.php
            │   ├── ContributionPaymentController.php
            │   ├── GroupLedgerController.php
            │   ├── FundProjectController.php
            │   ├── FundAllocationController.php
            │   ├── ProjectOwnershipController.php
            │   └── ApprovalRequestController.php
            │
            ├── Requests/
            │   ├── CreateFundGroupRequest.php
            │   ├── InviteMemberRequest.php
            │   ├── UpdateSharesRequest.php
            │   ├── RecordPaymentRequest.php
            │   ├── ProposeProjectRequest.php
            │   ├── RequestAllocationRequest.php
            │   └── ApprovalDecisionRequest.php
            │
            └── Resources/
                ├── FundGroupResource.php
                ├── FundGroupSummaryResource.php
                ├── FundGroupMemberResource.php
                ├── MemberBalanceResource.php
                ├── BillingCycleResource.php
                ├── ContributionDueResource.php
                ├── ContributionPaymentResource.php
                ├── GroupLedgerEntryResource.php
                ├── FundProjectResource.php
                ├── FundAllocationResource.php
                └── ProjectOwnershipResource.php

routes/
└── fund-group.php          ← registered in RouteServiceProvider under api middleware

database/migrations/
├── 2026_04_20_000001_create_fund_groups_table.php
├── 2026_04_20_000002_create_fund_group_members_table.php
├── 2026_04_20_000003_create_contribution_plans_table.php
├── 2026_04_20_000004_create_member_share_holdings_table.php
├── 2026_04_20_000005_create_billing_cycles_table.php
├── 2026_04_20_000006_create_contribution_dues_table.php
├── 2026_04_20_000007_create_contribution_payments_table.php
├── 2026_04_20_000008_create_group_ledger_entries_table.php
├── 2026_04_20_000009_create_fund_projects_table.php
├── 2026_04_20_000010_create_fund_allocations_table.php
├── 2026_04_20_000011_create_project_ownerships_table.php
├── 2026_04_20_000012_create_approval_requests_table.php
└── 2026_04_20_000013_create_approval_decisions_table.php
```

---

## 13. Migration Order

Migrations must be run in this exact order due to foreign key dependencies:

```
1.  fund_groups                 (depends on: companies, users)
2.  fund_group_members          (depends on: fund_groups, contacts, users)
3.  contribution_plans          (depends on: fund_groups)
4.  member_share_holdings       (depends on: fund_group_members)
5.  billing_cycles              (depends on: fund_groups)
6.  contribution_dues           (depends on: billing_cycles, fund_group_members)
7.  contribution_payments       (depends on: fund_groups, fund_group_members, contribution_dues, payments)
8.  group_ledger_entries        (depends on: fund_groups, users)
9.  fund_projects               (depends on: fund_groups, users)
10. fund_allocations            (depends on: fund_groups, fund_projects, users)
11. project_ownerships          (depends on: fund_projects, fund_group_members)
12. approval_requests           (depends on: fund_groups, users) [polymorphic]
13. approval_decisions          (depends on: approval_requests, users)
```

---

## 14. Key Workflows

### Workflow 1: Monthly Billing Cycle Generation

```
Scheduler triggers GenerateMonthlyBillingCycles (1st of month)
  → For each active FundGroup with auto_generate_cycles = true:
      → BillingCycleService::generateCycleForGroup(group, year, month)
          → Create BillingCycle record
          → For each active member:
              → Get active MemberShareHolding
              → Get advance balance from previous payments
              → Get arrears from previous unpaid due
              → Create ContributionDue record
              → expected_amount = shares × share_unit_amount
              → due_amount = expected_amount + arrear_carried + penalty - advance_used
          → Update BillingCycle.expected_total
          → Fire BillingCycleGenerated event
  → Listener: NotifyMembers of new dues
```

### Workflow 2: Recording a Payment

```
POST /api/v1/fund-groups/{uuid}/payments
  → ContributionPaymentService::recordPayment(dto)
      → Validate member is active in group
      → Create ContributionPayment record
      → Link to ContributionDue
      → Update ContributionDue.paid_amount, due_amount, status
      → Fire PaymentRecorded event

Listener: PostLedgerCreditEntry
  → GroupLedgerService::credit(group, CreditDTO)
      → Compute new running_balance
      → Create GroupLedgerEntry (credit)
      → Fire LedgerCredited event

Listener: UpdateBillingCycleTotals
  → Increment BillingCycle.collected_total
```

### Workflow 3: Fund Allocation to Project

```
POST /api/v1/fund-projects/{uuid}/allocations
  → FundAllocationService::requestAllocation(project, dto)
      → Validate requested amount ≤ current ledger balance
      → Create FundAllocation (approval_status = pending)
      → If group.requires_approval:
          → ApprovalService::createRequest(dto)
          → Fire AllocationRequested event
          → Notify approvers
      → Else:
          → Auto-approve and call postAllocation()

POST /api/v1/fund-allocations/{uuid}/approve
  → ApprovalService::recordDecision(request, user, 'approved')
      → Create ApprovalDecision
      → If all required approvers approved:
          → ApprovalService::resolveRequest → sets approval_status = approved
          → FundAllocationService::postAllocation(allocation)
              → GroupLedgerService::debit(group, DebitDTO)
                  → Guard: balance ≥ amount
                  → Create GroupLedgerEntry (debit)
              → Update FundAllocation.approval_status = approved
              → Update FundProject.allocated_amount += amount
              → Fire AllocationPosted event

Listener: RecalculateProjectOwnership
  → OwnershipService::recalculateAll(project)
      → Apply ownership_rule
      → Upsert ProjectOwnership records
```

### Workflow 4: Member Share Update

```
PATCH /api/v1/fund-groups/{uuid}/members/{memberUuid}/shares
  → MembershipService::updateShares(member, dto)
      → Set current MemberShareHolding.effective_to = last day of current month
      → Create new MemberShareHolding with effective_from = 1st of next month
      → Fire SharesUpdated event

Next billing cycle generation:
  → BillingCycleService picks up new share holding (effective_from matches cycle start)
  → New expected_amount reflects updated shares
```

---

## 15. Edge Cases & Error Handling

| Scenario | Handling |
|---|---|
| Payment recorded but ledger credit fails | Wrap in DB transaction; rollback payment if ledger write fails |
| Allocation requested when balance is 0 | Service-level guard, 422 response with `insufficient_balance` code |
| Two admins simultaneously approve allocation | Optimistic locking on `approval_requests.status`; second approve is a no-op if already resolved |
| Member exits with unpaid arrears | Exit is allowed; arrears remain on record; flag `exited_at` on member |
| Share count changed mid-cycle | New holding takes effect from next cycle; current cycle keeps the old share snapshot |
| Billing cycle generated twice for same month | Unique constraint on `(fund_group_id, cycle_year, cycle_month)`; job is idempotent |
| Ledger running_balance goes negative | Service throws `InsufficientFundException`; allocation is blocked |
| Ownership percentages don't sum to 100 | Validation in `OwnershipService`; 422 response |
| Duplicate member invitation | Unique constraint on `(fund_group_id, contact_id)`; 409 response |
| Group closed with open billing cycles | Close operation first closes all open cycles, marks unpaid dues as overdue |

---

## 16. Future Extension Points

These are not in v1 scope but the schema is designed to support them without breaking changes.

### Stripe-based automatic collection
`contribution_payments.payment_id` already links to the existing `payments` table. When automatic Stripe collection is added, the `ContributionPaymentService` can trigger a Stripe charge and link the result.

### Voting / Quorum governance
`approval_requests.required_approvers` is JSON and `approval_decisions` is a separate table. Switching from AND logic to majority-vote requires only a change in `ApprovalService::isFullyApproved()`.

### Profit / dividend distribution
`ProjectOwnership` percentages are already computed. A `Distribution` model linked to `FundProject` and `ProjectOwnership` can record payouts.

### Share transfer between members
`MemberShareHolding` supports history. A share transfer creates two holdings: one decreased, one increased, with the same `effective_from`.

### Multi-currency
`fund_groups.currency` is already per-group. Adding currency conversion to `GroupLedgerService` enables cross-currency reporting.

### Mobile money / bKash / Nagad integration
`contribution_payments.payment_method` is a free VARCHAR. A new `PaymentGateway` implementation (following the existing `BasePaymentGateway` contract) can be added without schema changes.

### Document attachments
Attach land documents, agreements, receipts to `FundProject` using the existing `spatie/laravel-medialibrary` integration already present in the platform.

---

## Appendix A: Glossary

| Term | Meaning |
|---|---|
| Fund Group | A named collaborative saving/investment unit |
| Member | A contact/user participating in a fund group with a defined role |
| Share | A unit of financial participation; member's monthly due = shares × unit rate |
| Billing Cycle | A monthly (or periodic) window during which dues are collected |
| Contribution Due | The amount a specific member owes for a specific billing cycle |
| Arrear | Unpaid amount from a prior cycle carried into the current cycle |
| Advance | Payment made before a due is generated; credited to next cycle |
| Group Ledger | The master fund account for a group, tracking all credits and debits |
| Fund Project | A named goal to which pooled group money is allocated |
| Allocation | Transfer from group ledger to a project |
| Ownership | Each member's percentage stake in a project, computed from contribution or shares |
| Approval Request | A formal governance record requiring one or more designated approvers |

---

## Appendix B: Quick Reference — Table Summary

| Table | Rows represent |
|---|---|
| `fund_groups` | One collaborative investment circle |
| `fund_group_members` | One person's membership in one group |
| `contribution_plans` | The financial rules for one group |
| `member_share_holdings` | A member's share count for a date range |
| `billing_cycles` | One collection period (e.g., April 2026) |
| `contribution_dues` | What one member owes in one cycle |
| `contribution_payments` | One actual payment received |
| `group_ledger_entries` | One financial movement in the group's fund |
| `fund_projects` | One named project/asset under a group |
| `fund_allocations` | One transfer from group fund to a project |
| `project_ownerships` | One member's % stake in one project |
| `approval_requests` | One governance approval request |
| `approval_decisions` | One approver's decision on one request |
