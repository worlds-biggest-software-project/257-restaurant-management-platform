# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Restaurant Management Platform · Created: 2026-05-22

## Philosophy

This model combines a conventional relational backbone for transactional operations (orders, payments, inventory) with a property graph layer for relationship-heavy queries. The graph layer models the connections between entities that are expensive to query in pure relational systems: "which staff members are trained on which stations and have worked at which locations?", "which ingredients come from which suppliers and go into which menu items?", "which guests know each other and share dining preferences?", "which menu items are frequently ordered together?"

Restaurant operations are surprisingly graph-dense. A single dinner service involves a web of relationships: servers are assigned to tables, tables hold guests, guests have allergens, orders contain items, items require ingredients from suppliers, items route to kitchen stations, stations are staffed by cooks, cooks have certifications. Traversing these relationships in SQL requires multi-way joins that grow exponentially with depth. A property graph makes these traversals constant-time operations.

The graph is implemented using PostgreSQL-native `graph_node` and `graph_edge` tables with JSONB properties, not an external graph database. This keeps the operational complexity low (single database) while enabling graph query patterns via recursive CTEs or the Apache AGE extension for PostgreSQL (which adds Cypher query support).

**Best for:** Platforms prioritising AI-powered recommendation engines, supply chain traceability, staff skill management, and guest relationship intelligence -- where the value comes from traversing connections between entities.

**Trade-offs:**
- Pro: Guest-to-guest, item-to-item, and staff-to-station relationships enable AI recommendation engines
- Pro: Supply chain traceability ("which supplier provided the lettuce in the recalled batch?") is a simple graph traversal
- Pro: Staff skill graphs enable intelligent scheduling ("find servers trained on bar who speak Spanish and are available Friday")
- Pro: Menu item co-occurrence graphs drive upselling ("guests who ordered X also ordered Y")
- Pro: PostgreSQL-native -- no external graph database to manage
- Con: Graph abstraction adds a learning curve for developers unfamiliar with graph query patterns
- Con: Generic graph_node/graph_edge tables sacrifice some of the self-documenting nature of explicit relational tables
- Con: Dual-model (relational + graph) means some data exists in two places, requiring synchronisation
- Con: Graph queries via recursive CTEs are less performant than native graph databases (Neo4j) for very deep traversals
- Con: BI tools do not natively query graph structures -- reporting still comes from relational tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Schema.org Restaurant/Menu/MenuItem | Menu items in graph nodes include Schema.org-compatible properties for JSON-LD export |
| PCI DSS v4.0 | Payment data remains in relational tables; graph edges reference payment nodes by ID only |
| ISO 22000 / HACCP | Ingredient provenance graph supports "farm to fork" traceability required by food safety regulations |
| FDA Allergen Guidance (2025) | Allergen relationships modelled as graph edges (Guest -[HAS_ALLERGY]-> Allergen, MenuItem -[CONTAINS]-> Allergen) enabling transitive safety checks |
| ISO 3166-1/2 | Location nodes include ISO 3166 jurisdiction properties |
| ISO 4217 | Monetary properties on graph nodes use ISO 4217 currency codes |
| GDPR / CCPA | Guest nodes support property-level erasure; edges can be severed for right-to-erasure |
| CloudEvents 1.0 | Audit events follow CloudEvents envelope; graph mutations emit events |
| W3C RDF / Property Graph | Graph layer follows the Labeled Property Graph model compatible with Apache TinkerPop and AGE |

---

## Relational Core (Transactional Operations)

The relational layer handles the high-throughput, ACID-critical operations: order processing, payment capture, inventory transactions, and time tracking. These tables are optimised for write performance and transactional integrity.

