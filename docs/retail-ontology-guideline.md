# Retail Enterprise Ontology Guideline

## Purpose

This guideline defines a practical ontology architecture for a retail enterprise
using the documented RFR model and the Site-Level Pricing SDD as source material.
It is a modeling recommendation, not a claim that either source document already
asserts this complete ontology.

The central recommendation is to build a layered ontology:

1. A reusable enterprise/core ontology.
2. Retail bounded-context ontologies.
3. Organization-specific extensions.

Do not create one giant ontology that mirrors SAP, Blue Yonder, Pricefx, or any
other source system.

## 1. Source interpretation

### RFR as the enterprise relationship foundation

The RFR model provides broad retail concepts and, more importantly, valuable
relationships:

- Vendor, distribution center, and store are location types.
- SKU associates an item with a location.
- A network lane combines source, destination, and transportation mode.
- Sourcing connects an item, source, destination, and transportation mode.
- Demand includes forecasts and order-related demand.
- Supply includes inventory, purchase orders, stock transfers, and recommended
  shipments.
- Planning includes forecasting, replenishment, allocation, deployment, and
  safety stock.

These observations support the proposed semantic model below. Predicate names
are recommendations and must be confirmed by business and data owners.

### Pricing SDD as a bounded-context foundation

The Site-Level Pricing SDD is strong pricing ontology material. It defines or
describes concepts such as:

- Site
- Product
- Price
- National Price
- Price List
- Price Zone
- Site-Level Price
- Price Exception
- Price Condition
- Price Determination
- Promotion
- Pricefx, SAP, and POS integrations

Its three-level pricing model is especially useful:

```text
National price -> price list / zone price -> site-level price
```

The more-specific applicable price takes precedence, subject to effective dates,
conditions, exceptions, and other governed rules.

## 2. Layered architecture

```text
RETAIL ENTERPRISE ONTOLOGY
├── Enterprise/Core
│   ├── Product
│   ├── Location
│   ├── Customer
│   ├── Supplier
│   ├── Organization
│   ├── Channel
│   ├── Order
│   ├── Inventory
│   ├── Price
│   ├── Promotion
│   ├── Transaction
│   └── Time
├── Retail bounded contexts
│   ├── Pricing
│   ├── Supply Chain
│   ├── Sales
│   ├── Service Operations
│   ├── Customer Journey
│   └── Promotions
└── Organization extension
    └── Discount Tire-specific vocabulary
```

### Layer 1: enterprise/core ontology

Place only concepts that are reusable across multiple domains in the core:

| Class | Guidance |
|---|---|
| `Product` | A sellable or serviceable offering; keep source-system product identifiers as mappings. |
| `Item` | A specific product representation used by supply and catalog processes. |
| `Location` | A physical, logical, or operational place. |
| `Store` | A customer-facing retail location. |
| `Supplier` | An organization that provides products or services. |
| `Customer` | A person or organization receiving retail goods or services. |
| `Organization` | A party with operational or commercial responsibility. |
| `Channel` | A sales, service, or fulfillment interaction channel. |
| `Order` | A request for products or services. |
| `Inventory` | A stock position or inventory quantity associated with an item and location. |
| `Price` | A monetary amount with currency, applicability, and validity. |
| `Promotion` | A commercial incentive or offer. |
| `Transaction` | A recorded commercial or operational event. |
| `Time` | Effective, event, promise, and validity periods. |

Avoid putting RFR planning terms or pricing implementation terms in the core
unless they are demonstrably shared by multiple bounded contexts.

### Layer 2: retail bounded contexts

Use context-specific namespaces and definitions:

