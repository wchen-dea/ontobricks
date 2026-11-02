# OntoBricks AI Ontology Context

**Purpose:** compact, grounding reference for AI systems that generate, inspect,
query, map, validate, or explain OntoBricks ontologies and knowledge graphs.

**Repository basis:** this document reflects the repository's implemented RDF,
OWL, RDFS, SHACL, SPARQL, mapping, reasoning, graph-storage, GraphQL, and MCP
patterns. The Discotun Tire section is a proposed domain blueprint: the
repository does not contain a dedicated Discotun Tire ontology or authoritative
retail vocabulary.

## 1. OntoBricks mental model

An OntoBricks **domain** combines:

1. An ontology: classes, datatype properties, object properties, labels,
   comments, inheritance, and property characteristics.
2. Databricks metadata: Unity Catalog tables, columns, and uploaded documents.
3. Mappings: SQL/R2RML instructions that turn source rows into RDF triples.
4. A knowledge graph: flat `(subject, predicate, object)` triples, optionally
   enriched with inferred triples.
5. Quality and reasoning: SHACL shapes, OWL 2 RL, SWRL, SPARQL CONSTRUCT rules,
   decision tables, and aggregate rules.
6. AI access: MCP tools, GraphQL schema/query, ontology assistants, OWL
   generation, auto-mapping, and graph chat.

The normal pipeline is:

```text
source tables + documents
        -> ontology design
        -> entity/relationship mappings
        -> triple-store synchronization
        -> reasoning and SHACL validation
        -> GraphQL, MCP, graph viewer, and AI answers
```

## 2. RDF and URI conventions

### RDF fundamentals

RDF represents facts as triples:

```text
subject  predicate  object
```

- **Subject:** usually an entity URI.
- **Predicate:** a URI identifying a relationship or attribute.
- **Object:** another URI for relationships, or a literal for attributes.
- `rdf:type` (written `a` in Turtle) asserts class membership.
- Literals should use explicit datatypes where practical, such as
  `xsd:date`, `xsd:decimal`, and `xsd:boolean`.

OntoBricks accepts Turtle, RDF/XML, N3, N-Triples, and JSON-LD through its
flexible RDF parser. URI local names are extracted from either the final `#`
fragment or final `/` path segment.

### Recommended namespace pattern

Use a stable domain base URI and one consistent separator. Both `#` and `/`
appear in repository compatibility logic, but a new ontology should choose one
and use it consistently.

```turtle
@prefix :     <https://example.com/discount-tire/ontology#> .
@prefix rdf:  <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix owl:  <http://www.w3.org/2002/07/owl#> .
@prefix xsd:  <http://www.w3.org/2001/XMLSchema#> .
@prefix sh:   <http://www.w3.org/ns/shacl#> .
```

> Replace the example namespace with the approved Discotun Tire namespace
> before production use. Do not copy the misspelled example above; use a valid
> > URI such as `https://example.com/discount-tire/ontology#` only after the
> organization approves its canonical spelling.

### URI rules for AI systems

- Treat full URIs as authoritative; labels and local names are display hints.
- Do not invent a URI when the ontology already supplies one.
- Resolve prefix-less SPARQL names against the domain base URI.
- Prefer object properties for references to named entities; use datatype
  properties only for literal values.
- Use one canonical URI per real-world identifier. If source systems use
  multiple URI patterns, preserve source identity with explicit mapping or
  reconciliation metadata rather than silently merging entities.

## 3. OWL/RDFS ontology model

### Core vocabulary

| Construct | Meaning in OntoBricks |
|---|---|
| `owl:Ontology` | Ontology metadata resource |
| `owl:Class` | Entity type; becomes a GraphQL type and graph entity type |
| `rdfs:subClassOf` | Inheritance; inherited properties and inferred type relationships |
| `owl:ObjectProperty` | URI-to-URI relationship |
| `owl:DatatypeProperty` | URI-to-literal attribute |
| `rdfs:domain` | Expected subject class |
| `rdfs:range` | Expected object class or datatype |
| `rdfs:label` | Human-readable name and preferred display label |
| `rdfs:comment` | Definition or usage guidance |
| `owl:inverseOf` | Semantically reverse relationship |
| `owl:FunctionalProperty` | At most one value per subject |
| `owl:InverseFunctionalProperty` | Object value identifies at most one subject |
| `owl:TransitiveProperty` | Relationship can be transitively expanded |
| `owl:SymmetricProperty` | Reverse direction is also entailed |