```sql
-- ============================================================
-- ORGANISATION & LOCATION (relational)
-- ============================================================

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan_tier       TEXT NOT NULL DEFAULT 'standard',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    address_line1   TEXT,
    city            TEXT,
    state_province  TEXT,
    postal_code     TEXT,
    country_code    CHAR(2) NOT NULL DEFAULT 'US',
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    latitude        NUMERIC(9,6),
    longitude       NUMERIC(9,6),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    is_ghost_kitchen BOOLEAN NOT NULL DEFAULT false,
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, slug)
);

CREATE INDEX idx_location_org ON location(organisation_id);

-- ============================================================
-- ORDERS & PAYMENTS (relational — high-throughput ACID)
-- ============================================================

CREATE TABLE restaurant_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    order_number    TEXT NOT NULL,
    order_type      TEXT NOT NULL DEFAULT 'dine_in',
    channel         TEXT NOT NULL DEFAULT 'pos',
    status          TEXT NOT NULL DEFAULT 'open',
    server_id       UUID,
    table_id        UUID,
    guest_id        UUID,
    subtotal        NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    tax_total       NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    discount_total  NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    tip_total       NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    grand_total     NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    guest_count     INT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_order_location ON restaurant_order(location_id, created_at);
CREATE INDEX idx_order_status ON restaurant_order(location_id, status);

CREATE TABLE order_line_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES restaurant_order(id),
    menu_item_id    UUID,
    name            TEXT NOT NULL,
    kitchen_name    TEXT,
    quantity        INT NOT NULL DEFAULT 1,
    unit_price      NUMERIC(10,2) NOT NULL,
    line_total      NUMERIC(10,2) NOT NULL,
    course_number   INT DEFAULT 1,
    seat_number     INT,
    status          TEXT NOT NULL DEFAULT 'pending',
    modifiers       JSONB NOT NULL DEFAULT '[]',
    notes           TEXT,
    fired_at        TIMESTAMPTZ,
    ready_at        TIMESTAMPTZ,
    served_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_line_item_order ON order_line_item(order_id);

CREATE TABLE payment_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES restaurant_order(id),
    payment_method  TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    amount          NUMERIC(12,2) NOT NULL,
    tip_amount      NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    gateway         TEXT,
    gateway_txn_id  TEXT,
    card_brand      TEXT,
    card_last_four  CHAR(4),
    processed_by    UUID,
    processed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_order ON payment_transaction(order_id);

-- ============================================================
-- INVENTORY TRANSACTIONS (relational — ledger pattern)
-- ============================================================

CREATE TABLE inventory_stock (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    ingredient_id   UUID NOT NULL,                  -- references graph node
    quantity_on_hand NUMERIC(12,4) NOT NULL DEFAULT 0,
    unit_of_measure TEXT NOT NULL,
    last_counted_at TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(location_id, ingredient_id)
);

CREATE TABLE inventory_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    ingredient_id   UUID NOT NULL,
    transaction_type TEXT NOT NULL,
    quantity        NUMERIC(12,4) NOT NULL,
    unit_of_measure TEXT NOT NULL,
    context         JSONB NOT NULL DEFAULT '{}',
    performed_by    UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inv_txn_location ON inventory_transaction(location_id, created_at);

-- ============================================================
-- SCHEDULING & TIME TRACKING (relational)
-- ============================================================

CREATE TABLE shift (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    staff_member_id UUID NOT NULL,                   -- references graph node
    role_name       TEXT,
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    status          TEXT NOT NULL DEFAULT 'scheduled',
    is_published    BOOLEAN NOT NULL DEFAULT false,
    clock_in        TIMESTAMPTZ,
    clock_out       TIMESTAMPTZ,
    break_minutes   INT NOT NULL DEFAULT 0,
    total_minutes   INT,
    hourly_rate     NUMERIC(10,2),
    total_pay       NUMERIC(10,2),
    tips_earned     NUMERIC(10,2) DEFAULT 0.00,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shift_location ON shift(location_id, start_time);
CREATE INDEX idx_shift_staff ON shift(staff_member_id, start_time);

-- ============================================================
-- RESERVATIONS (relational)
-- ============================================================

CREATE TABLE reservation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    guest_id        UUID,                           -- references graph node
    table_id        UUID,                           -- references graph node
    party_size      INT NOT NULL,
    reserved_at     TIMESTAMPTZ NOT NULL,
    duration_mins   INT NOT NULL DEFAULT 90,
    status          TEXT NOT NULL DEFAULT 'confirmed',
    source          TEXT NOT NULL DEFAULT 'direct',
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reservation_location ON reservation(location_id, reserved_at);

-- ============================================================
-- FOOD SAFETY (relational — immutable logs)
-- ============================================================

CREATE TABLE food_safety_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    log_type        TEXT NOT NULL,
    completed_by    UUID NOT NULL,
    shift_date      DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'complete',
    data            JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_safety_log_location ON food_safety_log(location_id, shift_date);

-- ============================================================
-- DAILY SUMMARY (relational — reporting aggregate)
-- ============================================================

CREATE TABLE daily_summary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    business_date   DATE NOT NULL,
    gross_sales     NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    net_sales       NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    tax_collected   NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    tips_collected  NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    order_count     INT NOT NULL DEFAULT 0,
    guest_count     INT NOT NULL DEFAULT 0,
    breakdown       JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(location_id, business_date)
);

-- ============================================================
-- AUDIT LOG (relational — append-only)
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    location_id     UUID,
    actor_id        UUID,
    actor_type      TEXT NOT NULL DEFAULT 'staff',
    event_type      TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    changes         JSONB,
    metadata        JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organisation_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Property Graph Layer

The graph layer models entities and their relationships. Every entity in the domain is represented as a node; every relationship is an edge. Nodes and edges carry typed properties in JSONB.

```sql
-- ============================================================
-- PROPERTY GRAPH — generic node/edge tables
-- ============================================================

CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    label           TEXT NOT NULL,                  -- node type (see Node Labels below)
    properties      JSONB NOT NULL DEFAULT '{}',    -- typed properties per label
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_node_org_label ON graph_node(organisation_id, label);
CREATE INDEX idx_node_properties ON graph_node USING GIN (properties);
CREATE INDEX idx_node_label ON graph_node(label);

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    source_id       UUID NOT NULL REFERENCES graph_node(id),
    target_id       UUID NOT NULL REFERENCES graph_node(id),
    label           TEXT NOT NULL,                  -- relationship type (see Edge Labels below)
    properties      JSONB NOT NULL DEFAULT '{}',    -- relationship properties
    weight          NUMERIC(10,4),                  -- optional numeric weight for scoring/ranking
    valid_from      TIMESTAMPTZ DEFAULT now(),      -- temporal validity
    valid_until     TIMESTAMPTZ,                    -- NULL = currently valid
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_edge_source ON graph_edge(source_id, label);
CREATE INDEX idx_edge_target ON graph_edge(target_id, label);
CREATE INDEX idx_edge_org_label ON graph_edge(organisation_id, label);
CREATE INDEX idx_edge_properties ON graph_edge USING GIN (properties);
CREATE INDEX idx_edge_active ON graph_edge(source_id, label) WHERE valid_until IS NULL;

-- Prevent duplicate active edges of the same type between the same nodes
CREATE UNIQUE INDEX idx_edge_unique_active 
    ON graph_edge(source_id, target_id, label) 
    WHERE valid_until IS NULL;
```

---

## Node Labels & Properties

```sql
-- ============================================================
-- NODE LABEL DEFINITIONS (documentation & examples)
-- ============================================================

-- Label: StaffMember
-- Properties: {
--   "email": "alex@restaurant.com",
--   "phone": "+1555...",
--   "first_name": "Alex",
--   "last_name": "Rivera",
--   "display_name": "Alex R.",
--   "pin_hash": "bcrypt...",
--   "hourly_rate": 18.50,
--   "currency": "USD",
--   "employment_type": "hourly",
--   "hire_date": "2025-03-15",
--   "languages": ["english", "spanish"],
--   "certifications": ["food_handler", "alcohol_service", "first_aid"]
-- }

-- Label: MenuItem
-- Properties: {
--   "name": "Grilled Salmon",
--   "kitchen_name": "SALMON GRL",
--   "description": "Atlantic salmon, herb butter, seasonal vegetables",
--   "sku": "MAIN-042",
--   "base_price": 28.00,
--   "cost_price": 9.80,
--   "currency": "USD",
--   "calories": 520,
--   "prep_time_mins": 18,
--   "tax_category": "food",
--   "is_alcoholic": false,
--   "image_url": "https://...",
--   "nutrition": {"fat_g": 22, "protein_g": 42, "carbs_g": 15}
-- }

-- Label: Ingredient
-- Properties: {
--   "name": "Atlantic Salmon Fillet",
--   "category": "protein",
--   "unit_of_measure": "kg",
--   "cost_per_unit": 24.50,
--   "currency": "USD",
--   "shelf_life_days": 3,
--   "storage_temp_min_c": 0,
--   "storage_temp_max_c": 4,
--   "origin_country": "NO"
-- }

