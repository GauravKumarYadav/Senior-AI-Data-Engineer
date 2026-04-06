# Module 7: Data Vault 2.0

---

## Screen 1: Why Data Vault?

### The Integration Problem

If you've studied Kimball dimensional modeling, you know its power: star schemas are fast to query, intuitive for business users, and excellent for department-level reporting. But when enterprises try to build a **single integrated data warehouse** serving dozens of source systems, Kimball's approach starts to crack.

**The pain points of Kimball for enterprise integration:**

- **Rigid coupling to business processes.** Each fact table maps to a specific business process (orders, shipments, claims). When the business restructures — merges departments, acquires companies, changes workflows — your entire warehouse schema needs surgery. Adding a new source system that doesn't fit neatly into an existing fact table means redesigning conformed dimensions and rebuilding ETL.

- **Tightly coupled ETL.** In Kimball, transformations happen *before* loading. Data is cleaned, conformed, and shaped into stars during ETL. If a business rule changes retroactively ("actually, we now want to classify that product category differently for the last 3 years"), you're reprocessing and reloading massive volumes.

- **Auditability gaps.** Once data enters a Kimball star schema, it's been transformed. The raw source record is gone. When compliance asks "show me exactly what System X sent us on March 15th, 2024," you can't — you only have the conformed, transformed result.

- **Parallel development bottlenecks.** Multiple teams can't easily build ETL independently when everything must conform to shared dimensions. Team A's changes to `dim_customer` can break Team B's fact table loads.

### Enter Data Vault

**Data Vault** is a modeling methodology invented by **Dan Linstedt** in the **1990s** (formalized in 2000, with Data Vault 2.0 published in 2013). It was designed to solve exactly these integration problems for large-scale enterprise data warehouses.

The core philosophy: **separate structure from content, and never lose anything.**

Data Vault's advantages:

- **Agility** — New sources are added without restructuring existing tables. No refactoring of shared dimensions.
- **Auditability** — Every record retains its load timestamp and source system. You can always trace data back to its origin.
- **Parallel loading** — Hubs, Links, and Satellites load independently. Multiple teams and ETL streams operate without conflicts.
- **Schema evolution** — Adding new attributes means adding new Satellites, not altering existing tables.
- **Historical completeness** — Full history is captured by default, without the complexity of SCD Type 2 flag management.

### The Three Building Blocks

Data Vault has only three core entity types:

```
┌──────────────────────────────────────────────────────────┐
│                    DATA VAULT MODEL                       │
│                                                          │
│   ┌──────────┐     ┌──────────┐     ┌──────────────┐    │
│   │   HUB    │────▶│   LINK   │◀────│     HUB      │    │
│   │(Business │     │(Relation-│     │  (Business   │    │
│   │   Key)   │     │   ship)  │     │    Key)      │    │
│   └────┬─────┘     └────┬─────┘     └──────┬───────┘    │
│        │                │                   │            │
│   ┌────▼─────┐     ┌────▼─────┐     ┌──────▼───────┐    │
│   │SATELLITE │     │SATELLITE │     │  SATELLITE   │    │
│   │(Context/ │     │(Context/ │     │  (Context/   │    │
│   │ History) │     │ History) │     │   History)   │    │
│   └──────────┘     └──────────┘     └──────────────┘    │
│                                                          │
│   Hubs = WHAT exists (business keys)                     │
│   Links = HOW things relate                              │
│   Satellites = WHY / WHEN / descriptive context          │
└──────────────────────────────────────────────────────────┘
```

> **Analogy:** Think of Data Vault like a passport system. **Hubs** are the passport registry — they record that a person *exists* and assign them a unique identifier. **Links** are the immigration stamps — they record relationships (this person visited this country on this date). **Satellites** are the pages of detailed information — physical description, address history, visa details. The registry never changes once a person is registered, relationships accumulate over time, and details are versioned as they change.

### 💡 Interview Insight

> When asked *"What is Data Vault and why would you use it?"*, say: **"Data Vault is a modeling methodology designed for enterprise data warehouse integration. It separates structure (Hubs and Links) from descriptive context (Satellites), making it agile — you can add new sources without refactoring existing tables. Every record carries its load timestamp and source system for full auditability. The pattern enables parallel loading because Hubs, Links, and Satellites are independent load units. I'd use Data Vault when integrating many source systems where requirements change frequently, and when auditability and traceability are critical — like in banking, insurance, or healthcare."**

---

## Screen 2: Hubs — The Anchor of Business Identity

### What Is a Hub?

A **Hub** represents a core **business concept** identified by its **business key**. Think of it as the "registry of existence" — a Hub says "this customer exists," "this product exists," "this account exists." Nothing more.

Hubs are intentionally *minimal*. They contain:

| Column | Purpose | Example |
|--------|---------|---------|
| `hash_key` (PK) | Deterministic hash of the business key | `MD5('CUST-10042')` |
| `business_key` | The natural key from the source | `'CUST-10042'` |
| `load_date` | When this key was *first* seen | `2024-01-15 09:30:00` |
| `record_source` | Which system provided this key first | `'CRM_SYSTEM'` |

**Hubs are immutable.** Once a business key is registered, that row never changes. No updates, no deletes. The load_date and record_source reflect the *first time* the key appeared in the warehouse. If the same customer appears later from a different source system, the Hub row doesn't change — the existing entry stands.

### Why Business Keys, Not Surrogate Keys?

In Kimball, surrogate keys (`dim_customer_sk = 42`) are king — they insulate the warehouse from source system key changes. In Data Vault, the **business key** is the anchor because:

1. **Integration point.** When five source systems all have a concept of "customer," the business key (e.g., email address, SSN, tax ID) is what lets you recognize they're talking about the *same* entity. Surrogates are meaningless across systems.
2. **Deterministic.** Anyone can compute the hash key from the business key independently. No centralized sequence generator needed. This enables parallel, distributed loading.
3. **Auditability.** Business users understand business keys. Nobody can audit a surrogate key.