```text
retail:Product
retail:Location
retail:Customer
retail:Store

pricing:Price
pricing:PriceRule
pricing:PriceList
pricing:PriceZone
pricing:SiteLevelPrice
pricing:PriceException
pricing:PriceCondition
pricing:PriceDetermination

supply:SKU
supply:Inventory
supply:Forecast
supply:Demand
supply:NetworkLane
supply:Sourcing
supply:Replenishment
supply:RecommendedShipment
supply:PurchaseOrder
supply:StockTransferOrder

sales:SalesOrder
sales:Invoice
sales:Payment

service:Appointment
service:WorkOrder
service:Service

promotion:Promotion
```

The same real-world entity may be referenced by several contexts, but each
context should own its definitions and rules. Cross-context links should use
stable identifiers and explicit mappings rather than duplicating source-system
classes.

### Layer 3: organization-specific extensions

Only add organization-specific vocabulary after the core and bounded contexts
are stable. Examples include:

```text
dt:PitPassStore
dt:FleetCenter
dt:VASTStore
dt:PriceListZone
dt:RetailPriceCondition
```

These terms must have an owner, definition, source, lifecycle, and mapping to
the appropriate generic class. Do not make a source-system label a business
concept without confirming its meaning.

## 3. RFR-derived conceptual model

```text
RetailEnterprise
├── Product
│   └── Item
├── Location
│   ├── Vendor
│   ├── DistributionCenter
│   │   ├── CrossDockDC
│   │   └── StockAndHoldDC
│   ├── Store
│   ├── FleetCenter
│   └── EcommerceLocation
├── SKU
│   ├── represents -> Item
│   └── at -> Location
├── Inventory
│   └── heldAt -> Location
├── NetworkLane
│   ├── source -> Location
│   ├── destination -> Location
│   └── uses -> TransportationMode
├── Sourcing
│   ├── for -> Item
│   ├── uses -> NetworkLane
│   └── supports -> SKU
├── Demand
│   ├── Forecast
│   ├── CustomerOrder
│   └── SupplementalOrder
├── Supply
│   ├── Inventory
│   ├── PurchaseOrder
│   ├── StockTransferOrder
│   └── RecommendedShipment
└── Planning
    ├── Forecasting
    ├── Replenishment
    ├── Allocation
    ├── Deployment
    └── SafetyStock
```

`Vendor` may be modeled as a `Supplier` and, where the source model requires it,
as a `Location` subtype or a party associated with a location. This distinction
must be resolved by the business definition; do not infer it solely from a
diagram.

## 4. Recommended relationship vocabulary

Relationships are the primary value of the RFR-derived model. Prefer explicit
object properties over opaque JSON attributes or source-specific foreign keys.

| Subject | Predicate | Object | Meaning |
|---|---|---|---|
| `Item` | `soldAt` | `Store` | Item is offered or sold at a store. |
| `Item` | `storedAt` | `DistributionCenter` | Item is stored at a distribution center. |
| `Item` | `purchasedFrom` | `Supplier` | Item is sourced from a supplier. |
| `SKU` | `represents` | `Item` | SKU is the operational representation of an item. |
| `SKU` | `at` | `Location` | SKU is associated with a location. |
| `Inventory` | `for` | `SKU` | Inventory position concerns a SKU. |
| `Inventory` | `heldAt` | `Location` | Inventory is held at a location. |
| `NetworkLane` | `source` | `Location` | Lane origin. |
| `NetworkLane` | `destination` | `Location` | Lane destination. |
| `NetworkLane` | `uses` | `TransportationMode` | Mode used on the lane. |
| `Sourcing` | `for` | `Item` | Sourcing policy concerns an item. |
| `Sourcing` | `uses` | `NetworkLane` | Sourcing uses a network lane. |
| `Demand` | `for` | `SKU` | Demand concerns a SKU. |
| `Forecast` | `predicts` | `Demand` | Forecast predicts demand. |
| `Replenishment` | `respondsTo` | `Demand` | Replenishment responds to demand. |
| `Replenishment` | `produces` | `RecommendedShipment` | Planning produces a recommended shipment. |
| `StockTransferOrder` | `from` | `DistributionCenter` | Transfer origin. |
| `StockTransferOrder` | `to` | `Store` | Transfer destination. |