-- Label: Supplier
-- Properties: {
--   "name": "Pacific Fresh Seafood",
--   "contact_name": "Maria Chen",
--   "email": "orders@pacificfresh.com",
--   "phone": "+1555...",
--   "payment_terms": "net_30",
--   "delivery_days": ["monday", "wednesday", "friday"],
--   "minimum_order": 200.00,
--   "rating": 4.8
-- }

-- Label: Guest
-- Properties: {
--   "first_name": "John",
--   "last_name": "Smith",
--   "email": "john@example.com",
--   "phone": "+1555...",
--   "total_visits": 24,
--   "total_spend": 3420.00,
--   "avg_check": 142.50,
--   "is_vip": true,
--   "consent_status": "opted_in",
--   "notes": "Prefers booth seating. Anniversary March 15."
-- }

-- Label: Allergen
-- Properties: {
--   "code": "shellfish",
--   "name": "Crustacean Shellfish",
--   "is_fda_major": true
-- }

-- Label: DietaryTag
-- Properties: {
--   "code": "vegetarian",
--   "name": "Vegetarian",
--   "schema_org_diet": "https://schema.org/VegetarianDiet"
-- }

-- Label: KitchenStation
-- Properties: {
--   "name": "Grill",
--   "location_id": "uuid...",
--   "display_order": 1,
--   "max_concurrent_tickets": 8
-- }

-- Label: Table
-- Properties: {
--   "table_number": "A1",
--   "floor_plan": "Main Dining",
--   "location_id": "uuid...",
--   "capacity": 4,
--   "min_covers": 2,
--   "shape": "round",
--   "is_combinable": true,
--   "x_position": 120.0,
--   "y_position": 200.0
-- }

-- Label: ModifierGroup
-- Properties: {
--   "name": "Temperature",
--   "selection_type": "single",
--   "is_required": true,
--   "min_selections": 1,
--   "max_selections": 1
-- }

-- Label: Modifier
-- Properties: {
--   "name": "Medium Rare",
--   "price_adjustment": 0.00,
--   "is_default": true
-- }

-- Label: Menu
-- Properties: {
--   "name": "Dinner Menu",
--   "description": "Available 5 PM - 10 PM",
--   "is_active": true
-- }

-- Label: MenuSection
-- Properties: {
--   "name": "Appetizers",
--   "sort_order": 1
-- }

-- Label: LoyaltyProgram
-- Properties: {
--   "name": "Rewards Club",
--   "points_per_dollar": 1.0,
--   "redemption_threshold": 100,
--   "reward_value": 10.00
-- }
```

---

## Edge Labels & Properties

```sql
-- ============================================================
-- EDGE LABEL DEFINITIONS (documentation & examples)
-- ============================================================

-- ---- Staff relationships ----
-- StaffMember -[WORKS_AT]-> Location
--   properties: {"is_primary": true, "since": "2025-03-15"}

-- StaffMember -[HAS_ROLE]-> Role (Role is a graph node with label "Role")
--   properties: {"location_id": "uuid...", "granted_by": "uuid...", "granted_at": "..."}

-- StaffMember -[TRAINED_ON]-> KitchenStation
--   properties: {"proficiency": "expert", "certified_at": "2025-06-01"}

-- StaffMember -[MANAGES]-> StaffMember
--   properties: {"since": "2025-09-01"}

-- ---- Menu relationships ----
-- Menu -[CONTAINS_SECTION]-> MenuSection
--   properties: {"sort_order": 1}

-- MenuSection -[CONTAINS_ITEM]-> MenuItem
--   properties: {"sort_order": 1, "is_featured": true}

-- MenuItem -[HAS_MODIFIER_GROUP]-> ModifierGroup
--   properties: {"sort_order": 1}

-- ModifierGroup -[HAS_OPTION]-> Modifier
--   properties: {"sort_order": 1}

-- MenuItem -[CONTAINS_ALLERGEN]-> Allergen
--   properties: {"severity": "contains"}  -- contains, may_contain, trace

-- MenuItem -[TAGGED_WITH]-> DietaryTag
--   properties: {}