A repository fixture demonstrates the expected pattern: `Customer`, `Order`,
and `Product` are classes; `hasOrder` is an object property with Customer as
domain and Order as range; `firstName`, `lastName`, and `orderDate` are datatype
properties.

### Modeling rules for AI generation

- Model real-world entities as classes, not one class per source column or
  attribute value.
- Name classes with specific domain terms; avoid generic placeholders such as
  `Thing`, `Object`, `Item`, or `Entity`.
- Name properties with relationship semantics only (`hasOrder`, `containsItem`,
  `soldAt`), not embedded domain names such as `customerHasOrder`.
- Add labels, comments, domain, and range for every public class/property.
- Do not create redundant direct links to both a parent and its grandparent.
- Avoid isolated class trees, unnecessary single-child hierarchy levels, and
  duplicate synonym properties.
- Add inverse properties when the reverse direction is useful.
- Promote identifier-like strings to object properties when they refer to a
  modeled entity.
- Do not assert disjointness between a class and its subclass.

## 4. SHACL data quality

SHACL describes constraints over instance data. OntoBricks supports in-memory
and SQL-backed validation and uses shapes for completeness, cardinality,
uniqueness, consistency, conformance, and structural quality.

Common constraints include:

| Constraint | Example use |
|---|---|
| `sh:minCount` / `sh:maxCount` | Required or single-valued identifier |
| `sh:datatype` | Decimal price, date, boolean flag |
| `sh:class` | Relationship points to a specific class |
| `sh:pattern` | SKU or store-code format |
| `sh:minInclusive` / `sh:maxInclusive` | Non-negative price, valid tread depth |
| `sh:minLength` / `sh:maxLength` | Identifier or description bounds |
| `sh:hasValue` | Required status or fixed category |
| `sh:severity` | Violation, warning, or informational result |

AI systems should distinguish **ontology semantics** from **data quality
constraints**: OWL describes meaning and inference; SHACL reports whether data
conforms. A failed SHACL check is not automatically an OWL contradiction.

## 5. SPARQL in OntoBricks

### Query forms

The local RDFLib runner returns:

- `SELECT`: variable columns and row dictionaries.
- `CONSTRUCT` and `DESCRIBE`: `subject`, `predicate`, and `object` rows.

SPARQL is also used for rules. A rule must be a `CONSTRUCT { ... } WHERE { ... }`
query. The SQL-backed rule engine translates supported triple patterns and
simple filters into queries over the flat triple table.

### Safe query pattern

```sparql
PREFIX : <https://example.com/discount-tire/ontology#>
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>

SELECT ?tire ?sku ?width
WHERE {
  ?tire rdf:type :Tire .
  ?tire :sku ?sku .
  ?tire :sectionWidth ?width .
  FILTER(?width >= 205)
}
LIMIT 100
```

### Rule pattern

```sparql
PREFIX : <https://example.com/discount-tire/ontology#>

CONSTRUCT {
  ?tire :requiresInspection true .
}
WHERE {
  ?tire a :Tire .
  ?tire :treadDepth ?depth .
  FILTER(?depth < 4)
}
```

Before issuing a query, an AI system should:

1. Discover the active domain and its base URI.
2. Inspect classes/properties or the GraphQL schema.
3. Use exact property URIs and existing class names.
4. Add a bounded `LIMIT` and avoid unconstrained traversal.
5. Never emit write keywords (`DROP`, `DELETE`, `INSERT`, `CREATE`, `CLEAR`,
   `LOAD`, `COPY`, `MOVE`, or `ADD`) through graph-chat query tools.