Data Vault *does* use a hash-based surrogate (the `hash_key`) as the primary key for join performance — but it's **derived deterministically from the business key**, not generated by a sequence.

### Hub Example: E-Commerce Customer

```sql
CREATE TABLE hub_customer (
    customer_hk        BINARY(16)      NOT NULL,  -- MD5 hash of customer_bk
    customer_bk        VARCHAR(50)     NOT NULL,   -- Business key (e.g., email)
    load_date          TIMESTAMP       NOT NULL,   -- First seen in warehouse
    record_source      VARCHAR(100)    NOT NULL,   -- Source system identifier
    
    CONSTRAINT pk_hub_customer PRIMARY KEY (customer_hk)
);

-- Loading a Hub: INSERT only if the business key doesn't already exist
INSERT INTO hub_customer (customer_hk, customer_bk, load_date, record_source)
SELECT
    MD5(src.customer_email)         AS customer_hk,
    src.customer_email              AS customer_bk,
    CURRENT_TIMESTAMP               AS load_date,
    'ECOMMERCE_APP'                 AS record_source
FROM staging_customers src
WHERE NOT EXISTS (
    SELECT 1 FROM hub_customer h
    WHERE h.customer_hk = MD5(src.customer_email)
);
```

### Hub Example: Banking Account

```sql
CREATE TABLE hub_account (
    account_hk         BINARY(16)      NOT NULL,
    account_bk         VARCHAR(20)     NOT NULL,   -- Account number
    load_date          TIMESTAMP       NOT NULL,
    record_source      VARCHAR(100)    NOT NULL,

    CONSTRAINT pk_hub_account PRIMARY KEY (account_hk)
);
```

### Composite Business Keys

Sometimes a business concept requires multiple columns to form its identity. For example, a "store location" might be identified by `(chain_code, store_number)`. In this case, you concatenate the components before hashing:

```sql
-- Composite business key: chain_code + store_number
INSERT INTO hub_store (store_hk, chain_code, store_number, load_date, record_source)
SELECT
    MD5(CONCAT(src.chain_code, '||', src.store_number))  AS store_hk,
    src.chain_code,
    src.store_number,
    CURRENT_TIMESTAMP,
    'POS_SYSTEM'
FROM staging_stores src
WHERE NOT EXISTS (
    SELECT 1 FROM hub_store h
    WHERE h.store_hk = MD5(CONCAT(src.chain_code, '||', src.store_number))
);
```

Note the `'||'` delimiter — this prevents collisions between keys like `('AB', 'C')` and `('A', 'BC')`, which would produce the same hash without a separator.

> **Analogy:** A Hub is like a birth certificate registry. It records that a person was born (exists), their legal name (business key), the date the birth was recorded, and which hospital filed the record. You don't put height, weight, or address on the birth registry — that's descriptive data, and it changes. The registry is permanent.

### 💡 Interview Insight

> **"Why are Hubs immutable?"**
>
> Say: **"Hubs represent the existence of a business entity, identified by its business key. Once we know a customer exists, that fact doesn't change — only the descriptive details about that customer change, and those go into Satellites. Making Hubs immutable guarantees referential stability — every Link and Satellite that references a Hub hash key can trust it will never be updated or deleted. This also simplifies loading: Hub loads are pure INSERT-if-not-exists operations with no UPDATE logic."**

---

## Screen 3: Links — Modeling Relationships

### What Is a Link?

A **Link** captures a **relationship** (or association, or transaction) between two or more Hubs. In Data Vault, *all relationships are modeled as many-to-many by default.* This is a fundamental departure from Kimball, where relationships are embedded in fact tables or enforced by dimension foreign keys.

Links are also immutable — once a relationship is recorded, it stays. If a customer *was once* associated with a product through an order, that association is permanently captured.

A Link contains:

| Column | Purpose | Example |
|--------|---------|---------|
| `link_hk` (PK) | Hash of the combined business keys | `MD5('CUST-42' + 'PROD-99')` |
| `hub_1_hk` (FK) | Hash key reference to first Hub | `MD5('CUST-42')` |
| `hub_2_hk` (FK) | Hash key reference to second Hub | `MD5('PROD-99')` |
| `load_date` | When this relationship was first seen | `2024-03-10 14:22:00` |
| `record_source` | Which system reported the relationship | `'ORDER_SYSTEM'` |

### Why Many-to-Many by Default?

In reality, most business relationships *are* many-to-many, or *become* many-to-many over time. A customer can buy many products. A product can be bought by many customers. An employee can belong to many departments (transfers). A patient can see many doctors.

By modeling everything as many-to-many from the start, Data Vault avoids the painful refactoring that happens in Kimball when a "one-to-many" relationship unexpectedly becomes many-to-many.

### Link Example: Orders (Customer ↔ Product)

```sql
CREATE TABLE link_order (
    order_hk           BINARY(16)      NOT NULL,  -- MD5(customer_bk || order_id || product_bk)
    customer_hk        BINARY(16)      NOT NULL,  -- FK to hub_customer
    product_hk         BINARY(16)      NOT NULL,  -- FK to hub_product
    order_hk_bk        VARCHAR(30)     NOT NULL,  -- Order business key (for traceability)
    load_date          TIMESTAMP       NOT NULL,
    record_source      VARCHAR(100)    NOT NULL,
    
    CONSTRAINT pk_link_order PRIMARY KEY (order_hk),
    CONSTRAINT fk_link_order_customer FOREIGN KEY (customer_hk) REFERENCES hub_customer(customer_hk),
    CONSTRAINT fk_link_order_product FOREIGN KEY (product_hk) REFERENCES hub_product(product_hk)
);

-- Loading a Link
INSERT INTO link_order (order_hk, customer_hk, product_hk, order_hk_bk, load_date, record_source)
SELECT
    MD5(CONCAT(src.customer_email, '||', src.order_id, '||', src.product_sku))  AS order_hk,
    MD5(src.customer_email)      AS customer_hk,
    MD5(src.product_sku)         AS product_hk,
    src.order_id                 AS order_hk_bk,
    CURRENT_TIMESTAMP            AS load_date,
    'ORDER_SYSTEM'               AS record_source
FROM staging_orders src
WHERE NOT EXISTS (
    SELECT 1 FROM link_order l
    WHERE l.order_hk = MD5(CONCAT(src.customer_email, '||', src.order_id, '||', src.product_sku))
);
```