-- MenuItem -[REQUIRES_INGREDIENT]-> Ingredient
--   properties: {"quantity": 0.2, "unit": "kg", "waste_factor": 0.05}

-- MenuItem -[ROUTES_TO]-> KitchenStation
--   properties: {"priority": 1}

-- MenuItem -[PAIRS_WITH]-> MenuItem
--   properties: {"confidence": 0.85, "co_occurrence_count": 142}
--   (AI-generated: items frequently ordered together)

-- MenuItem -[AVAILABLE_AT]-> Location
--   properties: {"price_override": 26.00, "is_available": true}

-- Menu -[ASSIGNED_TO]-> Location
--   properties: {"schedule": {"monday": [{"start": "17:00", "end": "22:00"}]}}

-- ---- Inventory & supply chain ----
-- Supplier -[SUPPLIES]-> Ingredient
--   properties: {"sku": "SUP-SAL-001", "unit_cost": 24.50, "min_order_qty": 5, "lead_time_days": 1}

-- Ingredient -[STORED_AT]-> Location
--   properties: {"par_level": 50, "storage_zone": "walk_in_cooler"}

-- Ingredient -[SOURCED_FROM]-> Supplier (reverse of SUPPLIES, for traversal convenience)
--   properties: {"is_preferred": true}

-- ---- Guest relationships ----
-- Guest -[HAS_ALLERGY]-> Allergen
--   properties: {"severity": "severe", "noted_at": "2025-06-10"}

-- Guest -[PREFERS]-> MenuItem
--   properties: {"order_count": 8, "last_ordered": "2026-05-15", "avg_rating": 4.5}
--   (AI-generated from order history)

-- Guest -[DINES_WITH]-> Guest
--   properties: {"co_visit_count": 12, "relationship": "spouse"}
--   (AI-generated from shared table sessions)

-- Guest -[ENROLLED_IN]-> LoyaltyProgram
--   properties: {"points_balance": 2400, "lifetime_points": 8200, "tier": "gold", "enrolled_at": "2025-06-10"}

-- Guest -[VISITED]-> Location
--   properties: {"visit_count": 18, "total_spend": 2160.00, "last_visit": "2026-05-15"}

-- Guest -[PREFERS_SEATING]-> Table
--   properties: {"request_count": 5, "preference_strength": "strong"}
--   (AI-generated from reservation patterns)

-- ---- Operational relationships ----
-- Table -[ON_FLOOR]-> FloorPlan (FloorPlan is a graph node)
--   properties: {}

-- Table -[ADJACENT_TO]-> Table
--   properties: {"can_combine": true}
--   (enables intelligent table combination for large parties)
```

---

## Graph Query Examples

### Allergen Safety Check

```sql
-- "Does any item in this order contain allergens that this guest is allergic to?"
-- This is a critical safety query that runs on every order for a known guest.

WITH guest_allergens AS (
    SELECT e.target_id AS allergen_id
    FROM graph_edge e
    WHERE e.source_id = '<<guest-uuid>>'
      AND e.label = 'HAS_ALLERGY'
      AND e.valid_until IS NULL
),
order_items AS (
    SELECT oli.menu_item_id
    FROM order_line_item oli
    WHERE oli.order_id = '<<order-uuid>>'
),
item_allergens AS (
    SELECT e.source_id AS menu_item_id, e.target_id AS allergen_id,
           n.properties->>'name' AS allergen_name,
           item_node.properties->>'name' AS item_name,
           e.properties->>'severity' AS severity
    FROM graph_edge e
    JOIN graph_node n ON n.id = e.target_id
    JOIN graph_node item_node ON item_node.id = e.source_id
    WHERE e.label = 'CONTAINS_ALLERGEN'
      AND e.source_id IN (SELECT menu_item_id FROM order_items)
)
SELECT ia.item_name, ia.allergen_name, ia.severity
FROM item_allergens ia
JOIN guest_allergens ga ON ga.allergen_id = ia.allergen_id
ORDER BY ia.severity DESC;
-- Result: "Grilled Salmon" contains "Crustacean Shellfish" (severity: may_contain)
-- WARNING displayed on POS before order is confirmed
```

### Supply Chain Traceability

```sql
-- "Which supplier provided the lettuce used in menu items ordered on May 20?"
-- Critical for food safety recall response.