The active graph-chat tool set prefers `describe_entity` and `query_graphql` for
full-graph access because the legacy `run_sparql` path queries the warehouse
Delta view and may not see inferred triples.

## 6. Mappings and graph storage

### Source-to-graph mapping

Entity mappings bind a class to a table or SQL query and define:

- logical table or SQL query;
- subject URI template and identifier column;
- optional label column;
- datatype-property-to-column mappings;
- relationship mappings using parent triples maps and join conditions.

R2RML is the W3C interchange format used for generated and parsed mappings.
Mappings should be deterministic, preserve source identifiers, and document
joins that can multiply rows.

### Triple storage

The canonical physical shape is:

```text
subject TEXT, predicate TEXT, object TEXT, datatype TEXT, lang TEXT
```

Lakebase Postgres and Unity Catalog Delta are SQL-oriented backends; Neo4j is a
pluggable native graph backend. Indexes favor subject/predicate and
predicate/object access. Materialized inferred triples are graph facts with
provenance and should not be confused with source assertions.

## 7. Reasoning and inference

OntoBricks supports:

- **OWL 2 RL:** forward-chaining closure over ontology or small ontology-plus-
  instance graphs; intended for type, hierarchy, and property entailments.
- **SWRL:** Horn-style antecedent/consequent rules compiled for graph engines.
- **SPARQL CONSTRUCT:** query-driven inferred triples.
- **Decision tables:** explicit conditional classifications.
- **Aggregate rules:** count, sum, average, minimum, and maximum thresholds.

Inference guidance:

- Mark inferred facts as derived and retain provenance when presenting them.
- Do not present an inferred fact as a source assertion.
- Use existing ontology classes in rule consequents; do not invent a class only
  to make a rule sound plausible.
- Ensure rule variables are bound in the antecedent.
- Keep large-scale reasoning in SQL/graph infrastructure; RDFLib OWL closure is
  intended for small graphs.

## 8. Proposed general retail ontology

This is a reusable modeling baseline, not a repository-provided vocabulary.

### Candidate classes

`RetailOrganization`, `Store`, `DistributionCenter`, `Supplier`, `Customer`,
`Product`, `ProductVariant`, `Brand`, `Category`, `InventoryItem`, `Inventory-
Snapshot`, `Price`, `Promotion`, `SalesOrder`, `OrderLine`, `Invoice`, `Payment`,
`Shipment`, `Return`, `Warranty`, `ServiceAppointment`, and `Employee`.

### Candidate relationships

| Property | Domain → range | Meaning |
|---|---|---|
| `hasStore` | organization → store | Organization operates a store |
| `hasSupplier` | organization → supplier | Supplier relationship |
| `hasBrand` | product → brand | Product brand |
| `hasCategory` | product → category | Product classification |
| `hasVariant` | product → product variant | Variant decomposition |
| `stocks` | store → product | Store carries a product |
| `hasInventorySnapshot` | store → inventory snapshot | Point-in-time inventory |
| `containsLine` | sales order → order line | Order contents |
| `referencesProduct` | order line → product | Product sold in line |
| `placedBy` | sales order → customer | Ordering customer |
| `soldAt` | sales order → store | Selling location |
| `hasPromotion` | product/store → promotion | Applied promotion |
| `fulfills` | shipment → sales order | Shipment fulfillment |
| `hasReturn` | sales order → return | Return associated with order |
| `coveredBy` | product → warranty | Warranty coverage |

### Candidate datatype properties

Use explicit datatypes for `sku`, `name`, `description`, `quantity`, `unitPrice`,
`currency`, `orderDate`, `status`, `availableFrom`, `availableTo`, and
`externalId`. Do not use a string property for a product, store, or supplier
reference when that reference is modeled as an entity.

## 9. Proposed Discotun Tire domain ontology

This section is a domain-specific starting point for Discount Tire-style tire
retail and service operations. Confirm names, business definitions, source
systems, and the canonical namespace with domain owners before implementation.

### Core classes

#### Product and tire catalog