### Multi-Hub Links

Links aren't limited to two Hubs. A three-way relationship is perfectly valid:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ hub_customer │     │  hub_product │     │  hub_store   │
│              │     │              │     │              │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       └────────────┬───────┘────────────────────┘
                    │
             ┌──────▼───────┐
             │  link_sale   │  ← Customer bought Product at Store
             │              │
             └──────────────┘
```

### Handling Relationship Changes

What happens when a customer cancels an order? Or an employee transfers departments? The Link row **stays** — it records that the relationship *once existed.* The change is tracked in a **Satellite on the Link** (called a Link Satellite):

```sql
-- Satellite on the Link to track relationship status changes
CREATE TABLE sat_order_details (
    order_hk           BINARY(16)      NOT NULL,  -- FK to link_order
    load_date          TIMESTAMP       NOT NULL,  -- When this version was loaded
    hash_diff          BINARY(16)      NOT NULL,  -- Hash of descriptive columns for CDC
    record_source      VARCHAR(100)    NOT NULL,
    order_status       VARCHAR(20),               -- 'PLACED', 'SHIPPED', 'CANCELLED'
    order_amount       DECIMAL(12,2),
    shipping_method    VARCHAR(50),
    
    CONSTRAINT pk_sat_order_details PRIMARY KEY (order_hk, load_date)
);
```

When an order is cancelled, a new row is inserted into `sat_order_details` with `order_status = 'CANCELLED'`. The Link itself remains untouched — the *relationship* between customer, product, and order existed, and we don't erase history.

> **Analogy:** Links are like marriage certificates. A marriage certificate records that two people entered into a relationship. If they divorce, you don't destroy the certificate — you file a new document (Satellite record) recording the status change. The historical record of the relationship's existence is preserved.

### 💡 Interview Insight

> **"How does Data Vault handle relationship changes?"**
>
> Say: **"Links are immutable — they record that a relationship existed. Changes to that relationship (status updates, cancellations, amendments) are captured in a Satellite attached to the Link. For example, a link_order captures that Customer X ordered Product Y. If the order is cancelled, a new row appears in the Link's Satellite with the updated status. This preserves the full history: we know the order was placed, when it was cancelled, and by which source system. The Link row itself never changes."**

---

## Screen 4: Satellites — The History Keepers

### What Is a Satellite?

If Hubs answer "WHAT exists?" and Links answer "HOW things relate?", then **Satellites** answer "WHAT do we know about it, and WHEN did we know it?"

Satellites hold all **descriptive, contextual, and historized** attributes. They are the *only* entity type in Data Vault that captures change over time. Every time an attribute value changes in the source, a new row is inserted into the Satellite with the new values, a new `load_date`, and a `hash_diff` for change detection.

### The One-Satellite-Per-Source Rule

A critical Data Vault 2.0 rule: **one Satellite per source system per Hub or Link.** If customer data comes from both the CRM and the billing system, you create two Satellites:

```
                    ┌──────────────┐
                    │ hub_customer │
                    │              │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
     ┌────────▼─────┐     │    ┌───────▼────────┐
     │sat_customer  │     │    │ sat_customer   │
     │  _crm        │     │    │   _billing     │
     │(from CRM)    │     │    │ (from Billing) │
     └──────────────┘     │    └────────────────┘
                          │
                 ┌────────▼─────────┐
                 │ sat_customer     │
                 │   _web_profile   │
                 │ (from Website)   │
                 └──────────────────┘
```

**Why separate Satellites per source?**

1. **Different rates of change.** The CRM might update customer names monthly, while the billing system updates payment info daily. Mixing them in one table means storing redundant copies of unchanged CRM fields every time a billing field changes.
2. **Auditability.** You can always ask "What did the billing system tell us about this customer on March 15?" without interference from other sources.
3. **Parallel loading.** Each source loads its own Satellite independently — no contention.

### Satellite Structure

```sql
CREATE TABLE sat_customer_crm (
    customer_hk        BINARY(16)      NOT NULL,  -- FK to hub_customer
    load_date          TIMESTAMP       NOT NULL,  -- When this version was loaded
    load_end_date      TIMESTAMP       NULL,      -- Optional: when next version arrived
    hash_diff          BINARY(16)      NOT NULL,  -- MD5 hash of all descriptive columns
    record_source      VARCHAR(100)    NOT NULL,  -- Source system
    -- Descriptive attributes:
    first_name         VARCHAR(100),
    last_name          VARCHAR(100),
    email              VARCHAR(200),
    phone              VARCHAR(20),
    loyalty_tier       VARCHAR(20),
    
    CONSTRAINT pk_sat_customer_crm PRIMARY KEY (customer_hk, load_date)
);
```

**Key columns explained:**

- **`customer_hk` + `load_date`** = composite primary key. This pair uniquely identifies a specific version of a customer's CRM data.
- **`hash_diff`** = MD5 or SHA-256 hash of *all descriptive columns* concatenated together. This is used for **change detection**: if the incoming record's `hash_diff` matches the current Satellite row's `hash_diff`, nothing has changed — skip the insert. If it's different, a new version is inserted.
- **`load_end_date`** (optional) = populated when the *next* version arrives, creating a closed time range for easy "point-in-time" queries. Some implementations omit this and rely on windowed queries instead.

### Loading a Satellite with Change Detection

```sql
-- Step 1: Compare incoming data against current Satellite state
INSERT INTO sat_customer_crm
    (customer_hk, load_date, hash_diff, record_source,
     first_name, last_name, email, phone, loyalty_tier)