Validate predicate direction, cardinality, and inverse properties with domain
owners before publishing the ontology.

## 5. Pricing ontology guidance

Model pricing as a decision context, not merely as a numeric product attribute.
A price should normally include:

- the product or SKU it applies to;
- the site, price zone, channel, or customer scope;
- currency and amount;
- effective start and end time;
- price list or rule source;
- conditions and exceptions;
- determination priority;
- source-system and provenance identifiers.

Recommended relationships include:

```text
PriceList ──appliesTo──────> PriceZone
PriceList ──defines────────> Price
Site ──belongsTo───────────> PriceZone
Price ──forProduct─────────> Product
PriceException ──overrides─> Price
PriceDetermination ──selects -> Price
PriceDetermination ──for───> Site
```

Represent precedence as a governed rule or determination result. Do not encode
precedence only through subclassing, and do not assume that the most-specific
price wins when an inactive, unauthorized, or otherwise invalid condition applies.

## 6. Modeling rules for OntoBricks

### Namespaces and identifiers

- Use one approved, stable base URI per ontology module.
- Use distinct prefixes for core, retail contexts, and organization extensions.
- Keep source identifiers as data properties or mapping keys; do not use labels
  as identifiers.
- Reuse one canonical URI for one real-world entity, even when source systems
  provide multiple identifiers.
- Record source-system identity and crosswalks explicitly.

### OWL/RDFS versus SHACL and rules

Use OWL/RDFS for stable semantics:

- subclass relationships;
- domain and range;
- inverse or transitive properties where justified;
- stable labels and definitions.

Use SHACL for operational quality constraints:

- required fields and cardinality;
- datatypes and code lists;
- valid currency and units;
- identifier formats;
- effective-date consistency;
- cross-entity integrity.

Use SPARQL CONSTRUCT, SWRL, decision tables, or application rules for governed
operational behavior such as price determination, replenishment, and inventory
availability. Keep rules versioned and explainable.

### Provenance and fact status

Separate asserted, inferred, predicted, and user-provided facts. Every inferred
or predicted fact should carry, where applicable:

- source references;
- rule or model version;
- generation timestamp;
- confidence;
- validity interval.

Do not use ontology inference as a substitute for authorization, financial
settlement, safety approval, or regulatory compliance.

## 7. Governance checklist

Before adding a class or property, answer:

1. Which bounded context owns the term?
2. Is the term a business concept or merely a source-system artifact?
3. What is the authoritative definition and data owner?
4. What are the URI, label, scope, domain, and range?
5. Is the relationship direction unambiguous?
6. What are the cardinality, lifecycle, and effective-date rules?
7. Which source tables and identifiers map to it?
8. Which SHACL shapes validate it?
9. Is the fact asserted, inferred, predicted, or user-provided?
10. What privacy, authorization, and downstream-impact controls apply?

## 8. Suggested implementation sequence

1. Approve core definitions for product, item, location, store, supplier,
   customer, order, inventory, price, promotion, and time.
2. Implement RFR supply concepts: SKU, network lane, sourcing, demand, supply,
   and planning.
3. Implement pricing concepts and the three-level price determination model.
4. Add mappings, provenance, and SHACL quality gates.
5. Validate representative cross-context journeys: forecast to replenishment,
   inventory to shipment, and product to site price.
6. Add organization-specific extensions only where a generic concept cannot
   express the governed business meaning.
7. Expose read-only graph retrieval before enabling actions that change prices,
   inventory, orders, appointments, or payments.

## 9. Scope and limitations

This guideline is derived from the supplied interpretation of the RFR and
pricing documents. It should be treated as a design baseline until business
owners approve definitions, namespaces, mappings, cardinalities, and rules.
The proposed predicate names and class hierarchy are not asserted facts from
those documents.