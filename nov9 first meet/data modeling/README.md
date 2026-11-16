# Data Modeling: OBT vs Star vs Snowflake vs Data Vault 2.0

**Mentorship session:** November 9, 2026  
**Focus:** Data engineering / analytics engineering interview preparation

## TL;DR

- **OBT (One Big Table):** denormalized, consumption-ready wide table; minimal joins for a predefined workload.
- **Star schema:** central fact table at an explicit grain, surrounded by relatively denormalized dimensions; a strong default for reusable BI and metrics.
- **Snowflake schema:** star-like fact table with dimensions normalized into related subdimensions; reduces duplication but increases join complexity.
- **Data Vault 2.0:** integration and history-oriented model using **Hubs, Links, Satellites**; commonly serves as a warehouse integration layer, with star schemas or wide tables downstream.

**Important:** They are not necessarily mutually exclusive. A common architecture is raw ingestion -> Data Vault / cleaned integration -> star schema -> purpose-built OBT or semantic views. Choose based on data sources, change rate, query patterns, history, governance, and maintenance cost.

## Comparison

| Aspect | One Big Table (OBT) | Star schema | Snowflake schema | Data Vault 2.0 |
|---|---|---|---|---|
| Core structure | Wide flattened table per useful grain/use case | Fact + denormalized dimensions | Fact + normalized dimension hierarchies | Hubs (keys), Links (relationships), Satellites (descriptive history) |
| Primary goal | Simple consumption, low join overhead | Understandable, reusable analytics | Reuse/manage dimension hierarchies | Integrate changing sources with traceability and history |
| Typical location | Serving / gold / feature dataset | Analytics / gold / marts | Analytics / gold where hierarchy is valuable | Integration / historical warehouse (often silver-like) |
| Query joins | Few or zero for intended questions | Typically fact-to-dimension joins | More joins, including dimension-to-dimension | Many joins and time-range logic; not ideal for direct dashboards |
| Data duplication | High | Moderate in dimensions | Lower among dimensions | Low repetition of business keys; multiple satellites/versions still consume storage |
| Historical changes | Must be explicitly engineered | SCD Type 2 and fact history possible | SCD handling across normalized dimensions can be intricate | Designed for historized source data, provided correct load patterns are implemented |
| Ease for analysts | Very high for a fixed use case | High | Medium | Low |
| Source changes | Potentially costly to rewrite wide outputs | Moderate effort with conformed dimensions | Moderate-to-high | Designed to absorb sources and schema changes incrementally |
| Biggest risk | Duplication, inconsistent definitions, grain mistakes | Fact grain, SCD complexity, dimension governance | Join complexity and usability | Implementation and operational complexity |

> Query performance is **not automatically** OBT > Star > Snowflake > Vault. Columnar engines, data size, pruning, clustering, caching, join optimization, and workload can change the outcome. Benchmark representative queries.

## Start with the grain (before selecting any model)

Restaurant example: **one row per order line** (not one row per order). Keys: `order_id`, `order_line_id`, `customer_id`, `product_id`; measures: `quantity`, `line_revenue`.

- If you join **order-level total** onto every line, summing that total will overcount.
- If you join a one-to-many `payment` table to order lines without preaggregating or allocating, you can multiply rows.
- Agree on the event/grain, measures, dimensions, and expected update/history behavior before building.

## 1. OBT — One Big Table

A table such as `order_line_analytics(order_line_id, order_id, order_date, customer_name, city, product_name, category, quantity, line_revenue)` repeats customer and product attributes across order lines.

**Pros**
- Easy for downstream users to discover and query; minimal joins for supported analytics.
- Helpful for exports, ML feature datasets and a dashboard with stable, well-defined questions.
- Can offer very good performance when data layout and workload are optimized.

**Cons**
- Repeated descriptive attributes increase storage and update/reprocessing work.
- Changing a shared definition or a dimension attribute can require rebuilding multiple wide datasets.
- Wide, mixed-grain joins can silently double-count metrics; common logic may drift between OBTs.
- Adding many unrelated subject areas produces sparse or hard-to-maintain tables.

**Choose when:** a stable use case needs a flattened output, with upstream reusable definitions and explicit grain.

## 2. Star schema (Kimball dimensional modeling)

`fact_order_line(order_line_id, date_key, customer_key, product_key, quantity, line_revenue)`  
`dim_customer(customer_key, customer_id, name, city, state, effective_from, effective_to)`  
`dim_product(product_key, product_id, product_name, category)`  
`dim_date(date_key, day, month, year)`

Facts record events and measures; dimensions give analysis context. Use **surrogate dimension keys** when tracking historical attribute versions (SCD Type 2). Reuse **conformed dimensions** across marts where appropriate.

**Pros**
- Business-friendly model with clear fact grain and reusable dimensions.
- Supports flexible slicing, grouping, and metric calculation across many dashboards.
- Often works well with BI semantic models and optimized analytical engines.

**Cons**
- Requires joins and deliberate handling of late-arriving dimensions, SCDs and fact updates.
- Denormalized dimensions can repeat hierarchy values and complicate shared ownership.
- Cross-process facts at different grains need careful modeling; never join two facts directly without ensuring compatible aggregation.

