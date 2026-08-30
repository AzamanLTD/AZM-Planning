# AZM Architecture Visual Maps

**Status:** IN PROGRESS / REFERENCE  
**Last updated:** 2026-08-30 (UTC)

This document collects the highest-value visual maps for readers who want to understand AZM's architecture before reading the detailed plans. The individual repository plans contain diagrams at the points where the flow matters to the surrounding explanation.

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

`AZM-Planning` records the intended architecture and verified project history; it does not become a runtime dependency.

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
4. Treat diagrams as explanatory views, not independent sources of implementation truth; update them whenever a depicted architectural boundary materially changes.
