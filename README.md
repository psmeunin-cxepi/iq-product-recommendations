# Cisco Product Recommendations Knowledge Graph

## Overview

This project implements a Neo4j-based knowledge graph for Cisco product lifecycle management and AI-driven product recommendations. The system supports replacement recommendations, specification comparisons, and licensing impact analysis.

## Scope and Objectives

### Primary Goals

Enable an AI assistant to answer lifecycle and product questions including:

- **Lifecycle status**: End-of-Sale (EoS) and End-of-Support dates
- **Product replacement**: "What should I replace this with?"
- **Specification comparison**: Side-by-side specs including power consumption
- **Licensing impact**: Required entitlements and compatibility

### Design Decisions

- **No hardcoded Recommendation nodes**: The AI infers recommendations dynamically
- **Two-phase recommendation process**:
  1. Generate candidate set from the graph
  2. Rank candidates using specifications, lifecycle, and licensing semantics
- **Lifecycle as properties**: Start with simple property-based lifecycle modeling
- **Successor hints**: Maintain relationship hints to guide candidate search without dictating single replacements

## Schema Overview

### Node Types

#### Product

A sellable Cisco product/model/SKU that customers own and inquire about.

**Key Properties:**
- `product_id` (string): Unique, stable identifier
- `sku` (string): Stock keeping unit identifier
- `model` (string): Product model number
- `name` (string): Product name
- `vendor` (string): Vendor name (default: "Cisco")
- `category` (string): Product category (e.g., switch, router, firewall)
- `introduced_date` (date): Product introduction date

#### ProductFamily

A portfolio grouping for products sharing positioning/architecture (e.g., "Catalyst 9000").

**Key Properties:**
- `family_id` (string): Unique identifier
- `name` (string): Family name
- `description` (string): Family description
- `segment` (string): Market segment

#### ProductSpecifications

A structured profile of technical characteristics for a product, containing normalized facts with units for comparison and filtering.

**Key Properties:**
- `spec_profile_id` (string): Unique identifier
- `profile_name` (string): Profile name
- `notes` (string): Additional notes

#### SpecFact

One atomic, queryable specification with normalized key and unit.

**Key Properties:**
- `key` (string): Normalized identifier (e.g., power_w, throughput_gbps, port_count)
- `value_num` (float): Numeric value
- `value_text` (string): Text value for non-numeric specifications
- `unit` (string): Unit of measurement (e.g., W, Gbps, count)

**Note:** SpecFact is the backbone of comparisons and inference. Consistent key naming and units are essential.

#### ProductLicensing

Licensing requirements for a product (what it needs to operate/unlock features). Focuses on requirements, not purchase offers.

**Key Properties:**
- `licensing_id` (string): Unique identifier
- `notes` (string): Additional notes
- `default_model` (string): Default licensing model (e.g., subscription, perpetual)

#### LicenseEntitlement

A capability/tier entitlement required by a product (e.g., "Essentials", "Advantage", "DNA", feature packs).

**Key Properties:**
- `entitlement_id` (string): Unique identifier
- `name` (string): Entitlement name
- `tier` (string): Entitlement tier
- `term_type` (string): Term type

### Lifecycle Properties

Lifecycle information is modeled as properties on the Product node.

**Recommended Lifecycle Properties:**
- `lifecycle_status` (string): Current status (e.g., ACTIVE, END_OF_SALE, END_OF_SUPPORT)
- `end_of_sale_date` (date): End of sale date
- `end_of_support_date` (date): End of support date
- `announcement_date` (date): Announcement date
- `last_updated` (datetime): Last update timestamp
- `lifecycle_notes` (string): Additional lifecycle notes

## Relationships and Semantics

### Product Grouping

```cypher
(:Product)-[:BELONGS_TO_FAMILY]->(:ProductFamily)
```

**Semantics:**
- Many products can belong to one family
- Used to narrow candidate sets for replacements/alternatives

### Product Specifications

```cypher
(:Product)-[:HAS_SPECS]->(:ProductSpecifications)
(:ProductSpecifications)-[:HAS_FACT]->(:SpecFact)
```

**Semantics:**
- ProductSpecifications acts as the spec container/profile
- SpecFact provides atomic, comparable values
- Keys must be normalized for stable querying and ranking

### Product Licensing Requirements

```cypher
(:Product)-[:HAS_LICENSING]->(:ProductLicensing)
(:ProductLicensing)-[:REQUIRES_ENTITLEMENT]->(:LicenseEntitlement)
```

**Semantics:**
- Enables answering: "Will licensing change if I move to product X?"
- Supports compatibility scoring during recommendation inference:
  - Same entitlements required → strong compatibility signal
  - Additional entitlements required → highlight impact/risks

### Successor Hint

```cypher
(:Product)-[:SUCCESSOR_HINT {reason, weight}]->(:Product)
```

**Properties:**
- `reason` (string): Reason for succession (e.g., "next generation in same line", "portfolio transition")
- `weight` (float): Weight value 0-1