SELECT
    MD5(stg.customer_email)                                           AS customer_hk,
    CURRENT_TIMESTAMP                                                 AS load_date,
    MD5(CONCAT_WS('||',
        stg.first_name, stg.last_name, stg.email,
        stg.phone, stg.loyalty_tier
    ))                                                                AS hash_diff,
    'CRM_SYSTEM'                                                     AS record_source,
    stg.first_name,
    stg.last_name,
    stg.email,
    stg.phone,
    stg.loyalty_tier
FROM staging_crm_customers stg
LEFT JOIN (
    -- Get the current (latest) version for each customer
    SELECT customer_hk, hash_diff
    FROM sat_customer_crm
    WHERE load_end_date IS NULL   -- Current record
) curr ON curr.customer_hk = MD5(stg.customer_email)
WHERE curr.customer_hk IS NULL                  -- New customer (no prior Satellite row)
   OR curr.hash_diff != MD5(CONCAT_WS('||',     -- Existing customer, data changed
        stg.first_name, stg.last_name, stg.email,
        stg.phone, stg.loyalty_tier
    ));
```

### How Satellites Replace SCD Complexity

In Kimball, implementing SCD Type 2 requires managing surrogate keys, current flags (`is_current`), effective/expiration dates, and careful UPDATE + INSERT logic. In Data Vault, **history is the natural default:**

| Kimball SCD Type 2 | Data Vault Satellite |
|--------------------|---------------------|
| Must decide SCD type per column upfront | All changes captured automatically |
| Surrogate key changes with each version | Hub hash key is stable across versions |
| `is_current` flag requires UPDATE | New row inserted; optional `load_end_date` |
| Complex ETL with UPDATE + INSERT | INSERT-only (append-only pattern) |
| Mixed concerns: structure + history in one table | Structure (Hub) and history (Sat) separated |

> **Analogy:** Satellites are like medical records. Every time you visit a doctor, a new entry is created with your current vitals (weight, blood pressure, medications). Nobody erases your previous records — they stack up chronologically. To see your current state, you look at the most recent entry. To see your history, you read all entries in order. The `hash_diff` is like the nurse checking "has anything changed since last visit?" — if nothing changed, no new entry is created.

### 💡 Interview Insight

> **"How do Satellites handle historical data differently from SCDs?"**
>
> Say: **"In Kimball, you must decide the SCD type per attribute upfront — Type 1 overwrites, Type 2 creates versioned rows with surrogate keys and current flags, Type 3 adds columns. In Data Vault, Satellites capture all history by default as an INSERT-only append pattern. Every change creates a new row with a load timestamp and hash_diff for change detection. There's no need for current flags or surrogate key management. The Hub's hash key is stable across all versions, making joins simpler. And because each source has its own Satellite, you get natural separation of change rates and full source-level auditability."**

---

## Screen 5: Hash Keys — The Engine of Parallel Loading

### Why Hashing Instead of Sequences?

In traditional data warehouses, surrogate keys are generated by **auto-incrementing sequences** (`IDENTITY` columns, `SERIAL`, `SEQUENCE`). This creates a serial bottleneck:

```
Source A ──┐                ┌── INSERT (SK=1001)
Source B ──┼── SEQUENCE ──▶ ├── INSERT (SK=1002)  ← Sequential, single-threaded!
Source C ──┘    (mutex)     └── INSERT (SK=1003)
```

Every insert must coordinate with the sequence generator. In a distributed or parallel loading scenario, this becomes the bottleneck. Two loaders can't independently generate keys — they'd produce duplicates or gaps.

Data Vault 2.0 replaces sequences with **deterministic hash functions**:

```
Source A ──── MD5('CUST-42') ──── INSERT (HK=0x7a3b...)  ← Independent!
Source B ──── MD5('CUST-42') ──── (same hash, skip!)     ← No coordination!
Source C ──── MD5('PROD-99') ──── INSERT (HK=0x4f1c...)  ← Parallel!
```

### Benefits of Hash Keys

1. **Parallel loading.** Any number of ETL processes can compute hash keys independently — no shared state, no locks, no sequence coordination. If two loaders encounter the same business key, they produce the same hash, and the INSERT-if-not-exists logic handles the dedup.

2. **Deterministic joins.** To join a Hub to a Link or Satellite, you don't need to look up the surrogate key — you compute it. `MD5('CUST-42')` always equals `MD5('CUST-42')` regardless of when or where it was computed.

3. **Distribution-friendly.** In distributed databases (BigQuery, Snowflake, Redshift), hash keys distribute evenly across nodes. Sequential integer keys cause hot-spotting where recent inserts all hit the same partition.

4. **Idempotent loads.** Re-running a load with the same source data produces the same hash keys, making loads safely repeatable.

### Computing Hash Keys in SQL

**Simple business key:**

```sql
-- Single-column business key
SELECT MD5(customer_email) AS customer_hk
FROM staging_customers;