WITH affected_items AS (
    SELECT DISTINCT oli.menu_item_id
    FROM order_line_item oli
    JOIN restaurant_order o ON o.id = oli.order_id
    WHERE o.created_at::DATE = '2026-05-20'
),
item_ingredients AS (
    SELECT e.source_id AS menu_item_id, e.target_id AS ingredient_id
    FROM graph_edge e
    WHERE e.label = 'REQUIRES_INGREDIENT'
      AND e.source_id IN (SELECT menu_item_id FROM affected_items)
),
lettuce_ingredients AS (
    SELECT ii.ingredient_id, ii.menu_item_id
    FROM item_ingredients ii
    JOIN graph_node n ON n.id = ii.ingredient_id
    WHERE n.properties->>'name' ILIKE '%lettuce%'
)
SELECT 
    supplier_node.properties->>'name' AS supplier_name,
    supplier_node.properties->>'contact_name' AS contact,
    supplier_node.properties->>'phone' AS phone,
    supply_edge.properties->>'sku' AS supplier_sku,
    ingredient_node.properties->>'name' AS ingredient,
    item_node.properties->>'name' AS used_in_item
FROM lettuce_ingredients li
JOIN graph_edge supply_edge ON supply_edge.target_id = li.ingredient_id AND supply_edge.label = 'SUPPLIES'
JOIN graph_node supplier_node ON supplier_node.id = supply_edge.source_id
JOIN graph_node ingredient_node ON ingredient_node.id = li.ingredient_id
JOIN graph_node item_node ON item_node.id = li.menu_item_id;
```

### Intelligent Staff Scheduling

```sql
-- "Find staff members trained on grill station who speak Spanish,
--  work at location X, and are available (no shift) on Friday evening"

WITH grill_trained AS (
    SELECT e.source_id AS staff_id
    FROM graph_edge e
    JOIN graph_node station ON station.id = e.target_id
    WHERE e.label = 'TRAINED_ON'
      AND station.properties->>'name' = 'Grill'
      AND e.valid_until IS NULL
),
spanish_speakers AS (
    SELECT n.id AS staff_id
    FROM graph_node n
    WHERE n.label = 'StaffMember'
      AND n.properties->'languages' ? 'spanish'
      AND n.is_active = true
),
location_staff AS (
    SELECT e.source_id AS staff_id
    FROM graph_edge e
    WHERE e.label = 'WORKS_AT'
      AND e.target_id = '<<location-uuid>>'
      AND e.valid_until IS NULL
),
friday_busy AS (
    SELECT staff_member_id
    FROM shift
    WHERE location_id = '<<location-uuid>>'
      AND start_time >= '2026-05-22 17:00:00-05'
      AND end_time <= '2026-05-22 23:00:00-05'
)
SELECT n.properties->>'first_name' AS first_name,
       n.properties->>'last_name' AS last_name,
       n.properties->>'hourly_rate' AS rate
FROM graph_node n
WHERE n.id IN (SELECT staff_id FROM grill_trained)
  AND n.id IN (SELECT staff_id FROM spanish_speakers)
  AND n.id IN (SELECT staff_id FROM location_staff)
  AND n.id NOT IN (SELECT staff_member_id FROM friday_busy);
```

### AI-Powered Menu Recommendations

```sql
-- "What items are most frequently ordered with Grilled Salmon?"
-- Used for upselling suggestions on the POS and online ordering.

SELECT 
    target_node.properties->>'name' AS recommended_item,
    target_node.properties->>'base_price' AS price,
    e.properties->>'confidence' AS confidence,
    (e.properties->>'co_occurrence_count')::INT AS times_ordered_together
FROM graph_edge e
JOIN graph_node target_node ON target_node.id = e.target_id
WHERE e.source_id = '<<salmon-menu-item-uuid>>'
  AND e.label = 'PAIRS_WITH'
  AND e.valid_until IS NULL
  AND target_node.is_active = true
ORDER BY (e.properties->>'confidence')::NUMERIC DESC
LIMIT 5;
-- Result:
-- "Caesar Salad"     | $14.00 | 0.85 | 142
-- "Pinot Grigio"     | $12.00 | 0.72 | 108
-- "Crème Brûlée"     | $11.00 | 0.65 | 94
-- "Sparkling Water"  | $4.00  | 0.58 | 87
-- "Mushroom Risotto" | $22.00 | 0.45 | 62
```

### Guest Relationship Intelligence

```sql
-- "Find guests who dine with John Smith and their shared preferences"
-- Used for VIP service and group reservation personalization.