- `TireProduct`: sellable tire family or catalog product.
- `TireVariant`: exact fitment/specification variant.
- `TireBrand`, `TireLine`, `TireCategory`.
- `TireSpecification`: measurable or coded tire specification.
- `Vehicle`, `VehicleMake`, `VehicleModel`, `VehicleTrim`, `VehicleYear`.
- `Fitment`: compatibility assertion between a tire and vehicle configuration.
- `Wheel`: wheel/rim product or installed wheel entity.

#### Retail and service operations

- `Customer`, `CustomerVehicle`, `Store`, `DistributionCenter`, `Supplier`.
- `InventoryItem`, `InventorySnapshot`, `PurchaseOrder`, `SalesOrder`,
  `OrderLine`, `Invoice`, `Payment`, `Shipment`, `Return`.
- `ServiceAppointment`, `ServiceOrder`, `ServiceOperation`, `Technician`.
- `Warranty`, `WarrantyClaim`, `RoadHazardProtection`, `Inspection`.

### Tire-specific relationships

| Property | Domain → range | Meaning |
|---|---|---|
| `hasTireVariant` | tire product → tire variant | Catalog decomposition |
| `hasSpecification` | tire variant → tire specification | Tire measurements/codes |
| `hasFitment` | tire variant → fitment | Vehicle compatibility |
| `fitsVehicle` | fitment → vehicle configuration | Compatible vehicle |
| `ownedBy` | customer vehicle → customer | Vehicle ownership |
| `availableAt` | tire variant → store | Sellable at location |
| `stockedAt` | inventory item → store/DC | Physical inventory location |
| `requiresService` | sales/order line → service operation | Installation/service need |
| `scheduledFor` | service appointment → customer vehicle | Appointment target |
| `performedAt` | service appointment → store | Service location |
| `performedBy` | service operation → technician | Technician assignment |
| `coveredByWarranty` | tire variant → warranty | Warranty coverage |
| `hasInspection` | customer vehicle → inspection | Inspection record |
| `hasReplacement` | service order → tire variant | Replacement tire |
| `suppliedBy` | tire product → supplier | Supply relationship |

### Tire-specific attributes

Use datatype properties for `sku`, `upc`, `brandName`, `lineName`, `season`,
`vehicleType`, `sectionWidth`, `aspectRatio`, `rimDiameter`, `loadIndex`,
`speedRating`, `construction`, `runFlatIndicator`, `treadDepth`, `tirePressure`,
`quantity`, `unitPrice`, `currency`, `dotCode`, `manufactureDate`, `mileage`,
`appointmentDateTime`, and `serviceStatus`.

Recommended datatype choices:

- measurements and prices: `xsd:decimal`;
- counts and indices: `xsd:integer`;
- dates: `xsd:date`;
- timestamps: `xsd:dateTime`;
- yes/no indicators: `xsd:boolean`;
- controlled codes and labels: `xsd:string` unless a controlled vocabulary is
  modeled as a class.

### Tire data-quality examples

- Every `TireVariant` has exactly one `sku`.
- `sectionWidth`, `aspectRatio`, and `rimDiameter` are present and positive.
- `loadIndex` and `speedRating` conform to approved code lists.
- A `Fitment` references exactly one tire variant and one vehicle configuration.
- An `InventoryItem` has one location and a non-negative quantity.
- An `OrderLine` has one product, a positive quantity, and a non-negative unit
  price.
- A `ServiceAppointment` has a customer vehicle, store, scheduled time, and
  status.
- `treadDepth` must use a documented unit and valid range; do not compare
  values across units without normalization.

### Example tire query

```sparql
PREFIX : <https://example.com/discount-tire/ontology#>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>

SELECT ?store ?sku ?quantity
WHERE {
  ?item a :InventoryItem .
  ?item :sku ?sku .
  ?item :stockedAt ?store .
  ?item :quantity ?quantity .
  FILTER(?quantity > 0)
}
ORDER BY DESC(?quantity)
LIMIT 100
```

## 10. AI operating procedure

