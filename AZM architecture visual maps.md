# AZM Architecture Visual Maps

**Status:** IN PROGRESS / LIVING REFERENCE  
**Last updated:** 2026-08-30 (UTC)

This document collects the highest-value visual maps for readers who want to understand AZM's architecture before reading the detailed plans. The individual repository plans contain diagrams at the points where the flow matters to the surrounding explanation.

Diagrams are explanatory views, not independent implementation truth. When implementation changes a depicted boundary or lifecycle, update the relevant map and the repository plan in the same planning batch.

## Master platform

```mermaid
flowchart LR
    A[Identity & Trust] --> B[Unified Money / Ledger]
    B --> C[Domain Verticals]
    C --> D[SDUI / Presentation]
    D --> E[Realtime / Eventing]
    E --> F[Social Graph]
    F --> G[Notifications]
    G --> H[Observability / Reconciliation]
    H -. controls & feedback .-> A
```

## Cross-repository relationship

```mermaid
flowchart TD
    A[AZM-Planning] --> B[AZM-frontend]
    A --> C[AZM-backend]
    A --> D[AZM-businessPortal]
    A --> E[AZM-adminPortal]
    B <--> C
    D <--> C
    E <--> C
    D --> F[Storefront Contract]
    F --> B
    C --> G[Domain Events]
    G --> B
    G --> D
    G --> E
```

`AZM-Planning` records intended architecture and verified project history; it does not become a runtime dependency.

## Retail end-to-end

```mermaid
flowchart LR
    A[Portal Configuration] --> B[Storefront Contract]
    B --> C[Customer Renderer]
    C --> D[Product / Variant]
    D --> E[Cart]
    E --> F[Checkout Intent]
    F --> G[Backend Validation]
    G --> H[Inventory Reservation]
    H --> I[Payment / Escrow]
    I --> J[Order]
    J --> K[Fulfillment]
    J --> L[Order History]
    J --> M[Safe Domain Activity]
    M --> N[Social / Notifications]
```

## Financial truth boundary

```mermaid
flowchart TD
    A[Customer / Merchant UI] --> B[Intent]
    B --> C[Authenticated API]
    C --> D[Domain Validation]
    D --> E[Atomic Database State]
    E --> F[Ledger / Payment Authority]
    F --> G[Auditable Domain Event]
    G --> H[Realtime / Push / Feed]
    H -. display only .-> A
```

The UI never becomes the authority for money, balances, inventory, payment settlement, or protected state.

## Financial transaction lifecycle

```mermaid
flowchart TD
    A[Customer Action] --> B[Transaction Quote]
    B --> C[Payment Intent]
    C --> D[Escrow / Payment State]
    D --> E{Authoritative Outcome}
    E -->|Funded| F[Order / Booking Becomes Financially Eligible]
    E -->|Released| G[Settlement]
    E -->|Refunded| H[Refund + Inventory Release Where Applicable]
    E -->|Disputed| I[Dispute Resolution]
    I --> G
    I --> H
    F --> J[Domain Event]
    G --> J
    H --> J
    J --> K[Audit / Reconciliation]
    J --> L[Realtime / Notifications]
```

## Escrow funding concurrency

```mermaid
flowchart TD
    A[Funding Request A] --> C{DB state = DRAFT?}
    B[Funding Request B] --> C
    C -->|single atomic winner| D[DRAFT → FUNDED]
    C -->|losing concurrent transition| E[Reject / Roll Back Entire Transaction]
    D --> F[Debit + Fee + Escrow Lock + History]
    F --> G[Domain Event]
    E --> H[No Partial Financial Mutation]
```

The database transition is the final concurrency boundary; service checks remain responsible for authorization and financial validation.

## Order lifecycle

```mermaid
stateDiagram-v2
    [*] --> CREATED
    CREATED --> AWAITING_PAYMENT
    AWAITING_PAYMENT --> PAID
    PAID --> DELIVERED
    DELIVERED --> COMPLETED
    CREATED --> CANCELLED
    AWAITING_PAYMENT --> CANCELLED
    PAID --> REFUNDED
    PAID --> DISPUTED
    DELIVERED --> DISPUTED
    DISPUTED --> REFUNDED
    DISPUTED --> COMPLETED: authoritative resolution permits
```

## Inventory lifecycle

```mermaid
stateDiagram-v2
    [*] --> Available
    Available --> Reserved: accepted order
    Reserved --> Consumed: fulfillment completes
    Reserved --> Available: cancellation/refund
    Available --> Available: untracked item
```

## Reconciliation boundary

```mermaid
flowchart LR
    A[Order State] --> C{Reconciliation}
    B[Payment / Escrow State] --> C
    D[Ledger / Transaction History] --> C
    C -->|consistent| E[Verified]
    C -->|mismatch| F[Discrepancy Record]
    F --> G[Review / Safe Repair]
    G --> H[Auditable Resolution]
```

Reconciliation observes authoritative sources; it must not silently rewrite financial history.

## Notifications and realtime

```mermaid
flowchart LR
    A[Authoritative State Change] --> B[Domain Event]
    B --> C[Notification Policy]
    C --> D[Push]
    C --> E[Realtime]
    C --> F[In-app History]
    D -. delivery retry .-> B
    E -. duplicate / late safe .-> B
```

## Social-to-commerce loop

```mermaid
flowchart LR
    A[People] --> B[Businesses / Places]
    B --> C[Products / Services]
    C --> D[Safe Activity]
    D --> E[Discovery]
    E --> F[Conversation / Engagement]
    F --> G[Transaction]
    G --> D
```

The loop is intentionally constrained: sensitive payment instruments, balances, private transaction metadata and other protected financial details do not become public social activity.

## Agent reading order

1. Start with `README.md` for the master architecture.
2. Use the relevant repository plan for implementation state and detailed flows.
3. Use this file when a visual architecture map provides faster orientation.
4. Treat diagrams as explanatory views, not independent sources of implementation truth.
5. When implementation materially changes a depicted flow, update this map and the affected repository plan in the same coherent batch.
6. Record newly discovered risks and unresolved questions rather than silently deleting them.