WITH john AS (
    SELECT id FROM graph_node
    WHERE label = 'Guest'
      AND properties->>'email' = 'john@example.com'
      AND organisation_id = '<<org-uuid>>'
),
dining_companions AS (
    SELECT 
        CASE WHEN e.source_id = (SELECT id FROM john) THEN e.target_id
             ELSE e.source_id END AS companion_id,
        e.properties->>'co_visit_count' AS visits_together,
        e.properties->>'relationship' AS relationship
    FROM graph_edge e
    WHERE e.label = 'DINES_WITH'
      AND ((e.source_id = (SELECT id FROM john)) OR (e.target_id = (SELECT id FROM john)))
      AND e.valid_until IS NULL
)
SELECT 
    companion.properties->>'first_name' AS name,
    dc.relationship,
    dc.visits_together,
    -- Find shared menu item preferences
    (SELECT jsonb_agg(target.properties->>'name')
     FROM graph_edge pref
     JOIN graph_node target ON target.id = pref.target_id
     WHERE pref.source_id = dc.companion_id
       AND pref.label = 'PREFERS'
       AND (pref.properties->>'order_count')::INT >= 3
    ) AS favourite_items,
    -- Find companion allergens
    (SELECT jsonb_agg(allergen.properties->>'name')
     FROM graph_edge allergy
     JOIN graph_node allergen ON allergen.id = allergy.target_id
     WHERE allergy.source_id = dc.companion_id
       AND allergy.label = 'HAS_ALLERGY'
    ) AS allergens
FROM dining_companions dc
JOIN graph_node companion ON companion.id = dc.companion_id
ORDER BY dc.visits_together::INT DESC;
```

---

## Graph Maintenance & AI Pipeline

```sql
-- ============================================================
-- GRAPH SYNC — keeps graph in sync with relational writes
-- ============================================================

-- Option 1: PostgreSQL triggers on relational tables
-- When an order is closed, update guest PREFERS edges and PAIRS_WITH edges.

-- Option 2: Background worker processes audit_log events
-- A worker reads new audit_log entries and updates graph edges accordingly.

-- Option 3: Apache AGE extension for native Cypher queries
-- CREATE EXTENSION age;
-- SET search_path = ag_catalog, "$user", public;
-- SELECT create_graph('restaurant');
-- 
-- SELECT * FROM cypher('restaurant', $$
--   MATCH (g:Guest)-[:HAS_ALLERGY]->(a:Allergen)<-[:CONTAINS_ALLERGEN]-(m:MenuItem)
--   WHERE g.email = 'john@example.com'
--   RETURN m.name, a.name
-- $$) AS (menu_item TEXT, allergen TEXT);

-- ============================================================
-- AI PIPELINE TABLES — batch-generated graph edges
-- ============================================================