**Choose when:** multiple recurring BI/reporting questions share common business entities and measures. **Default recommendation for a curated reporting layer.**

## 3. Snowflake schema

Same `fact_order_line`, but `dim_product(product_key, product_id, product_name, category_key)` references `dim_category(category_key, category_name)`; similarly, `dim_customer` can reference a shared `dim_geography`.

**Pros**
- Reduces repeated attributes in dimension hierarchies.
- Changes to a shared hierarchy can be maintained in a single place.
- Helps when large dimensions contain complex or centrally governed hierarchies.

**Cons**
- More joins and more involved semantic/query logic.
- Harder for casual analysts; joins may hurt performance for some workloads.
- Normalizing tiny dimensions can add complexity without meaningful benefit.

**Choose when:** hierarchy consistency and dimension maintainability outweigh extra complexity. **Do not snowflake purely to save a few bytes.**

## 4. Data Vault 2.0

- **Hub** = distinct durable business key, e.g. `hub_customer(customer_hk, customer_business_key, load_dts, record_source)`.
- **Link** = association among business keys, e.g. `link_order_customer(order_customer_hk, order_hk, customer_hk, load_dts, record_source)`.
- **Satellite** = descriptive attributes + history, e.g. `sat_customer_details(customer_hk, load_dts, hashdiff, name, city, record_source)`.

In a *Raw Vault*, loads prioritize retaining source lineage/history rather than replacing original source facts with current business-rule interpretations. A *Business Vault* can add derived rules, point-in-time (PIT) and bridge structures. Deliver BI-friendly stars/views downstream.

**Pros**
- Strong source lineage and reproducible historical integration when designed and loaded correctly.
- Easier to onboard sources and evolve descriptive attributes without replacing a monolithic model.
- Hub/link/satellite patterns enable parallel, incremental ingestion.

**Cons**
- Many objects, joins, orchestration rules and operational conventions.
- Hard for consumers to query directly; typically needs a presentation model.
- Point-in-time history, duplicate handling, effectivity and integration keys require expert design; **history is not automatic without correct pipelines**.
- Overkill for a few stable sources and simple analytics.

**Choose when:** enterprise-wide integration has many source systems, changing definitions, auditable history and complex lineage requirements.

## Practical decision guide

1. **Who queries it?** Business analysts with open-ended questions -> star. Fixed extract/ML input -> OBT.
2. **How complex are dimension hierarchies?** Stable and meaningful shared hierarchies -> consider snowflake for selected dimensions.
3. **How many sources, and how often do they change?** Many heterogeneous/volatile inputs + audit needs -> consider Data Vault as integration layer.
4. **Do we need as-was vs as-is views?** Specify event time, source changes, effective time, load time, SCD behavior and late-arriving corrections explicitly.
5. **What is the workload/cost?** Benchmark representative dashboards and batch transforms on the actual platform.

### Example architecture

```text
POS / Reservations / CRM
          |
       Bronze (raw)
          |
  Silver (cleaned / integration)
     [optional Data Vault]
          |
    Gold star schema
     /            \
BI semantic model  OBT for ML/export
```

Bronze/Silver/Gold are **pipeline-layer labels**, not mandatory one-to-one mappings to these models.

## Common interview questions and strong answers

**Q1. OBT vs star: which is faster?**  
Neither always wins. OBT eliminates joins for supported questions but can increase scan width and data repetition. Star introduces joins but benefits from narrow facts, reusable dimensions, pruning and mature query optimizers. Measure workloads, latency and refresh cost.

**Q2. Star vs snowflake?**  
Both use fact/dimension modeling. Star keeps dimensional attributes together for easier queries; snowflake normalizes hierarchical dimension attributes for centralized maintenance at the cost of joins.

**Q3. Is Data Vault a replacement for star?**  
Usually no. Vault optimizes integration, adaptability, lineage and history; star optimizes consumption. Both can coexist.

**Q4. What mistakes would you look for?**  
Wrong fact grain, fanout after joining orders/order lines/payments, mismatched facts, missing unknown dimension members, incorrect SCD Type 2 effective dates, double-counted revenue, inconsistent business keys, and silently dropped late-arriving records.

**Q5. How would you model a new system?**  
Clarify business questions -> identify processes and fact grain -> list measures/dimensions -> choose integration approach based on source complexity -> select a presentation model -> define historical semantics and tests -> benchmark with actual queries.

## Source reading (Nov 9 mentorship reference)

These notes synthesize and extend the following user-provided DataArchitectStudio reference materials; they are an original summary, not a reproduction:

- [Data Warehouse Modelling Approaches (seven patterns)](https://drive.google.com/file/d/1t7d4fqHvOOwLTZqSBWRbbqRnh8VJwRrO/view)
- [3 Ways to Structure a Data Warehouse (Kimball / Inmon / Data Vault)](https://drive.google.com/file/d/1MOlkL0WIaAsyrhquvEZbUtxwN4-6QwhJ/view)
- [Data Modeling Fundamentals](https://drive.google.com/file/d/1psfM0plRQ_S3cQPoFeHJIVV6UFNhDh-W/view)
- [Full reference folder](https://drive.google.com/drive/folders/10wl9Zq6Rsh1717Isye7ifjs47Fm2xyl4)