-- Using SHA-256 for lower collision risk (32 bytes vs 16 bytes)
SELECT SHA2(customer_email, 256) AS customer_hk
FROM staging_customers;
```

**Composite business key (multiple columns):**

```sql
-- Always use a consistent delimiter to avoid collision ambiguity
SELECT MD5(CONCAT_WS('||', 
    UPPER(TRIM(chain_code)), 
    UPPER(TRIM(store_number))
)) AS store_hk
FROM staging_stores;
```

**Link hash keys (combination of parent Hub business keys):**

```sql
-- Link hash key is a hash of all the participating business keys
SELECT MD5(CONCAT_WS('||',
    UPPER(TRIM(customer_email)),
    UPPER(TRIM(order_id)),
    UPPER(TRIM(product_sku))
)) AS order_hk
FROM staging_orders;
```

**Hash diff for Satellites (change detection):**

```sql
-- hash_diff covers ALL descriptive columns
SELECT MD5(CONCAT_WS('||',
    COALESCE(first_name, ''),
    COALESCE(last_name, ''),
    COALESCE(email, ''),
    COALESCE(phone, ''),
    COALESCE(CAST(loyalty_tier AS VARCHAR), '')
)) AS hash_diff
FROM staging_crm_customers;
```

### Critical Implementation Rules

| Rule | Why |
|------|-----|
| **Always UPPER + TRIM** before hashing | `' John '` and `'john'` should produce the same hash |
| **Use a consistent delimiter** like `'||'` | Prevents `('AB','C')` and `('A','BC')` from colliding |
| **COALESCE NULLs to empty string** | `MD5(NULL)` = `NULL`, which breaks everything |
| **Choose one hash function and stick with it** | MD5 everywhere, or SHA-256 everywhere — never mix |
| **Document your hashing convention** | Everyone on the team must hash the same way |

### MD5 vs SHA-256: The Collision Question

```
MD5:    128-bit → 16 bytes → ~3.4 × 10^38 possible values
SHA-256: 256-bit → 32 bytes → ~1.2 × 10^77 possible values
```

**Collision risk in practice:**

With MD5, the birthday paradox tells us you'd expect a collision at roughly **2^64 (~18.4 quintillion)** records. For reference, the largest enterprise data warehouses hold billions to low trillions of records — still orders of magnitude below the collision threshold.

**Practical recommendation:**

| Factor | MD5 | SHA-256 |
|--------|-----|---------|
| Speed | Faster | ~30-40% slower |
| Storage | 16 bytes | 32 bytes |
| Collision risk | Negligible for DW scales | Essentially zero |
| Cryptographic security | Broken (don't use for security!) | Secure |
| Industry adoption for DV | Very common | Growing |

For Data Vault purposes (where hash keys are used for deduplication and joins, not security), **MD5 is perfectly safe** and more storage-efficient. Use SHA-256 if regulatory or organizational policy requires it, or if the warehouse will scale to extreme volumes (trillions+ of records).

> **Analogy:** Hash keys are like fingerprints. Every person (business key) has a unique fingerprint (hash), and you can take that fingerprint anywhere in the world without coordinating with a central registry. Two officers in different cities can independently fingerprint the same person and get identical results. That's the power of deterministic hashing — no coordination needed.

### 💡 Interview Insight

> **"Why does Data Vault use hash keys instead of auto-incrementing surrogate keys?"**
>
> Say: **"Hash keys enable parallel, distributed loading without coordination. Any ETL process can independently compute the same hash from the same business key — there's no centralized sequence to serialize on. This means multiple source systems can load simultaneously without conflicts. Hash keys are also deterministic, making loads idempotent — re-running a load produces the same keys. And they distribute evenly across database nodes, avoiding the hot-spotting problem that sequential integers cause in distributed architectures like BigQuery or Snowflake."**

---

## Screen 6: PIT Tables & Bridge Tables — Performance Optimization

### The Query Performance Problem

Data Vault is optimized for **loading** — parallel, auditable, agile. But it's not optimized for **querying.** To reconstruct a complete picture of a customer at a specific point in time, you need to join the Hub to multiple Satellites, each with its own timeline:

```sql
-- "Show me the complete customer profile as of March 15, 2024"
-- This requires temporal joins across multiple Satellites:
SELECT
    h.customer_bk,
    crm.first_name,
    crm.last_name,
    crm.email,
    bill.payment_method,
    bill.credit_score,
    web.last_login,
    web.preferred_language
FROM hub_customer h
LEFT JOIN sat_customer_crm crm
    ON h.customer_hk = crm.customer_hk
    AND crm.load_date = (
        SELECT MAX(load_date) FROM sat_customer_crm
        WHERE customer_hk = h.customer_hk
        AND load_date <= '2024-03-15'
    )
LEFT JOIN sat_customer_billing bill
    ON h.customer_hk = bill.customer_hk
    AND bill.load_date = (
        SELECT MAX(load_date) FROM sat_customer_billing
        WHERE customer_hk = h.customer_hk
        AND load_date <= '2024-03-15'
    )
LEFT JOIN sat_customer_web web
    ON h.customer_hk = web.customer_hk
    AND web.load_date = (
        SELECT MAX(load_date) FROM sat_customer_web
        WHERE customer_hk = h.customer_hk
        AND web.load_date <= '2024-03-15'
    );
```

This query is **correct** but **horrifically expensive.** Each correlated subquery scans the Satellite table. With three Satellites, you have three correlated subqueries. With ten Satellites? Unmanageable.

### Point-in-Time (PIT) Tables

A **PIT table** is a pre-computed lookup structure that, for each Hub key and each snapshot date, stores the `load_date` of the applicable Satellite record. It eliminates the correlated subqueries:

```sql
CREATE TABLE pit_customer (
    customer_hk        BINARY(16)      NOT NULL,
    snapshot_date      DATE            NOT NULL,
    -- One column per Satellite, pointing to the effective load_date:
    sat_crm_load_date      TIMESTAMP,
    sat_billing_load_date  TIMESTAMP,
    sat_web_load_date      TIMESTAMP,
    
    CONSTRAINT pk_pit_customer PRIMARY KEY (customer_hk, snapshot_date)
);
```

**PIT tables are populated by a batch process** (typically nightly or hourly):

```sql
-- Populate PIT for each snapshot date
INSERT INTO pit_customer
SELECT
    h.customer_hk,
    snap.snapshot_date,
    (SELECT MAX(load_date) FROM sat_customer_crm
     WHERE customer_hk = h.customer_hk AND load_date <= snap.snapshot_date),
    (SELECT MAX(load_date) FROM sat_customer_billing
     WHERE customer_hk = h.customer_hk AND load_date <= snap.snapshot_date),
    (SELECT MAX(load_date) FROM sat_customer_web
     WHERE customer_hk = h.customer_hk AND load_date <= snap.snapshot_date)