For any ontology or graph question:

1. **Identify the domain and version.** MCP clients must select a domain before
   querying; only the active API/MCP version is authoritative for published use.
2. **Discover the schema.** Use `describe_ontology`, `list_entity_types`, or
   GraphQL schema introspection before selecting names.
3. **Ground every term.** Reuse exact class/property URIs, labels, ranges, and
   source descriptions. Ask for clarification when a term is absent or
   ambiguous.
4. **Separate layers.** Distinguish ontology facts, source data, inferred facts,
   SHACL violations, and business-rule proposals.
5. **Bound retrieval.** Prefer targeted entity descriptions or GraphQL fields;
   use small traversal depth and result limits.
6. **Explain provenance.** State whether an answer came from a source triple,
   an inferred triple, a mapping, or a quality/rule result.
7. **Never invent data.** Empty, missing, or unmodeled values must be reported as
   unavailable—not filled with plausible retail or tire values.
8. **Treat actions as proposals.** Unity Catalog actions require separate UI
   confirmation; an AI tool must not claim that an action was executed merely
   because it proposed one.

## 11. Repository implementation map

| Capability | Primary repository area |
|---|---|
| RDF parsing and URI helpers | `src/back/core/w3c/rdf_utils.py` |
| OWL/RDFS | `src/back/core/w3c/owl/`, `src/back/core/w3c/rdfs/` |
| SHACL | `src/back/core/w3c/shacl/` |
| SPARQL query and translation | `src/back/core/w3c/sparql/` |
| R2RML | `src/back/core/w3c/r2rml/` |
| Reasoning | `src/back/core/reasoning/` |
| Triple/graph backends | `src/back/core/graphdb/` |
| Ontology domain operations | `src/back/objects/ontology/` |
| Mapping domain operations | `src/back/objects/mapping/` |
| Knowledge-graph operations | `src/back/objects/digitaltwin/` |
| OWL generation and ontology assistance | `src/agents/agent_owl_generator/`, `src/agents/agent_ontology_assistant/` |
| Graph chat and AI access | `src/agents/agent_dtwin_chat/`, `src/mcp-server/` |

## 12. Synonym glossary

Use these synonyms for search and AI grounding, while preserving the canonical
ontology term in generated RDF, SPARQL, mappings, and explanations.

| Canonical term | Common synonyms and aliases |
|---|---|
| ontology | semantic model, domain model, vocabulary |
| class | entity type, concept, category |
| property | predicate, field, attribute, relationship |
| object property | relationship, link, entity reference |
| datatype property | attribute, literal property, scalar field |
| individual | instance, entity, resource |
| knowledge graph | RDF graph, semantic graph, graph dataset |
| triple | RDF statement, fact, subject-predicate-object statement |
| mapping | source mapping, semantic mapping, R2RML mapping |
| inference | reasoning, entailment, derived knowledge |
| validation | SHACL validation, conformance checking, data-quality checking |
| retail store | shop, branch, sales location, store location |
| distribution center | warehouse, fulfillment center, DC |
| sales order | customer order, order, purchase order* |
| order line | line item, order item, basket item |
| tire | tyre, tire product, tire item |
| tire variant | tire specification, tire configuration, SKU variant |
| fitment | vehicle compatibility, application, vehicle match |
| wheel | rim, wheel assembly |
| tread depth | tread measurement, remaining tread, tread wear depth |
| service appointment | installation appointment, booking, service booking |
| store inventory | stock, on-hand quantity, available inventory |
| SKU | stock keeping unit, product code, item number |

\* `PurchaseOrder` is a distinct procurement concept in the proposed retail
ontology. Do not treat it as a synonym for `SalesOrder` when modeling data.

## 13. Abbreviation glossary