**Semantics:**
- Not "the replacement" but a curation hint to guide AI candidate generation
- AI still ranks candidates using lifecycle/spec/licensing constraints
- Can output multiple options

**Alternative (family-level):**
```cypher
(:ProductFamily)-[:EVOLVES_TO]->(:ProductFamily)
```

## Recommendation Inference Logic

When users request replacements or alternatives, the AI follows this process:

### Step 1: Identify Source Product

Match by SKU/model/name and return:
- Lifecycle fields (status, end-of-sale date, end-of-support date)
- Key specifications (from SpecFact)
- Required entitlements

### Step 2: Generate Candidate Set

Candidates come from:
- SUCCESSOR_HINT targets (highest priority)
- Products in the same ProductFamily (or same category) where lifecycle indicates viability (e.g., status = ACTIVE, or future end-of-support date)

### Step 3: Rank Candidates

Rank according to user constraints and defaults:
- Prefer equal/higher performance (e.g., throughput, port count)
- Prefer lower power consumption if requested
- Prefer licensing compatibility (same entitlements)
- Penalize missing critical specs

### Step 4: Respond with Top N + Reasoning

Return top 3 candidates with:
- Key specification comparisons
- Lifecycle status of each candidate
- Licensing impact summary
- Confidence notes and missing data indicators

## Supported Query Examples

### Lifecycle and Support

- "What is the lifecycle status of WS-C4503?"
- "When is end of sale and end of support for C9300-48UXM-A?"
- "Is product X still supported in 2026?"

### Replacement/Alternatives

- "My switch X is EoL. What should I replace it with?"
- "Give me 3 replacement options for C9300-48UXM-A with similar port count."
- "Suggest an alternative with lower power consumption but comparable throughput."

### Specification Comparison

- "Compare power consumption between Product A and Product B."
- "Does the replacement have 48 ports and PoE?"
- "Which option has the best throughput per watt?"

### Licensing Impact

- "If I replace X with Y, will my current license still work?"
- "What entitlements does Product Y require?"
- "Show me replacements that require the same licensing tier as my current product."

### Constraint-Driven Selection

- "Find a replacement that is Active, has >= 48 ports, and power <= 250W."
- "Recommend a successor that matches my specs as closely as possible."

## Naming Conventions

### SpecFact Key Examples

Use consistent, normalized keys:

- `power_w` - Power consumption in watts
- `throughput_gbps` - Throughput in gigabits per second
- `port_count` - Number of ports
- `poe_supported` - PoE support (true/false or 0/1)
- `poe_budget_w` - PoE budget in watts
- `rack_units` - Rack unit size
- `uplink_speed_gbps` - Uplink speed in gigabits per second

**Important:** Keep these keys stable as AI and Cypher templates depend on them.

## Next Steps

### Immediate Enhancements

- Cypher "candidate generation" query templates that return consistent result shapes for agent ranking
- Compact LLM ontology context blocks (node/relationship descriptions) for injection into Cypher-planner/reasoning agents

### Future: Integration with Topology & Telemetry

**Value Proposition:** Enhance recommendations by incorporating real-world operational data alongside static product specifications.

**Key Benefits:**
- **Operational Context**: Recommendations consider actual device usage patterns (power consumption, utilization) rather than just manufacturer specifications
- **Site-Aware Recommendations**: Factor in location-specific constraints and performance requirements
- **Capacity Planning**: Identify replacement opportunities based on observed headroom and growth patterns
- **Cost Optimization**: Recommend right-sized replacements that avoid over-provisioning while meeting actual performance needs

**Integration Approach:** Maintain separate domain graphs (product catalog vs. operational data) connected through a lightweight semantic hub layer that maps devices to products and provides aggregated telemetry metrics for enhanced decision-making. Operational data is derived from Marvin.

### Future: RAG Integration with Cisco Documentation

**Value Proposition:** Enhance recommendation quality by combining structured knowledge graph data with authoritative Cisco documentation for comprehensive, citation-backed guidance.

**Key Benefits:**
- **Enhanced Rationale**: Provide doc-backed reasoning beyond structured data (migration paths, compatibility notes, thermal considerations)
- **Licensing Nuances**: Access detailed licensing guidance often missing from structured data
- **Migration Insights**: Surface documented caveats, prerequisites, and best practices from official sources
- **Authoritative Citations**: Reference specific datasheets, ordering guides, and lifecycle bulletins

**Architecture Pattern:** KG-first approach where the knowledge graph handles entity resolution and candidate generation, then RAG retrieves product-specific documentation to enrich recommendations with authoritative context and detailed guidance.

**Key Mitigations:**
- **Precision Retrieval**: Filter by product_id/family_id + document type to avoid cross-contamination from similar SKUs
- **Source Authority**: Establish clear precedence rules (e.g., EoL bulletins override, prefer latest publication dates)
- **Confidence Transparency**: Distinguish between "official" replacements (explicitly documented) and "inferred" recommendations with clear confidence indicators