FROM hub_customer h
CROSS JOIN (
    SELECT DISTINCT snapshot_date FROM dim_date
    WHERE snapshot_date BETWEEN '2024-01-01' AND CURRENT_DATE
) snap;
```

Now the previously horrible query becomes a simple set of equi-joins:

```sql
-- Fast point-in-time query using the PIT table
SELECT
    h.customer_bk,
    crm.first_name, crm.last_name, crm.email,
    bill.payment_method, bill.credit_score,
    web.last_login, web.preferred_language
FROM pit_customer pit
JOIN hub_customer h           ON pit.customer_hk = h.customer_hk
LEFT JOIN sat_customer_crm crm
    ON pit.customer_hk = crm.customer_hk
    AND pit.sat_crm_load_date = crm.load_date
LEFT JOIN sat_customer_billing bill
    ON pit.customer_hk = bill.customer_hk
    AND pit.sat_billing_load_date = bill.load_date
LEFT JOIN sat_customer_web web
    ON pit.customer_hk = web.customer_hk
    AND pit.sat_web_load_date = web.load_date
WHERE pit.snapshot_date = '2024-03-15';
```

**No correlated subqueries.** Pure equi-joins. Orders of magnitude faster.

### Bridge Tables

**Bridge tables** resolve many-to-many relationships from Links into structures that the presentation layer (star schemas, BI tools) can consume efficiently. They pre-walk the Link relationships and flatten them:

```sql
CREATE TABLE bridge_customer_product (
    customer_hk        BINARY(16)      NOT NULL,
    product_hk         BINARY(16)      NOT NULL,
    order_hk           BINARY(16)      NOT NULL,
    snapshot_date      DATE            NOT NULL,
    order_count        INT,                        -- Aggregated metric
    total_spend        DECIMAL(12,2),              -- Aggregated metric
    
    CONSTRAINT pk_bridge_cust_prod PRIMARY KEY (customer_hk, product_hk, snapshot_date)
);
```

Bridge tables sit in the **Business Vault** layer (between Raw Vault and the presentation star schemas) and are rebuilt periodically. They're particularly useful for:

- Resolving complex multi-hop Link traversals (Customer → Order → Product → Category)
- Pre-computing aggregates for dashboard performance
- Feeding star schema fact tables in the Information Mart

```
┌────────────────────────────────────────────────────────┐
│                  QUERY FLOW                             │
│                                                        │
│   Raw Vault              Business Vault    Info Mart   │
│   ┌──────────┐          ┌──────────────┐  ┌────────┐  │
│   │ Hubs     │───PIT───▶│ PIT Tables   │─▶│ Star   │  │
│   │ Links    │──Bridge─▶│ Bridge Tables│─▶│ Schema │  │
│   │Satellites│          │ Calc Sats    │  │        │  │
│   └──────────┘          └──────────────┘  └────────┘  │
└────────────────────────────────────────────────────────┘
```

> **Analogy:** PIT tables are like a library's daily catalog snapshot. Instead of searching through every book's checkout history to find who had it on March 15, you check the snapshot from March 15 which already lists the state of every book. Bridge tables are like a "recommended books" cross-reference card that pre-computes which readers also liked which other books — so you don't have to traverse the full checkout graph in real-time.

### 💡 Interview Insight

> **"How do you make Data Vault performant for querying?"**
>
> Say: **"Raw Data Vault is optimized for loading, not querying. For query performance, we build PIT (Point-in-Time) tables that pre-compute which Satellite record is effective for each Hub key at each snapshot date — eliminating expensive correlated subqueries and turning temporal joins into simple equi-joins. For complex Link traversals, we use Bridge tables that pre-walk many-to-many relationships. Both PIT and Bridge tables sit in the Business Vault layer and are refreshed periodically. For end-user consumption, we typically build a star schema Information Mart on top — giving analysts the familiar Kimball experience backed by Data Vault's integration and auditability."**

---

## Screen 7: Data Vault vs Kimball — The Decision Framework

### When to Use Each

This isn't a religious war — Data Vault and Kimball solve *different problems*, and the best architectures often use **both**.

| Criterion | Kimball Star Schema | Data Vault 2.0 |
|-----------|-------------------|----------------|
| **Primary purpose** | Business reporting & analytics | Enterprise data integration |
| **Optimization target** | Query performance | Load agility & auditability |
| **Number of sources** | Few (1-5 sources per mart) | Many (10-100+ source systems) |
| **Rate of change** | Stable requirements | Frequently changing requirements |
| **Schema changes** | Painful (ALTER + backfill) | Add new Satellites, no refactoring |
| **Auditability** | Limited (data is transformed) | Full (source + timestamp on every record) |
| **History tracking** | SCD Types (complex) | Satellite append (simple) |
| **ETL complexity** | Transform-heavy (ELT/ETL) | Load-heavy (ELT, minimal transform) |
| **Query complexity** | Simple (star joins) | Complex (temporal joins, needs PIT/Bridge) |
| **End-user accessibility** | Excellent (BI-native) | Poor (not designed for direct querying) |
| **Parallel development** | Bottlenecked on shared dims | Independent team loading |
| **Skill requirements** | Widely understood | Specialized knowledge needed |
| **Data volume sweet spot** | Small-to-large | Large-to-massive |
| **Regulatory compliance** | Adequate | Excellent |

### When to Use Both: The Layered Architecture

The most common production architecture uses Data Vault for **integration** and Kimball for **presentation**:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ENTERPRISE DATA WAREHOUSE                         │
│                                                                     │
│  ┌───────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │  STAGING      │    │   RAW VAULT      │    │  BUSINESS VAULT  │  │
│  │  LAYER        │───▶│                  │───▶│                  │  │
│  │               │    │  Hubs            │    │  PIT Tables      │  │
│  │  Source data  │    │  Links           │    │  Bridge Tables   │  │
│  │  as-is,       │    │  Satellites      │    │  Calc Satellites │  │
│  │  temp storage │    │                  │    │  Business rules  │  │
│  │               │    │  No business     │    │  applied HERE    │  │
│  │               │    │  rules applied   │    │                  │  │
│  └───────────────┘    └──────────────────┘    └────────┬─────────┘  │
│                                                        │            │
│                                                        ▼            │
│                                               ┌──────────────────┐  │
│                                               │ INFORMATION MART │  │
│                                               │ (Star Schemas)   │  │
│                                               │                  │  │
│                                               │  fact_sales      │  │
│                                               │  dim_customer    │  │
│                                               │  dim_product     │  │
│                                               │  dim_date        │  │
│                                               │                  │  │
│                                               │  Kimball-style   │  │
│                                               │  for BI tools    │  │
│                                               └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

**How the layers work together:**

1. **Staging Layer** — Raw data from source systems lands here, unmodified. Temporary — cleared after loading to the Raw Vault.

2. **Raw Vault** — Data is loaded into Hubs, Links, and Satellites with *zero business rules applied.* This is the "system of record" — it stores exactly what the sources sent, with full lineage (load timestamps, record sources). Data is never deleted from the Raw Vault.

3. **Business Vault** — This is where business rules, calculations, and derived data live. **Calculated Satellites** (e.g., `sat_customer_lifetime_value`) apply business logic to Raw Vault data. PIT and Bridge tables are built here for performance. If a business rule changes, you re-derive — you never touch the Raw Vault.

4. **Information Mart** — Kimball star schemas built from the Business Vault. This is what BI tools (Tableau, Power BI, Looker) and analysts consume. Familiar fact-and-dimension structure. Multiple marts can serve different business domains, all sourced from the same integrated vault.

### Decision Flowchart

```
                        ┌───────────────────────────┐
                        │ How many source systems?   │
                        └─────────────┬─────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │ 1-5 sources     │                  │ 10+ sources
                    ▼                 │                  ▼
            ┌───────────────┐        │       ┌──────────────────┐
            │ Requirements  │        │       │ Data Vault +     │
            │ stable?       │        │       │ Star Schema Marts│
            └───────┬───────┘        │       └──────────────────┘
                    │                │
          ┌─────────┼─────────┐      │
          │ Yes     │         │ No   │
          ▼         │         ▼      │
   ┌─────────────┐  │  ┌──────────────────┐
   │ Kimball     │  │  │ Data Vault +     │
   │ Star Schema │  │  │ Star Schema Marts│
   └─────────────┘  │  └──────────────────┘
                    │
                    │ 5-10 sources
                    ▼
            ┌───────────────┐
            │ Need audit    │
            │ trail?        │
            └───────┬───────┘
                    │
          ┌─────────┼─────────┐
          │ Yes              │ No
          ▼                  ▼
   ┌──────────────┐   ┌─────────────┐
   │ Data Vault + │   │ Kimball     │
   │ Star Marts   │   │ Star Schema │
   └──────────────┘   └─────────────┘