| Abbreviation | Meaning |
|---|---|
| AI | Artificial intelligence |
| API | Application programming interface |
| DC | Distribution center |
| DOT | Department of Transportation tire identification code |
| GraphQL | Graph Query Language |
| IRI | Internationalized Resource Identifier |
| JSON-LD | JavaScript Object Notation for Linked Data |
| MCP | Model Context Protocol |
| N3 | Notation3 RDF serialization |
| OWL | Web Ontology Language |
| RDF | Resource Description Framework |
| RDFS | RDF Schema |
| R2RML | RDB to RDF Mapping Language |
| SKU | Stock keeping unit |
| SHACL | Shapes Constraint Language |
| SPARQL | SPARQL Protocol and RDF Query Language |
| SQL | Structured Query Language |
| SWRL | Semantic Web Rule Language |
| URI | Uniform Resource Identifier |
| URL | Uniform Resource Locator |
| UPC | Universal Product Code |
| W3C | World Wide Web Consortium |
| XML | Extensible Markup Language |
| XSD | XML Schema Definition vocabulary |

## 14. Enterprise implementation blueprint for Discount Tire AI systems

This is a proposed implementation blueprint for a Discount Tire-style tire
retail and service enterprise. It is not an authoritative Discount Tire
ontology, does not claim access to proprietary systems, and must be reviewed by
business, legal, security, data-governance, and domain experts before use.

### 14.1 Establish scope and governance first

Define the ontology charter before creating classes or mappings:

1. **Business scope:** catalog, vehicle fitment, stores, inventory, pricing,
   ecommerce, orders, installation, warranties, returns, suppliers, logistics,
   customer service, and finance.
2. **System boundaries:** identify the systems of record for each fact. For
   example, catalog owns product specifications, inventory owns on-hand
   quantity, and service systems own appointment status.
3. **Ownership:** assign a product owner, ontology steward, data owners, and
   approvers for each module.
4. **Lifecycle:** version the ontology and mappings; record deprecations,
   effective dates, source lineage, and backward-compatibility policy.
5. **Privacy:** classify customer, vehicle, payment, location, and employee data;
   expose only the minimum fields required for each AI use case.

Every canonical class and property should have an owner, definition, status,
source-system reference, sensitivity classification, and example query.

### 14.2 Use a modular enterprise ontology

Keep the core small and compose specialized modules:

| Module | Primary concepts |
|---|---|
| Foundation | identifiers, organization, location, time, provenance, units |
| Product catalog | tire product, variant, brand, specification, warranty |
| Vehicle fitment | vehicle, make, model, trim, year, fitment, restrictions |
| Store operations | store, service bay, technician, appointment, operation |
| Inventory | item, location, snapshot, transfer, reservation, availability |
| Commerce | cart, sales order, line, price, promotion, payment, invoice |
| Supply chain | supplier, purchase order, shipment, receipt, distribution center |
| Customer | customer, consent, vehicle ownership, communication preference |
| Quality and safety | inspection, defect, recall, warranty claim, service result |
| Analytics | metric, event, cohort, forecast, recommendation, business rule |

Import only the modules needed by an application. Keep operational assertions,
inferred facts, analytics observations, and reference vocabularies in separate
named graphs or provenance partitions.

### 14.3 Define canonical identifiers and identity resolution

Use stable IRIs that do not embed mutable labels:

```text
https://ontology.example.com/discount-tire/id/tire-variant/{source}/{id}
https://ontology.example.com/discount-tire/id/store/{source}/{id}
https://ontology.example.com/discount-tire/id/vehicle/{source}/{id}
```

For each entity, define:

- a global IRI;
- the source-system identifier and source authority;
- identifier type, validity interval, and confidence;
- external identifiers such as SKU, UPC, DOT code, or vehicle code;
- merge, split, and supersession relationships.

Do not merge records solely because labels match. Use deterministic keys first,
then governed probabilistic matching with confidence and review status. Preserve
the original identifiers and never overwrite source identity during
reconciliation.

### 14.4 Model temporal and unit semantics explicitly

Enterprise tire data changes over time. Represent effective and observed times
for prices, inventory, fitment, promotions, warranties, appointments, and
vehicle ownership. Distinguish:

- **valid time:** when a fact is true in the business domain;
- **transaction time:** when the enterprise recorded the fact;
- **observation time:** when a sensor, inspection, or process observed it.

