# Shopify ↔ SYNC ↔ Odoo — Scenario-Based Integration Guide

> **Version:** 1.0  
> **Date:** 2026-02-10  
> **Audience:** Developers, Integration Architects, Operations Team

---

## Table of Contents

1. [Role Distribution (Core Principle)](#role-distribution)
2. [Architecture Overview](#architecture-overview)
3. [Scenario 1 — Normal Warehouse Order](#scenario-1--normal-warehouse-order)
4. [Scenario 2 — Dropship Order](#scenario-2--dropship-order)
5. [Scenario 3 — Mixed Order (Warehouse + Dropship)](#scenario-3--mixed-order-warehouse--dropship)
6. [Scenario 4 — Out of Stock → Backorder / Fallback](#scenario-4--out-of-stock--backorder--fallback)
7. [Scenario 5 — Refund / Return](#scenario-5--refund--return)
8. [Scenario 6 — Odoo Stock Update → Shopify](#scenario-6--odoo-stock-update--shopify)
9. [Critical Field: location_id](#critical-field-location_id)
10. [Summary Matrix](#summary-matrix)

---

## Role Distribution

Each system in the pipeline has a single, well-defined responsibility:

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#e8e8e8', 'primaryTextColor': '#1a1a1a', 'lineColor': '#555', 'textColor': '#1a1a1a', 'fontSize': '14px'}}}%%
graph LR
    subgraph "📦 SHOPIFY"
        S["Order & Fulfillment\nDecision Center"]
    end
    subgraph "🔄 SYNC"
        Y["Data Carrier\n& Transformer"]
    end
    subgraph "⚙️ ODOO"
        O["Operations &\nAccounting Engine"]
    end

    S -->|"Payload"| Y
    Y -->|"Mapped Data"| O
    O -.->|"Stock Updates"| Y
    Y -.->|"Inventory Levels"| S

    style S fill:#96bf48,stroke:#5e8e3e,color:#1a1a1a
    style Y fill:#f5a623,stroke:#d4891a,color:#1a1a1a
    style O fill:#c9a5c0,stroke:#5a3a52,color:#1a1a1a
```

| System | Role | Owns |
|--------|------|------|
| **Shopify** | Order & Fulfillment Decision Center | Customer data, payment, location selection |
| **SYNC** | Data Carrier & Transformer | Payload mapping, field cleaning, format conversion |
| **Odoo** | Operations & Accounting Engine | Stock movements, picking, invoicing, payments |

---

## Architecture Overview

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#f0f0f0', 'primaryTextColor': '#1a1a1a', 'lineColor': '#555', 'textColor': '#1a1a1a', 'fontSize': '14px'}}}%%
flowchart TB
    Customer(("🛒 Customer")) --> Shopify

    subgraph Shopify ["Shopify"]
        SO["Sales Order"]
        LOC["Location Selection"]
        FF["Fulfillment"]
        REF["Refund"]
    end

    subgraph SYNC ["SYNC Layer"]
        MAP["Payload Mapper"]
        VAL["Validator"]
        TRANS["Transformer"]
    end

    subgraph Odoo ["Odoo ERP"]
        OO["Sale Order"]
        WH["Warehouse Picking"]
        DS["Dropship Flow"]
        INV["Invoice / Payment"]
        STK["Stock Management"]
    end

    SO --> MAP
    LOC --> MAP
    MAP --> VAL --> TRANS --> OO
    OO --> WH
    OO --> DS
    OO --> INV
    STK -.->|"Stock Sync"| TRANS
    TRANS -.->|"Inventory Levels"| Shopify

    style Shopify fill:#eafbe7,stroke:#96bf48,color:#1a1a1a
    style SYNC fill:#fff4e0,stroke:#f5a623,color:#1a1a1a
    style Odoo fill:#f3edf2,stroke:#714b67,color:#1a1a1a
```

---

## Scenario 1 — Normal Warehouse Order

> A standard order where all items ship from the company's own warehouse.

### Flow

```mermaid
sequenceDiagram
    actor Customer
    participant Shopify
    participant SYNC
    participant Odoo

    Customer->>Shopify: Places order
    Note over Shopify: Assigns Warehouse location

    Shopify->>SYNC: Order payload<br/>(location = Allierbygget)
    SYNC->>SYNC: Maps & transforms payload<br/>to Odoo format
    SYNC->>Odoo: Creates Sale Order

    activate Odoo
    Odoo->>Odoo: Confirms order
    Odoo->>Odoo: Warehouse stock reserved
    Odoo->>Odoo: Picking created
    Odoo->>Odoo: Invoice & payment processed
    deactivate Odoo

    Note over Odoo: ✅ Stock decremented from Allierbygget
```

### Roles

| System | Responsibility |
|--------|---------------|
| **Shopify** | Selects Allierbygget (Bergen) location + order source of truth |
| **SYNC** | Correct field mapping & payload delivery |
| **Odoo** | Stock reservation, picking, invoicing, payment |

---

## Location Architecture — How Shopify & Odoo Locations Connect

Before diving into Dropship scenarios, it's essential to understand how locations are structured across both systems and how SYNC maps between them.

### Shopify Side: Locations

Shopify has a flat list of locations. Each represents a fulfillment point:

| Shopify Location | Purpose |
|-----------------|---------|
| **Allierbygget (Bergen)** | **Main Warehouse** — physical stock (primary) |
| **Nettlager** | **Dropship** — aggregated vendor stock |

### Odoo Side: Vendor Locations (Detailed)

Odoo maintains **per-vendor** stock locations, each with two sub-locations:

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#f0f0f0', 'primaryTextColor': '#1a1a1a', 'lineColor': '#555', 'textColor': '#1a1a1a', 'fontSize': '13px'}}}%%
flowchart TB
    subgraph WH ["Main Warehouse"]
        WHS["WH/Stock"]
    end

    subgraph VendorLocations ["Vendor Locations (Type: Vendor)"]
        subgraph Ahlsell ["Ahlsell"]
            AS["View/Stock"]
            AQ["View/Quarantine"]
        end
        subgraph Dahl ["Brødrene Dahl"]
            DS["View/Stock"]
            DQ["View/Quarantine"]
        end
        subgraph Heidenreich ["Heidenreich"]
            HS["View/Stock"]
            HQ["View/Quarantine"]
        end
        subgraph Korsbakken ["Korsbakken"]
            KS["View/Stock"]
            KQ["View/Quarantine"]
        end
        subgraph Sanipro ["Sanipro"]
            SS["View/Stock"]
            SQ["View/Quarantine"]
        end
        subgraph VikingBad ["VikingBad"]
            VS["View/Stock"]
            VQ["View/Quarantine"]
        end
    end

    style WH fill:#eafbe7,stroke:#96bf48,color:#1a1a1a
    style VendorLocations fill:#fff4e0,stroke:#f5a623,color:#1a1a1a
    style Ahlsell fill:#f9f9f9,stroke:#ccc,color:#1a1a1a
    style Dahl fill:#f9f9f9,stroke:#ccc,color:#1a1a1a
    style Heidenreich fill:#f9f9f9,stroke:#ccc,color:#1a1a1a
    style Korsbakken fill:#f9f9f9,stroke:#ccc,color:#1a1a1a
    style Sanipro fill:#f9f9f9,stroke:#ccc,color:#1a1a1a
    style VikingBad fill:#f9f9f9,stroke:#ccc,color:#1a1a1a
```

| Vendor Location | Sub-Location | Purpose |
|----------------|-------------|---------|
| **Ahlsell** | `View/Stock` | Available stock from Ahlsell |
| | `View/Quarantine` | Quarantined / reserved stock |
| **Brødrene Dahl** | `View/Stock` | Available stock from Dahl |
| | `View/Quarantine` | Quarantined stock |
| **Heidenreich** | `View/Stock` / `View/Quarantine` | Same pattern |
| **Korsbakken** | `View/Stock` / `View/Quarantine` | Same pattern |
| **Sanipro** | `View/Stock` / `View/Quarantine` | Same pattern |
| **VikingBad** | `View/Stock` / `View/Quarantine` | Same pattern |

### Shopify ↔ Odoo Location Mapping (via `sync.shopify.location`)

The mapping model supports two types:

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#f0f0f0', 'primaryTextColor': '#1a1a1a', 'lineColor': '#555', 'textColor': '#1a1a1a', 'fontSize': '13px'}}}%%
flowchart LR
    subgraph Shopify ["Shopify Locations"]
        SA["Allierbygget (Bergen)"]
        SN["Nettlager"]
    end

    subgraph Mapping ["sync.shopify.location"]
        M1["Type: Warehouse\n1:1 Mapping"]
        M2["Type: Aggregated\nMulti-Source"]
    end

    subgraph Odoo ["Odoo Stock Locations"]
        OW["WH/Stock\n(Allierbygget)"]
        OA["Ahlsell View/Stock"]
        OD["Dahl View/Stock"]
        OH["Heidenreich View/Stock"]
        OK["Korsbakken View/Stock"]
        OS["Sanipro View/Stock"]
        OV["VikingBad View/Stock"]
    end

    SA --- M1 --- OW
    SN --- M2
    M2 --- OA
    M2 --- OD
    M2 --- OH
    M2 --- OK
    M2 --- OS
    M2 --- OV

    style Shopify fill:#eafbe7,stroke:#96bf48,color:#1a1a1a
    style Mapping fill:#fff4e0,stroke:#f5a623,color:#1a1a1a
    style Odoo fill:#f3edf2,stroke:#714b67,color:#1a1a1a
```

| Mapping Type | Shopify Location | Odoo Location(s) | Use Case |
|-------------|-----------------|-------------------|----------|
| **Warehouse** (1:1) | **Allierbygget (Bergen)** | WH/Stock | Main warehouse fulfillment |
| **Aggregated** (N:1) | **Nettlager** | Ahlsell, Dahl, Heidenreich, Korsbakken, Sanipro, VikingBad `View/Stock` | Combined vendor stock for Dropship |

> [!IMPORTANT]
> **Aggregated mapping** is key for Dropship: Shopify shows one "Nettlager" location, but Odoo tracks stock per-vendor separately. SYNC aggregates all vendor `View/Stock` quantities and pushes the combined total to Shopify's "Nettlager" location.

---

## Scenario 2 — Dropship Order

> The order is fulfilled directly by a vendor — no warehouse stock is touched. Odoo uses vendor-specific locations to route the Purchase Order.

### Flow

```mermaid
sequenceDiagram
    actor Customer
    participant Shopify
    participant SYNC
    participant Odoo
    participant Vendor as Vendor (e.g. Ahlsell)

    Customer->>Shopify: Places order
    Note over Shopify: Product available at<br/>"Nettlager" (Dropship)

    Shopify->>SYNC: Order payload<br/>(location = Nettlager)
    SYNC->>SYNC: Maps payload +<br/>includes location_id
    SYNC->>Odoo: Creates Sale Order<br/>(with Dropship location)

    activate Odoo
    Note over Odoo: Detects Dropship route<br/>on the product

    Odoo->>Odoo: NO warehouse stock deducted
    Odoo->>Odoo: Dropship route triggers
    Odoo->>Odoo: Finds vendor on product<br/>(e.g. Ahlsell)
    Odoo->>Vendor: Creates Purchase Order (PO)
    Note over Odoo: Stock source:<br/>Ahlsell View/Stock

    Vendor-->>Customer: Ships directly
    deactivate Odoo

    Note over Odoo: WH/Stock untouched
```

### Odoo Internal Routing (Dropship)

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#f0f0f0', 'primaryTextColor': '#1a1a1a', 'lineColor': '#555', 'textColor': '#1a1a1a', 'fontSize': '13px'}}}%%
flowchart TD
    SO["Sale Order\n(Dropship Route)"] --> ROUTE{"Product Route?"}
    
    ROUTE -->|"Dropship"| PO["Purchase Order\nCreated for Vendor"]
    ROUTE -->|"Warehouse"| PICK["Warehouse Picking\n(WH/Stock)"]
    
    PO --> VENDOR_LOC["Vendor Location\n(e.g. Ahlsell View/Stock)"]
    VENDOR_LOC --> SHIP["Ship Directly\nto Customer"]
    
    PICK --> WH_SHIP["Ship from\nWarehouse"]

    style SO fill:#f3edf2,stroke:#714b67,color:#1a1a1a
    style ROUTE fill:#fff4e0,stroke:#f5a623,color:#1a1a1a
    style PO fill:#e8f4fd,stroke:#4a90d9,color:#1a1a1a
    style PICK fill:#eafbe7,stroke:#96bf48,color:#1a1a1a
    style VENDOR_LOC fill:#fff4e0,stroke:#f5a623,color:#1a1a1a
    style SHIP fill:#e8f4fd,stroke:#4a90d9,color:#1a1a1a
    style WH_SHIP fill:#eafbe7,stroke:#96bf48,color:#1a1a1a
```

### Roles

| System | Responsibility |
|--------|---------------|
| **Shopify** | Routes to "Vendors" location based on availability |
| **SYNC** | Carries `location_id` to Odoo (critical!) |
| **Odoo** | Creates PO for specific vendor, uses vendor `View/Stock` location |

> [!CAUTION]
> If `location_id` is missing from the SYNC payload, Odoo defaults to Warehouse route — causing **incorrect stock deductions from WH/Stock** instead of triggering the vendor PO flow.

---

## Scenario 3 — Mixed Order (Warehouse + Dropship)

> A single order contains items from both the warehouse and one or more dropship vendors.

### Flow

```mermaid
sequenceDiagram
    actor Customer
    participant Shopify
    participant SYNC
    participant Odoo
    participant Ahlsell as Vendor: Ahlsell

    Customer->>Shopify: Places order (mixed items)
    Note over Shopify: Split fulfillment:<br/>Item A → Allierbygget (WH)<br/>Item B → Nettlager (Dropship)

    Shopify->>SYNC: Order payload with<br/>per-line location_id

    SYNC->>SYNC: Parses split information<br/>per line item
    SYNC->>Odoo: Creates Sale Order<br/>(with split location data)

    activate Odoo
    Note right of Odoo: --- WAREHOUSE FLOW (Item A) ---
    Odoo->>Odoo: Reserve stock from WH/Stock
    Odoo->>Odoo: Create picking (Allierbygget)

    Note right of Odoo: --- DROPSHIP FLOW (Item B) ---
    Odoo->>Odoo: No WH/Stock touched
    Odoo->>Odoo: Find vendor for product (Ahlsell)
    Odoo->>Ahlsell: Create Purchase Order
    Note right of Odoo: Source: Ahlsell View/Stock
    Ahlsell-->>Customer: Ships directly

    Note right of Odoo: --- COMMON ---
    Odoo->>Odoo: Invoice and payment for full order
    deactivate Odoo
```

### Split Routing Detail

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#f0f0f0', 'primaryTextColor': '#1a1a1a', 'lineColor': '#555', 'textColor': '#1a1a1a', 'fontSize': '13px'}}}%%
flowchart LR
    subgraph Order ["Sale Order (Mixed)"]
        L1["Line 1: Faucet\nRoute: Warehouse"]
        L2["Line 2: Bathtub\nRoute: Dropship"]
    end

    subgraph Processing ["Odoo Processing"]
        P1["Picking\nWH/Stock → Customer"]
        P2["PO → Ahlsell\nAhlsell View/Stock → Customer"]
    end

    L1 --> P1
    L2 --> P2

    style Order fill:#f3edf2,stroke:#714b67,color:#1a1a1a
    style Processing fill:#fff4e0,stroke:#f5a623,color:#1a1a1a
```

### Roles

| System | Responsibility |
|--------|---------------|
| **Shopify** | Splits fulfillment: Nettlager vs Vendors per line |
| **SYNC** | Carries per-line `location_id` values faithfully |
| **Odoo** | Routes each line: WH/Stock picking vs Vendor PO |

> [!IMPORTANT]
> The split is decided **entirely by Shopify**. SYNC must faithfully transport per-line `location_id` values. Odoo then routes each line to either `WH/Stock` (picking) or the appropriate vendor `View/Stock` (PO).

---

## Scenario 4 — Out of Stock → Backorder / Fallback

> Warehouse stock is 0, but vendor stock exists. Shopify routes to the "Nettlager" location automatically.

### Flow

```mermaid
sequenceDiagram
    actor Customer
    participant Shopify
    participant SYNC
    participant Odoo

    Customer->>Shopify: Places order

    Note over Shopify: Allierbygget (WH) stock = 0<br/>Nettlager stock available<br/>(aggregated from Ahlsell,<br/>Dahl, Korsbakken, etc.)

    Shopify->>Shopify: Availability check
    Note over Shopify: Decision: Route to Nettlager

    Shopify->>SYNC: Order payload<br/>(location_id = Nettlager)
    SYNC->>Odoo: Creates Sale Order<br/>(Dropship route)

    activate Odoo
    Odoo->>Odoo: WH/Stock NOT touched
    Odoo->>Odoo: Detects Dropship route
    Odoo->>Odoo: Finds vendor for product
    Odoo->>Odoo: Creates PO for vendor<br/>(e.g. Korsbakken)
    Note over Odoo: Source: Korsbakken View/Stock
    deactivate Odoo

    Note over Odoo: Correct routing — no phantom stock loss
```

### How Stock Availability Flows Back

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#f0f0f0', 'primaryTextColor': '#1a1a1a', 'lineColor': '#555', 'textColor': '#1a1a1a', 'fontSize': '13px'}}}%%
flowchart RL
    subgraph Odoo ["Odoo (Stock Truth)"]
        WH["WH/Stock = 0"]
        V1["Ahlsell View/Stock = 15"]
        V2["Dahl View/Stock = 8"]
        V3["Korsbakken View/Stock = 22"]
    end

    subgraph SYNC_AGG ["SYNC Aggregation"]
        AGG["Sum vendor stocks\n15 + 8 + 22 = 45"]
    end

    subgraph Shopify ["Shopify Display"]
        SA["Allierbygget: 0"]
        SN["Nettlager: 45"]
    end

    WH -->|"0"| SA
    V1 --> AGG
    V2 --> AGG
    V3 --> AGG
    AGG -->|"45"| SN

    style Odoo fill:#f3edf2,stroke:#714b67,color:#1a1a1a
    style SYNC_AGG fill:#fff4e0,stroke:#f5a623,color:#1a1a1a
    style Shopify fill:#eafbe7,stroke:#96bf48,color:#1a1a1a
```

### Roles

| System | Responsibility |
|--------|---------------|
| **Shopify** | Sees aggregated vendor stock, routes to "Vendors" when WH is empty |
| **SYNC** | Aggregates per-vendor `View/Stock` → single "Vendors" quantity for Shopify |
| **Odoo** | Maintains per-vendor stock truth, executes PO for correct vendor |

> [!NOTE]
> Shopify never knows *which* vendor has stock — it only sees the aggregated "Vendors" total. Odoo determines the specific vendor (Ahlsell, Dahl, etc.) based on the product's supplier configuration when creating the Purchase Order.

---

## Scenario 5 — Refund / Return

> A refund is initiated from Shopify and needs to be reflected in Odoo's accounting.

### Flow

```mermaid
sequenceDiagram
    actor Customer
    participant Shopify
    participant SYNC
    participant Odoo

    Customer->>Shopify: Requests refund
    Shopify->>Shopify: Processes refund

    Shopify->>SYNC: Refund payload<br/>(refunds[] array)
    SYNC->>SYNC: Maps refund data<br/>to Odoo format
    SYNC->>Odoo: Sends refund data

    activate Odoo
    Odoo->>Odoo: Creates Credit Note
    
    opt Physical return
        Odoo->>Odoo: Creates Return Picking
        Odoo->>Odoo: Stock re-added to inventory
    end

    Odoo->>Odoo: Reconciles payment
    deactivate Odoo

    Note over Odoo: ✅ Credit note + optional stock return
```

### Roles

| System | Responsibility |
|--------|---------------|
| **Shopify** | Initiates and processes the refund event |
| **SYNC** | Carries `refunds[]` payload to Odoo |
| **Odoo** | Creates credit note, optional return picking, payment reconciliation |

---

## Scenario 6 — Odoo Stock Update → Shopify

> Stock changes in Odoo (from vendors, warehouse adjustments) need to be reflected in Shopify.

### Flow

```mermaid
sequenceDiagram
    participant Vendor
    participant Odoo
    participant SYNC
    participant Shopify

    Vendor->>Odoo: Stock file received<br/>(price/qty update)
    Odoo->>Odoo: Updates internal stock<br/>(per location)

    Odoo->>SYNC: Stock change event<br/>(location-based quantities)
    SYNC->>SYNC: Maps to Shopify<br/>inventory format
    SYNC->>Shopify: Inventory level update<br/>(per location_id)

    Shopify->>Shopify: Updates storefront<br/>stock display

    Note over Shopify: ✅ Customer sees accurate stock
```

### Data Flow Direction

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#f0f0f0', 'primaryTextColor': '#1a1a1a', 'lineColor': '#555', 'textColor': '#1a1a1a', 'fontSize': '14px'}}}%%
flowchart LR
    Odoo["⚙️ Odoo\n(Stock Truth)"] -->|"Location-based\nstock levels"| SYNC["🔄 SYNC"]
    SYNC -->|"Inventory API"| Shopify["📦 Shopify\n(Customer View)"]

    style Odoo fill:#c9a5c0,stroke:#5a3a52,color:#1a1a1a
    style SYNC fill:#f5a623,stroke:#d4891a,color:#1a1a1a
    style Shopify fill:#96bf48,stroke:#5e8e3e,color:#1a1a1a
```

### Roles

| System | Responsibility |
|--------|---------------|
| **Odoo** | Source of truth for stock levels |
| **SYNC** | Transfers location-based stock data |
| **Shopify** | Displays stock to customer |

> [!NOTE]
> This is the **reverse flow** — Odoo → SYNC → Shopify. Odoo is the stock truth source; Shopify is the display layer.

---

## Critical Field: `location_id`

The `location_id` is the **single most important field** in the entire integration. It determines which operational path Odoo executes.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#f0f0f0', 'primaryTextColor': '#1a1a1a', 'lineColor': '#555', 'textColor': '#1a1a1a', 'fontSize': '14px'}}}%%
flowchart TD
    LID{"location_id\npresent?"}
    
    LID -->|"Yes: Warehouse"| WH["Warehouse Flow\n• Stock reserved\n• Picking created\n• Standard shipment"]
    LID -->|"Yes: Dropship"| DS["Dropship Flow\n• No stock deducted\n• PO to vendor\n• Direct shipment"]
    LID -->|"❌ Missing"| ERR["⚠️ DEFAULT: Warehouse\n• WRONG stock deducted\n• Inventory mismatch\n• Manual correction needed"]

    style WH fill:#eafbe7,stroke:#96bf48,color:#1a1a1a
    style DS fill:#fff4e0,stroke:#f5a623,color:#1a1a1a
    style ERR fill:#fde8e8,stroke:#e53e3e,color:#1a1a1a
```

> [!CAUTION]
> **Missing `location_id` = Silent Data Corruption.**  
> Odoo will not raise an error — it will silently default to Warehouse, causing stock discrepancies that are difficult to trace after the fact.

---

## Summary Matrix

| Scenario | Shopify Decides | SYNC Carries | Odoo Executes |
|----------|----------------|-------------|---------------|
| **1. Warehouse** | Warehouse location | Mapped payload | Stock ↓ + Picking + Invoice |
| **2. Dropship** | Dropship location | `location_id` | Vendor PO (no stock ↓) |
| **3. Mixed** | Split fulfillment | Per-line locations | Parallel WH + DS flows |
| **4. Backorder** | Availability routing | Location decision | Correct flow per location |
| **5. Refund** | Refund event | `refunds[]` payload | Credit Note + Return |
| **6. Stock Sync** | Displays stock | Stock levels | Stock truth source |

---

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#f0f0f0', 'primaryTextColor': '#1a1a1a', 'lineColor': '#555', 'textColor': '#1a1a1a', 'fontSize': '14px'}}}%%
graph TB
    subgraph Legend
        direction LR
        A["🟢 Shopify: Decides"] ~~~ B["🟡 SYNC: Carries"] ~~~ C["🟣 Odoo: Executes"]
    end

    style A fill:#96bf48,stroke:#5e8e3e,color:#1a1a1a
    style B fill:#f5a623,stroke:#d4891a,color:#1a1a1a
    style C fill:#c9a5c0,stroke:#5a3a52,color:#1a1a1a
```

> **Golden Rule:** Shopify decides, SYNC transports, Odoo executes.  
> `location_id` is the bridge that ensures the right decision reaches the right operation.