CREATE TABLE ai_graph_job (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_type        TEXT NOT NULL,                  -- 'co_occurrence', 'guest_clustering', 'preference_scoring'
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    status          TEXT NOT NULL DEFAULT 'pending', -- pending, running, completed, failed
    parameters      JSONB NOT NULL DEFAULT '{}',
    results_summary JSONB,
    edges_created   INT,
    edges_updated   INT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- AI jobs create/update graph edges:
-- 1. Co-occurrence analysis: scans order_line_item to build MenuItem -[PAIRS_WITH]-> MenuItem edges
-- 2. Guest clustering: identifies Guest -[DINES_WITH]-> Guest edges from shared table sessions
-- 3. Preference scoring: computes Guest -[PREFERS]-> MenuItem edges from order history
-- 4. Seating preference: computes Guest -[PREFERS_SEATING]-> Table edges from reservation history

-- Example: Co-occurrence batch job SQL
-- INSERT INTO graph_edge (organisation_id, source_id, target_id, label, properties, weight)
-- SELECT 
--     o.organisation_id,
--     a.menu_item_id,
--     b.menu_item_id,
--     'PAIRS_WITH',
--     jsonb_build_object(
--         'co_occurrence_count', COUNT(*),
--         'confidence', COUNT(*)::NUMERIC / 
--             (SELECT COUNT(DISTINCT order_id) FROM order_line_item WHERE menu_item_id = a.menu_item_id)
--     ),
--     COUNT(*)::NUMERIC / 
--         (SELECT COUNT(DISTINCT order_id) FROM order_line_item WHERE menu_item_id = a.menu_item_id)
-- FROM order_line_item a
-- JOIN order_line_item b ON a.order_id = b.order_id AND a.menu_item_id < b.menu_item_id
-- JOIN restaurant_order o ON o.id = a.order_id
-- WHERE o.created_at >= now() - INTERVAL '90 days'
-- GROUP BY o.organisation_id, a.menu_item_id, b.menu_item_id
-- HAVING COUNT(*) >= 5
-- ON CONFLICT (source_id, target_id, label) WHERE valid_until IS NULL
-- DO UPDATE SET properties = EXCLUDED.properties, weight = EXCLUDED.weight;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Location | 2 | Relational core |
| Orders & Payments | 3 | Relational — high-throughput ACID |
| Inventory | 2 | Relational ledger + stock levels |
| Scheduling | 1 | Relational time tracking |
| Reservations | 1 | Relational with graph FK to guest/table nodes |
| Food Safety | 1 | Relational immutable logs |
| Reporting | 1 | Daily summary aggregate |
| Audit | 1 | Append-only audit log |
| **Graph Layer** | **2** | graph_node + graph_edge (replaces ~20 entity tables) |
| AI Pipeline | 1 | Graph edge generation jobs |
| **Total** | **15** | Plus ~14 node labels and ~20 edge labels in graph |

---

## Key Design Decisions

1. **Graph for entities, relational for transactions** — the clear separation principle is: if the data is high-frequency, ACID-critical, and append-heavy (orders, payments, inventory transactions, time entries), it goes in relational tables. If the data is about entity identity, properties, and relationships (staff, menu items, ingredients, guests, suppliers), it lives in the graph.

2. **PostgreSQL-native graph, no external database** — using `graph_node` and `graph_edge` tables keeps the deployment to a single PostgreSQL instance. For teams that outgrow recursive CTEs, the Apache AGE extension adds Cypher query support without changing infrastructure.

3. **Temporal edges with valid_from/valid_until** — every graph edge has temporal validity. When a staff member changes location, the old WORKS_AT edge gets a `valid_until` timestamp and a new edge is created. This enables historical queries ("who was trained on grill in March?") without losing data.

4. **AI-generated edges are first-class graph citizens** — PAIRS_WITH, DINES_WITH, PREFERS, and PREFERS_SEATING edges are created by batch AI jobs and stored alongside manually-created edges. The `weight` column enables ranked retrieval (top 5 paired items, most frequent dining companions).

5. **Allergen safety as a graph traversal** — the query "does this order contain allergens dangerous to this guest?" is a two-hop graph traversal: Guest -> HAS_ALLERGY -> Allergen <- CONTAINS_ALLERGEN <- MenuItem. This is more natural and extensible than relational junction table queries, especially when adding transitive allergen relationships (e.g., "tree nuts" includes almonds, walnuts, etc.).

6. **Supply chain traceability via graph paths** — tracing an ingredient from supplier through recipes to orders served is a graph path query. During a food safety recall, this traversal identifies every affected order, guest, and supplier in seconds.

7. **Table adjacency graph enables intelligent seating** — Table -> ADJACENT_TO -> Table edges model physical proximity, enabling the system to automatically suggest table combinations for large parties or identify available adjacent tables.

8. **Unique active edge constraint prevents duplicates** — the partial unique index on `(source_id, target_id, label) WHERE valid_until IS NULL` ensures only one active edge of each type exists between any two nodes, while allowing historical edges to accumulate.

9. **GIN indexes on node and edge properties** — PostgreSQL GIN indexes support containment queries on JSONB properties, enabling queries like "find all staff who speak Spanish" without a separate languages table.

10. **Dual-model synchronisation via audit_log** — when a relational write occurs (e.g., new order closes), the audit_log entry triggers a background worker to update relevant graph edges (guest visit count, item preference scores, co-occurrence data). This eventual consistency model keeps the graph fresh without adding latency to the transactional path.