Represent measurement value, unit, and normalization method together. A tread
depth of `4` is not interpretable without a unit and observation context. Apply
the same rule to pressure, mileage, dimensions, currency, weight, and
temperature.

### 14.5 Implement mappings with source ownership and lineage

For every mapping, document:

1. source table, query, API, or event;
2. source owner and refresh cadence;
3. subject IRI template and identity key;
4. column-to-property mapping and datatype conversion;
5. join condition and expected cardinality;
6. null, invalid, duplicate, and late-arriving-record behavior;
7. valid/transaction time extraction;
8. lineage, load timestamp, and mapping version.

Use deterministic mappings and idempotent synchronization. A re-run must not
create duplicate entities or silently delete facts. Quarantine records that
cannot be mapped safely and expose their error reason to data stewards.

### 14.6 Add enterprise constraints and reasoning carefully

Use SHACL for required fields, datatypes, code lists, cardinality, units,
identifier formats, and cross-entity integrity. Use OWL/RDFS for stable
subclass, domain, range, inverse, and transitive semantics. Use SPARQL
CONSTRUCT, SWRL, or decision tables for governed operational rules such as:

- deriving `AvailableForSale` from inventory, reservation, and store status;
- identifying a vehicle-compatible tire from approved fitment facts;
- flagging a service appointment that lacks a required inspection;
- identifying inventory below a replenishment threshold;
- detecting incompatible units or conflicting product specifications.

Separate **asserted**, **inferred**, **predicted**, and **user-provided** facts.
Every derived or predicted fact should carry rule/model version, generation
time, confidence where applicable, and source references. Do not use OWL
inference as a substitute for authorization, financial settlement, safety
approval, or regulatory compliance.

### 14.7 Build AI-ready retrieval contracts

Expose stable, task-oriented tools instead of unrestricted graph access:

| AI task | Required grounding |
|---|---|
| Tire recommendation | vehicle configuration, fitment, inventory, price, constraints, timestamp |
| Store availability | store identity, inventory snapshot, reservation state, refresh time |
| Service scheduling | customer vehicle, operation, store capacity, appointment rules |
| Order explanation | order, line items, prices, promotions, fulfillment, source timestamps |
| Warranty answer | product identity, warranty terms, eligibility, claim status, provenance |
| Catalog search | canonical product, normalized specifications, aliases, fitment |

Each AI response should include answer scope, as-of time, source/provenance,
confidence or ambiguity, and a safe next action. Retrieval should enforce
tenant, role, geography, purpose, and field-level privacy filters before the
model receives data. Limit graph depth and result size, and refuse to infer
missing safety or eligibility facts.

### 14.8 Evaluate quality, safety, and AI behavior

Create test fixtures for:

- valid and invalid tire specifications;
- duplicate products and conflicting identifiers;
- multiple vehicle fitments and discontinued products;
- inventory snapshots, reservations, and late events;
- price and promotion effective dates;
- service, warranty, return, and recall scenarios;
- restricted customer and employee information.

Measure schema conformance, mapping completeness, duplicate rate, freshness,
reasoning correctness, retrieval precision, citation/provenance coverage,
refusal accuracy, privacy leakage, and answer consistency. Require human review
for safety-critical recommendations, customer-impacting decisions, and actions
that change orders, inventory, appointments, or payments.

### 14.9 Roll out incrementally

Use a staged delivery plan:

1. **Pilot:** catalog, vehicle fitment, store, and read-only inventory.
2. **Grounding:** mappings, provenance, SHACL quality gates, and AI search.
3. **Operations:** service appointments, installation, warranty, and returns.
4. **Commerce:** pricing, promotions, orders, and fulfillment with strict
   authorization boundaries.
5. **Scale:** event synchronization, reasoning materialization, monitoring, and
   governed cross-domain analytics.

Do not enable write actions merely because read retrieval is working. Promote
each module only after data owners approve its definitions, mappings, quality
metrics, privacy controls, and rollback procedure.