```

### Real-World Example: E-Commerce Company

Consider an e-commerce company with these source systems: website, mobile app, CRM, payment processor, warehouse management, shipping carrier, customer support, marketing automation, product catalog, and a returns system. That's **10 source systems** — all with their own version of "customer," "order," and "product."

**Without Data Vault (Kimball only):**
- One massive `dim_customer` conformed dimension that must accommodate all 10 sources
- Every time a new source is added, the dimension is altered and all ETL is retested
- Historical changes require SCD Type 2 management across multiple source versions
- If marketing changes their customer segmentation rules, you reprocess years of data

**With Data Vault + Kimball Marts:**
- `hub_customer` integrates customer identity across all 10 sources
- Each source has its own Satellite: `sat_customer_crm`, `sat_customer_web`, `sat_customer_billing`, etc.
- Adding a new source? Create a new Satellite. Zero impact on existing structures.
- The Information Mart builds `dim_customer` from the Business Vault, applying current business rules
- If rules change, re-derive the mart — the Raw Vault is untouched and auditable

> **Analogy:** Data Vault is the **vault** (pun intended) — the secure, fireproof, auditable repository where every document is stored in its original form with a chain-of-custody log. Kimball star schemas are the **display cases** — beautifully organized exhibits designed for visitors (business users) to browse easily. You wouldn't let visitors rummage through the vault, and you wouldn't store your originals in a display case. You need both.

### 💡 Interview Insight

> **"When would you choose Data Vault over Kimball, or vice versa?"**
>
> Say: **"It's not either-or — in most enterprise architectures, you use both. Data Vault excels at the integration layer: when you have many source systems, changing requirements, and regulatory auditability needs. Kimball excels at the presentation layer: star schemas are intuitive for business users and BI tools. The pattern is Raw Vault for ingestion and integration, Business Vault for applying business rules with PIT and Bridge tables, and Kimball-style Information Marts for consumption. I'd go Kimball-only for a small, stable, single-purpose analytics project. I'd add Data Vault when integrating more than 5-10 sources or when compliance requires full data lineage."**

---

## Screen 8: Quiz — Test Your Data Vault Knowledge

### Question 1

**What is the primary purpose of a Hub in Data Vault?**

- A) To store all descriptive attributes about a business entity
- B) To register the existence of a business entity using its business key, with load metadata
- C) To capture relationships between different business entities
- D) To apply business rules and transformations to source data

**✅ Correct Answer: B**

*Explanation: Hubs are deliberately minimal — they contain only the business key, a hash key (derived from the business key), the load_date (when the key was first seen), and the record_source. All descriptive attributes go into Satellites. Relationships go into Links. Business rules go into the Business Vault layer.*

---

### Question 2

**Why does Data Vault 2.0 use hash keys instead of auto-incrementing surrogate keys?**

- A) Hash keys are more secure and prevent unauthorized access
- B) Hash keys are smaller and use less storage than integers
- C) Hash keys are deterministic, enabling parallel loading without coordination, distributed joins, and idempotent loads
- D) Hash keys are required by modern cloud data warehouses

**✅ Correct Answer: C**

*Explanation: The key benefit is deterministic computation — any ETL process can independently compute the same hash from the same business key without coordinating with a centralized sequence generator. This enables parallel loading from multiple sources, makes loads safely repeatable (idempotent), and distributes evenly across database nodes. Hash keys are actually larger than integers (16 or 32 bytes vs 4-8 bytes), so B is incorrect.*

---

### Question 3

**A customer's phone number changes. In Data Vault, what happens?**

- A) The Hub record for the customer is updated with the new phone number
- B) A new row is inserted into the customer's Satellite with the new phone number, a new load_date, and a new hash_diff
- C) The Link between the customer and their phone number is deleted and re-created
- D) A new Hub record is created with the new phone number as the business key

**✅ Correct Answer: B**

*Explanation: Hubs are immutable — they never change after initial insert. Phone number is a descriptive attribute, so it lives in a Satellite. When it changes, a new Satellite row is inserted with the updated value, a new load_date timestamp, and a new hash_diff (proving the data actually changed). The previous Satellite row remains, preserving the full history. No UPDATE operations occur anywhere.*

---

### Question 4

**Why does Data Vault recommend one Satellite per source system per Hub?**

- A) To reduce storage costs by eliminating duplicate data
- B) To separate different rates of change, enable parallel loading from each source, and maintain source-level auditability
- C) Because each source system requires a different primary key structure
- D) To comply with third normal form (3NF) requirements

**✅ Correct Answer: B**

*Explanation: Different sources update at different rates — CRM might update monthly while billing updates daily. If they shared a Satellite, every billing change would duplicate unchanged CRM fields. Separate Satellites also allow each source's ETL to load independently (parallel loading, no contention) and maintain clear auditability ("What did the billing system tell us about this customer on this date?").*

---

### Question 5

**What problem do Point-in-Time (PIT) tables solve?**

- A) They replace Satellites for storing historical data
- B) They eliminate the need for Links between Hubs
- C) They pre-compute which Satellite record is effective for each Hub key at each snapshot date, replacing expensive correlated subqueries with equi-joins
- D) They encrypt sensitive data at specific points in time

**✅ Correct Answer: C**

*Explanation: Querying a Hub with multiple Satellites at a specific point in time requires correlated subqueries to find the latest Satellite record before the target timestamp — one per Satellite. This is extremely expensive. PIT tables pre-compute these lookups, storing the effective load_date for each Satellite per Hub key per snapshot date. Queries then use simple equi-joins (Hash Key + load_date), which are orders of magnitude faster.*

---

### Question 6

**In a Data Vault architecture, where do business rules get applied?**

- A) In the Raw Vault, as data is loaded into Hubs, Links, and Satellites
- B) In the Staging Layer, before data enters the vault
- C) In the Business Vault, through Calculated Satellites, PIT tables, and Bridge tables
- D) In the Hub, through computed columns derived from the business key

**✅ Correct Answer: C**

*Explanation: The Raw Vault stores data exactly as received from source systems — no business rules, no transformations. This preserves auditability and source fidelity. Business rules (customer segmentation, lifetime value calculations, currency conversions) are applied in the Business Vault layer through Calculated Satellites, PIT tables, and Bridge tables. If a business rule changes, you re-derive in the Business Vault without touching the Raw Vault. The Information Mart (star schemas) is then built from the Business Vault.*

---

### Question 7

**Which architecture is most appropriate for an enterprise with 15+ source systems, frequent regulatory audits, and a BI team that uses Tableau?**

- A) Kimball star schema only — it's the simplest approach
- B) Data Vault Raw Vault only — the BI team can query the vault directly
- C) Data Vault for integration (Raw Vault + Business Vault) with Kimball star schema Information Marts for BI consumption
- D) Third Normal Form (3NF) for everything — it's the most academically correct

**✅ Correct Answer: C**

*Explanation: With 15+ sources, Data Vault's agility (add sources without refactoring) is essential. Regulatory audits require the full lineage and source traceability that the Raw Vault provides. But BI tools like Tableau work best with star schemas, not raw vault structures. The correct architecture is Data Vault for integration and auditability, with Kimball-style Information Marts built from the Business Vault for the BI team. This gives you the best of both worlds.*

---

## Key Takeaways for Interviews

1. **Data Vault separates structure from content.** Hubs and Links define *what exists* and *how things relate* (structure). Satellites define *what we know about them* (content). This separation is what makes Data Vault agile — you can add new descriptive data without changing the structural model.

2. **Hubs are immutable identity registries.** A Hub records that a business entity exists, identified by its business key. Once registered, the row never changes. All descriptive attributes and history go into Satellites.

3. **Links are always many-to-many.** By modeling all relationships as many-to-many from the start, Data Vault avoids painful refactoring when business relationships evolve. Relationship status changes are tracked in Link Satellites.

4. **Satellites are INSERT-only history.** Every change creates a new row with a new `load_date` and `hash_diff`. No UPDATEs, no current flags, no SCD complexity. One Satellite per source per Hub/Link for separation of concerns.

5. **Hash keys unlock parallelism.** Deterministic hashing (MD5/SHA-256) replaces auto-increment sequences, enabling independent parallel loading, distributed joins, and idempotent operations. Always normalize inputs (UPPER, TRIM, COALESCE NULLs) before hashing.

6. **PIT + Bridge tables solve the query problem.** Raw Data Vault is optimized for loading, not querying. PIT tables pre-compute temporal lookups, and Bridge tables pre-walk Link relationships — both sit in the Business Vault to make the presentation layer performant.

7. **Data Vault + Kimball = best of both worlds.** Use Data Vault (Raw Vault → Business Vault) for integration, auditability, and agility. Use Kimball star schemas (Information Marts) for business user consumption. The Raw Vault never changes; business rules live in the Business Vault; BI tools query the marts.

8. **Know when NOT to use Data Vault.** For small projects with few sources and stable requirements, Data Vault adds unnecessary complexity. Kimball-only is simpler and faster to deliver. Data Vault shines when you have 10+ source systems, regulatory requirements, or rapidly changing business needs.
