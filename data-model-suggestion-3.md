# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Restaurant Management Platform · Created: 2026-05-22

## Philosophy

This model uses normalised relational tables for the stable core of the domain (organisations, locations, staff, orders, payments) while leveraging PostgreSQL JSONB columns for everything that varies by location, jurisdiction, cuisine type, or deployment context. The key insight is that a restaurant in Tokyo, a ghost kitchen in Los Angeles, and a stadium food court in London share the same order-processing pipeline but have wildly different menus, tax rules, tipping conventions, food safety checklists, and regulatory requirements.

Rather than modelling every possible variation as a nullable column or a junction table (which leads to schema sprawl), the hybrid approach stores variable data in typed JSONB columns with JSON Schema validation at the application layer. This is the approach used internally by platforms like Square, whose Catalog API stores restaurant-specific fields (kitchen_name, buyer_facing_name) alongside generic product attributes, and by Deliverect, whose menu sync middleware must accommodate hundreds of different restaurant configurations through a single schema.

This approach is best suited for teams building an MVP rapidly, supporting multi-region deployment where jurisdiction-specific fields vary significantly, or operating ghost kitchen / multi-brand scenarios where menu structures differ across virtual brands sharing the same physical kitchen.

**Best for:** Rapid MVP development, multi-region deployments with jurisdiction-specific requirements, and ghost kitchen platforms managing diverse virtual brands from a single schema.

**Trade-offs:**
- Pro: Far fewer tables than full normalisation (30-35 vs. 50+), faster to develop and migrate
- Pro: New fields can be added to JSONB without schema migrations — just deploy code
- Pro: Multi-region variation (UK VAT vs. US sales tax vs. Japan consumption tax) handled without schema branches
- Pro: Ghost kitchen multi-brand menus naturally fit in JSONB (each brand has its own menu structure)
- Pro: PostgreSQL JSONB indexing (GIN) provides excellent query performance on structured JSON
- Con: JSONB fields lack database-level referential integrity — validation moves to application layer
- Con: Complex JSONB queries are harder to write and debug than simple JOINs
- Con: Schema drift risk if JSONB structures are not governed by JSON Schema validation
- Con: Reporting tools (Metabase, Looker) work less naturally with JSONB than flat columns
- Con: Allergen and dietary queries across JSONB arrays are slower than junction table lookups

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Schema.org Restaurant/Menu/MenuItem | Menu items stored with Schema.org-compatible JSONB structure for direct JSON-LD export |
| PCI DSS v4.0 | Payment tokens stored in relational columns; JSONB never contains raw card data |
| ISO 22000 / HACCP | Food safety checklists stored as JSONB templates with schema-validated response structures |
| FDA Allergen Guidance (2025) | Allergen arrays in menu item JSONB use standardised codes from FDA's 9 major allergens |
| ISO 3166-1/2 | Jurisdiction configuration in JSONB uses ISO 3166 codes as keys |
| ISO 4217 | Currency codes in monetary JSONB fields follow ISO 4217 |
| JSON Schema (2020-12) | Every JSONB column has a corresponding JSON Schema definition for application-layer validation |
| CloudEvents 1.0 | Webhook payloads follow CloudEvents envelope with JSONB data field |
| GDPR / CCPA | Guest preferences JSONB includes consent tracking; supports selective field erasure |

---

## Organisation & Location Layer

```sql
-- ============================================================
-- ORGANISATION & LOCATION — relational core with JSONB settings
-- ============================================================

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    billing_email   TEXT,
    plan_tier       TEXT NOT NULL DEFAULT 'standard',
    -- JSONB: org-wide configuration that varies per deployment
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "default_currency": "USD",
    --   "default_timezone": "America/New_York",
    --   "tipping": {"enabled": true, "suggested_percentages": [18, 20, 25]},
    --   "loyalty": {"enabled": true, "points_per_dollar": 1.0},
    --   "branding": {"logo_url": "...", "primary_color": "#FF5722"},
    --   "integrations": {"accounting": "xero", "payroll": "adp"},
    --   "feature_flags": {"ai_menu_engineering": true, "ghost_kitchen": true}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    -- Core address fields (relational for geocoding/search)
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    state_province  TEXT,
    postal_code     TEXT,
    country_code    CHAR(2) NOT NULL DEFAULT 'US',
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    latitude        NUMERIC(9,6),
    longitude       NUMERIC(9,6),
    phone           TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    is_ghost_kitchen BOOLEAN NOT NULL DEFAULT false,
    -- JSONB: location-specific overrides and configuration
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "operating_hours": {
    --     "monday": {"open": "11:00", "close": "22:00"},
    --     "tuesday": {"open": "11:00", "close": "22:00"},
    --     ...
    --   },
    --   "tax_rules": [
    --     {"name": "State Sales Tax", "rate": 0.0625, "applies_to": "all", "jurisdiction": "US-TX"},
    --     {"name": "Alcohol Tax", "rate": 0.0825, "applies_to": "alcohol", "jurisdiction": "US-TX"}
    --   ],
    --   "service_charge": {"auto_gratuity_threshold": 6, "rate": 0.20},
    --   "tipping_override": {"suggested_percentages": [15, 18, 22]},
    --   "delivery_channels": ["first_party", "uber_eats", "doordash"],
    --   "kds_stations": ["grill", "saute", "cold", "dessert", "expo"],
    --   "fiscal_compliance": {
    --     "type": "us_standard",  -- or "fr_nf525", "de_kassensichv", "it_rt"
    --     "fiscal_printer_id": null
    --   },
    --   "food_safety": {
    --     "temperature_check_interval_hours": 4,
    --     "equipment": ["Walk-in Cooler", "Freezer #1", "Hot Hold"]
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, slug)
);

CREATE INDEX idx_location_org ON location(organisation_id);
CREATE INDEX idx_location_config ON location USING GIN (config);

-- Row-Level Security
ALTER TABLE location ENABLE ROW LEVEL SECURITY;
CREATE POLICY location_tenant ON location
    USING (organisation_id = current_setting('app.current_org_id')::UUID);
```

---

## Staff & Access Control

```sql
-- ============================================================
-- STAFF — relational core, JSONB for variable employment details
-- ============================================================

CREATE TABLE staff_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           TEXT,
    phone           TEXT,
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    display_name    TEXT,
    pin_hash        TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: employment details that vary by jurisdiction
    employment      JSONB NOT NULL DEFAULT '{}',
    -- Example employment:
    -- {
    --   "type": "hourly",
    --   "hourly_rate": 18.50,
    --   "currency": "USD",
    --   "hire_date": "2025-03-15",
    --   "roles": ["server", "bartender"],
    --   "certifications": ["food_handler", "alcohol_service"],
    --   "locations": ["uuid-loc-1", "uuid-loc-2"],
    --   "primary_location": "uuid-loc-1",
    --   "tax_withholding": {"federal": "W4", "state": "TX"},
    --   "emergency_contact": {"name": "Jane Doe", "phone": "+1555..."}
    -- }
    -- JSONB: permissions and access control
    permissions     JSONB NOT NULL DEFAULT '{}',
    -- Example permissions:
    -- {
    --   "role": "manager",
    --   "scopes": ["pos.*", "inventory.view", "inventory.adjust", "staff.schedule", "reports.*"],
    --   "location_access": ["uuid-loc-1"],  -- empty array = all locations
    --   "pos_limits": {"max_discount_pct": 50, "can_void": true, "can_comp": true}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_staff_org ON staff_member(organisation_id);
CREATE INDEX idx_staff_active ON staff_member(organisation_id, is_active);
CREATE INDEX idx_staff_permissions ON staff_member USING GIN (permissions);
```

---

## Menu Catalogue

```sql
-- ============================================================
-- MENU CATALOGUE — hybrid: relational structure, JSONB details
-- ============================================================

CREATE TABLE menu (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    sort_order      INT NOT NULL DEFAULT 0,
    -- JSONB: which locations and time windows this menu applies to
    availability    JSONB NOT NULL DEFAULT '{}',
    -- Example availability:
    -- {
    --   "locations": ["uuid-loc-1", "uuid-loc-2"],  -- empty = all
    --   "channels": ["dine_in", "takeout", "online"],
    --   "schedule": {
    --     "monday": [{"start": "11:00", "end": "15:00"}],
    --     "friday": [{"start": "11:00", "end": "15:00"}, {"start": "17:00", "end": "22:00"}],
    --     "saturday": [{"start": "10:00", "end": "15:00"}]
    --   },
    --   "seasonal": {"start_date": "2026-06-01", "end_date": "2026-08-31"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_menu_org ON menu(organisation_id);

CREATE TABLE menu_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    menu_id         UUID REFERENCES menu(id),
    section_name    TEXT,                           -- "Appetizers", "Mains" — denormalised for simplicity
    name            TEXT NOT NULL,
    kitchen_name    TEXT,
    description     TEXT,
    sku             TEXT,
    base_price      NUMERIC(10,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    sort_order      INT NOT NULL DEFAULT 0,
    image_url       TEXT,
    -- JSONB: all variable menu item properties
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "calories": 520,
    --   "prep_time_mins": 18,
    --   "cost_price": 7.85,
    --   "margin_pct": 0.72,
    --   "tax_category": "food",
    --   "is_alcoholic": false,
    --   "allergens": ["milk", "wheat", "eggs"],
    --   "dietary_tags": ["vegetarian"],
    --   "nutrition": {
    --     "fat_g": 22, "protein_g": 35, "carbs_g": 48,
    --     "sodium_mg": 680, "fiber_g": 4
    --   },
    --   "schema_org": {
    --     "@type": "MenuItem",
    --     "suitableForDiet": ["https://schema.org/VegetarianDiet"]
    --   },
    --   "modifiers": [
    --     {
    --       "group_name": "Temperature",
    --       "selection_type": "single",
    --       "required": true,
    --       "options": [
    --         {"name": "Rare", "price_adj": 0},
    --         {"name": "Medium Rare", "price_adj": 0},
    --         {"name": "Medium", "price_adj": 0},
    --         {"name": "Well Done", "price_adj": 0}
    --       ]
    --     },
    --     {
    --       "group_name": "Sides",
    --       "selection_type": "single",
    --       "required": true,
    --       "options": [
    --         {"name": "Fries", "price_adj": 0},
    --         {"name": "Salad", "price_adj": 0},
    --         {"name": "Sweet Potato Fries", "price_adj": 2.00}
    --       ]
    --     }
    --   ],
    --   "station_routing": ["grill"],
    --   "availability_override": {
    --     "locations": {"uuid-loc-2": {"price": 26.00, "is_available": false}}
    --   },
    --   "recipe": {
    --     "yield": 1,
    --     "ingredients": [
    --       {"name": "Beef Patty", "quantity": 200, "unit": "g"},
    --       {"name": "Brioche Bun", "quantity": 1, "unit": "each"},
    --       {"name": "Lettuce", "quantity": 30, "unit": "g"}
    --     ]
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_menu_item_org ON menu_item(organisation_id);
CREATE INDEX idx_menu_item_menu ON menu_item(menu_id);
CREATE INDEX idx_menu_item_details ON menu_item USING GIN (details);

-- Example: find all items containing milk allergen
-- SELECT * FROM menu_item
-- WHERE details->'allergens' ? 'milk'
--   AND organisation_id = '<<org-uuid>>';

-- Example: find all vegetarian items under $15
-- SELECT * FROM menu_item
-- WHERE details->'dietary_tags' ? 'vegetarian'
--   AND base_price < 15.00
--   AND is_active = true;

-- Example: get Schema.org JSON-LD export
-- SELECT jsonb_build_object(
--   '@context', 'https://schema.org',
--   '@type', 'MenuItem',
--   'name', name,
--   'description', description,
--   'offers', jsonb_build_object(
--     '@type', 'Offer',
--     'price', base_price,
--     'priceCurrency', currency_code
--   ),
--   'nutrition', details->'nutrition',
--   'suitableForDiet', details->'schema_org'->'suitableForDiet'
-- ) AS jsonld
-- FROM menu_item WHERE is_active = true;
```

---

## Floor Plan & Tables

```sql
-- ============================================================
-- FLOOR PLAN & TABLES — relational with JSONB layout
-- ============================================================

CREATE TABLE floor_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    name            TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: visual layout data for drag-and-drop UI
    layout          JSONB NOT NULL DEFAULT '{}',
    -- Example layout:
    -- {
    --   "width": 800, "height": 600,
    --   "background_image_url": null,
    --   "tables": [
    --     {"table_id": "uuid...", "x": 120, "y": 200, "width": 80, "height": 80, "rotation": 0},
    --     {"table_id": "uuid...", "x": 300, "y": 200, "width": 60, "height": 60, "rotation": 45}
    --   ],
    --   "walls": [{"x1": 0, "y1": 400, "x2": 800, "y2": 400}],
    --   "labels": [{"text": "Bar", "x": 400, "y": 50}]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_floor_plan_location ON floor_plan(location_id);

CREATE TABLE restaurant_table (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    floor_plan_id   UUID NOT NULL REFERENCES floor_plan(id),
    table_number    TEXT NOT NULL,
    capacity        INT NOT NULL DEFAULT 4,
    min_covers      INT NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    is_combinable   BOOLEAN NOT NULL DEFAULT true,
    -- Current status (denormalised for fast POS queries)
    status          TEXT NOT NULL DEFAULT 'available',
    current_server_id UUID REFERENCES staff_member(id),
    current_covers  INT,
    seated_at       TIMESTAMPTZ,
    active_order_id UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(floor_plan_id, table_number)
);

CREATE INDEX idx_table_floor ON restaurant_table(floor_plan_id);
```

---

## Orders & Payments

```sql
-- ============================================================
-- ORDERS — relational structure, JSONB for variable details
-- ============================================================

CREATE TABLE restaurant_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    table_id        UUID REFERENCES restaurant_table(id),
    server_id       UUID REFERENCES staff_member(id),
    guest_id        UUID,
    order_number    TEXT NOT NULL,
    order_type      TEXT NOT NULL DEFAULT 'dine_in',
    channel         TEXT NOT NULL DEFAULT 'pos',
    status          TEXT NOT NULL DEFAULT 'open',
    -- Monetary totals (relational for fast aggregation)
    subtotal        NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    tax_total       NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    discount_total  NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    tip_total       NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    service_charge  NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    grand_total     NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    guest_count     INT,
    -- JSONB: variable order metadata
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata:
    -- {
    --   "source_platform": "uber_eats",
    --   "external_order_id": "UE-20260522-abc123",
    --   "delivery": {
    --     "address": "123 Main St, Austin TX",
    --     "instructions": "Leave at door",
    --     "estimated_delivery": "2026-05-22T19:45:00-05:00",
    --     "driver": {"name": "Alex", "phone": "+1555..."}
    --   },
    --   "discounts_applied": [
    --     {"name": "Happy Hour 20%", "type": "percentage", "value": 20, "amount": 12.40}
    --   ],
    --   "taxes_applied": [
    --     {"name": "State Sales Tax", "rate": 0.0625, "taxable": 62.00, "amount": 3.88},
    --     {"name": "City Tax", "rate": 0.02, "taxable": 62.00, "amount": 1.24}
    --   ],
    --   "void_info": null,
    --   "notes": "Birthday dinner — bring cake with candles after mains"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_order_location ON restaurant_order(location_id, created_at);
CREATE INDEX idx_order_status ON restaurant_order(location_id, status);
CREATE INDEX idx_order_channel ON restaurant_order(channel, created_at);
CREATE INDEX idx_order_metadata ON restaurant_order USING GIN (metadata);

CREATE TABLE order_line_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES restaurant_order(id),
    menu_item_id    UUID REFERENCES menu_item(id),
    name            TEXT NOT NULL,
    kitchen_name    TEXT,
    quantity        INT NOT NULL DEFAULT 1,
    unit_price      NUMERIC(10,2) NOT NULL,
    line_total      NUMERIC(10,2) NOT NULL,
    course_number   INT DEFAULT 1,
    seat_number     INT,
    status          TEXT NOT NULL DEFAULT 'pending',
    fired_at        TIMESTAMPTZ,
    ready_at        TIMESTAMPTZ,
    served_at       TIMESTAMPTZ,
    -- JSONB: modifiers, special instructions, void info
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "modifiers": [
    --     {"name": "Medium Rare", "group": "Temperature", "price_adj": 0.00},
    --     {"name": "Sub Caesar Salad", "group": "Sides", "price_adj": 1.50}
    --   ],
    --   "notes": "No onions, extra sauce on side",
    --   "cost_snapshot": 7.85,
    --   "station": "grill",
    --   "void_info": {
    --     "voided_by": "uuid...",
    --     "reason": "Customer changed mind",
    --     "voided_at": "2026-05-22T19:32:00-05:00"
    --   }
    -- }
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
    -- JSONB: payment gateway details (never raw card data)
    gateway_data    JSONB NOT NULL DEFAULT '{}',
    -- Example gateway_data:
    -- {
    --   "gateway": "stripe",
    --   "payment_intent_id": "pi_3xyz...",
    --   "card_brand": "visa",
    --   "card_last_four": "4242",
    --   "auth_code": "ABC123",
    --   "receipt_url": "https://...",
    --   "refund": {
    --     "amount": 28.00,
    --     "reason": "Food quality issue",
    --     "refund_id": "re_xyz...",
    --     "refunded_at": "2026-05-22T20:15:00-05:00"
    --   }
    -- }
    processed_by    UUID REFERENCES staff_member(id),
    processed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_order ON payment_transaction(order_id);
```

---

## Inventory

```sql
-- ============================================================
-- INVENTORY — lightweight hybrid approach
-- ============================================================

CREATE TABLE ingredient (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    category        TEXT,
    unit_of_measure TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: variable ingredient properties
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties:
    -- {
    --   "cost_per_unit": 4.25,
    --   "currency": "USD",
    --   "par_levels": {
    --     "uuid-loc-1": {"par": 50, "unit": "kg"},
    --     "uuid-loc-2": {"par": 30, "unit": "kg"}
    --   },
    --   "storage": {"temp_min_c": 0, "temp_max_c": 4, "shelf_life_days": 7},
    --   "supplier_options": [
    --     {"supplier_id": "uuid...", "sku": "SUP-001", "unit_cost": 4.25, "min_order": 10},
    --     {"supplier_id": "uuid...", "sku": "ALT-001", "unit_cost": 4.50, "min_order": 5}
    --   ],
    --   "allergens": ["milk"],
    --   "organic": false,
    --   "origin_country": "US"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ingredient_org ON ingredient(organisation_id);
CREATE INDEX idx_ingredient_props ON ingredient USING GIN (properties);

CREATE TABLE inventory_stock (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    ingredient_id   UUID NOT NULL REFERENCES ingredient(id),
    quantity_on_hand NUMERIC(12,4) NOT NULL DEFAULT 0,
    unit_of_measure TEXT NOT NULL,
    last_counted_at TIMESTAMPTZ,
    last_received_at TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(location_id, ingredient_id)
);

CREATE INDEX idx_stock_location ON inventory_stock(location_id);

CREATE TABLE inventory_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    ingredient_id   UUID NOT NULL REFERENCES ingredient(id),
    transaction_type TEXT NOT NULL,
    quantity        NUMERIC(12,4) NOT NULL,
    unit_of_measure TEXT NOT NULL,
    -- JSONB: transaction context
    context         JSONB NOT NULL DEFAULT '{}',
    -- Example context (for a 'received' transaction):
    -- {
    --   "unit_cost": 4.25,
    --   "purchase_order_id": "uuid...",
    --   "supplier_id": "uuid...",
    --   "invoice_number": "INV-2026-0542",
    --   "expiry_date": "2026-06-15",
    --   "batch_number": "B-20260520"
    -- }
    -- Example context (for a 'wasted' transaction):
    -- {
    --   "reason": "expired",
    --   "estimated_value": 21.25,
    --   "waste_category": "spoilage"
    -- }
    performed_by    UUID REFERENCES staff_member(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inv_txn_location ON inventory_transaction(location_id, created_at);
CREATE INDEX idx_inv_txn_ingredient ON inventory_transaction(ingredient_id);
```

---

## Labour & Scheduling

```sql
-- ============================================================
-- LABOUR — relational scheduling, JSONB for compliance details
-- ============================================================

CREATE TABLE shift (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    staff_member_id UUID NOT NULL REFERENCES staff_member(id),
    role_name       TEXT,
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    status          TEXT NOT NULL DEFAULT 'scheduled',
    is_published    BOOLEAN NOT NULL DEFAULT false,
    -- JSONB: time tracking and compliance
    time_tracking   JSONB NOT NULL DEFAULT '{}',
    -- Example time_tracking:
    -- {
    --   "clock_in": "2026-05-22T16:58:00-05:00",
    --   "clock_out": "2026-05-22T23:05:00-05:00",
    --   "breaks": [
    --     {"start": "2026-05-22T19:00:00-05:00", "end": "2026-05-22T19:30:00-05:00", "type": "meal"}
    --   ],
    --   "total_minutes": 337,
    --   "break_minutes": 30,
    --   "worked_minutes": 307,
    --   "overtime_minutes": 0,
    --   "hourly_rate": 18.50,
    --   "total_pay": 94.66,
    --   "tips_earned": 145.00,
    --   "tip_pool_contribution": 29.00,
    --   "compliance": {
    --     "mandatory_break_met": true,
    --     "max_hours_met": true,
    --     "minor_labor_rules": null
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shift_location ON shift(location_id, start_time);
CREATE INDEX idx_shift_staff ON shift(staff_member_id, start_time);
CREATE INDEX idx_shift_status ON shift(location_id, status);
```

---

## Guests, Reservations & Loyalty

```sql
-- ============================================================
-- GUESTS & RESERVATIONS — relational core, JSONB preferences
-- ============================================================

CREATE TABLE guest (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           TEXT,
    phone           TEXT,
    first_name      TEXT,
    last_name       TEXT,
    display_name    TEXT,
    consent_status  TEXT NOT NULL DEFAULT 'none',
    is_vip          BOOLEAN NOT NULL DEFAULT false,
    -- JSONB: guest preferences and history summary
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example profile:
    -- {
    --   "allergens": ["shellfish", "peanuts"],
    --   "dietary": ["pescatarian"],
    --   "preferences": {
    --     "seating": "booth preferred",
    --     "wine": "prefers Pinot Noir",
    --     "occasions": ["anniversary: March 15"]
    --   },
    --   "stats": {
    --     "total_visits": 24,
    --     "total_spend": 3420.00,
    --     "avg_check": 142.50,
    --     "last_visit": "2026-05-15",
    --     "favourite_items": [
    --       {"id": "uuid...", "name": "Grilled Salmon", "count": 8},
    --       {"id": "uuid...", "name": "Caesar Salad", "count": 12}
    --     ]
    --   },
    --   "loyalty": {
    --     "points_balance": 2400,
    --     "lifetime_points": 8200,
    --     "tier": "gold",
    --     "enrolled_at": "2025-06-10"
    --   },
    --   "consent": {
    --     "marketing_email": true,
    --     "marketing_sms": false,
    --     "data_sharing": false,
    --     "consent_date": "2025-06-10",
    --     "retention_until": "2028-06-10"
    --   },
    --   "notes": "Husband is allergic to shellfish. Always orders dessert."
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_guest_org ON guest(organisation_id);
CREATE INDEX idx_guest_email ON guest(organisation_id, email);
CREATE INDEX idx_guest_phone ON guest(organisation_id, phone);
CREATE INDEX idx_guest_profile ON guest USING GIN (profile);

-- Example: find all guests with peanut allergy
-- SELECT * FROM guest
-- WHERE profile->'allergens' ? 'peanuts'
--   AND organisation_id = '<<org-uuid>>';

-- Example: find VIP guests who haven't visited in 30+ days
-- SELECT * FROM guest
-- WHERE is_vip = true
--   AND (profile->'stats'->>'last_visit')::DATE < CURRENT_DATE - 30
--   AND organisation_id = '<<org-uuid>>';

CREATE TABLE reservation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    guest_id        UUID REFERENCES guest(id),
    table_id        UUID REFERENCES restaurant_table(id),
    party_size      INT NOT NULL,
    reserved_at     TIMESTAMPTZ NOT NULL,
    duration_mins   INT NOT NULL DEFAULT 90,
    status          TEXT NOT NULL DEFAULT 'confirmed',
    source          TEXT NOT NULL DEFAULT 'direct',
    -- JSONB: reservation details and guest info
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "guest_name": "John Smith",
    --   "guest_phone": "+1555...",
    --   "guest_email": "john@...",
    --   "external_ref": "OT-20260522-xyz",
    --   "special_requests": "Quiet corner table, birthday cake at 8 PM",
    --   "dietary_notes": "2 vegetarian, 1 gluten-free",
    --   "communication_log": [
    --     {"type": "sms", "direction": "outbound", "message": "Reservation confirmed for 7 PM", "at": "..."},
    --     {"type": "sms", "direction": "inbound", "message": "Running 10 min late", "at": "..."}
    --   ],
    --   "cancellation": null
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reservation_location ON reservation(location_id, reserved_at);
CREATE INDEX idx_reservation_guest ON reservation(guest_id);
CREATE INDEX idx_reservation_status ON reservation(location_id, status);
```

---

## Food Safety & Audit

```sql
-- ============================================================
-- FOOD SAFETY — JSONB-driven checklists
-- ============================================================

CREATE TABLE food_safety_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    log_type        TEXT NOT NULL,                  -- temperature_check, opening_checklist, closing_checklist, line_check
    completed_by    UUID NOT NULL REFERENCES staff_member(id),
    shift_date      DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'complete',
    -- JSONB: the actual log data (structure varies by log_type)
    data            JSONB NOT NULL,
    -- Example data (temperature_check):
    -- {
    --   "readings": [
    --     {"equipment": "Walk-in Cooler", "temp_c": 3.2, "min": 0, "max": 4, "in_range": true},
    --     {"equipment": "Freezer #1", "temp_c": -18.5, "min": -22, "max": -15, "in_range": true},
    --     {"equipment": "Hot Hold", "temp_c": 58.0, "min": 57, "max": 100, "in_range": true}
    --   ]
    -- }
    -- Example data (opening_checklist):
    -- {
    --   "items": [
    --     {"task": "Handwash stations stocked", "completed": true},
    --     {"task": "Prep surfaces sanitised", "completed": true},
    --     {"task": "Deliveries checked and stored", "completed": true, "notes": "Milk delivery short by 2 units"},
    --     {"task": "Equipment temperatures logged", "completed": true}
    --   ],
    --   "corrective_actions": ["Contacted supplier re: missing milk units"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_safety_log_location ON food_safety_log(location_id, shift_date);
CREATE INDEX idx_safety_log_type ON food_safety_log(location_id, log_type);

-- ============================================================
-- AUDIT LOG
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
    -- JSONB: change details
    changes         JSONB,
    metadata        JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organisation_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_type ON audit_log(event_type, created_at);

-- ============================================================
-- SUPPLIER (lightweight)
-- ============================================================

CREATE TABLE supplier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: contact and terms
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details:
    -- {
    --   "contact_name": "Maria Garcia",
    --   "email": "orders@freshproduce.com",
    --   "phone": "+1555...",
    --   "address": "456 Market St, Austin TX",
    --   "payment_terms": "net_30",
    --   "categories": ["produce", "dairy"],
    --   "delivery_days": ["monday", "wednesday", "friday"],
    --   "minimum_order": 100.00
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_supplier_org ON supplier(organisation_id);
```

---

## Daily Summary

```sql
-- ============================================================
-- DAILY SUMMARY — reporting aggregate
-- ============================================================

CREATE TABLE daily_summary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    business_date   DATE NOT NULL,
    -- Core metrics (relational for fast BI queries)
    gross_sales     NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    net_sales       NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    tax_collected   NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    tips_collected  NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    order_count     INT NOT NULL DEFAULT 0,
    guest_count     INT NOT NULL DEFAULT 0,
    -- JSONB: detailed breakdown
    breakdown       JSONB NOT NULL DEFAULT '{}',
    -- Example breakdown:
    -- {
    --   "by_channel": {
    --     "dine_in": {"orders": 85, "revenue": 4250.00},
    --     "takeout": {"orders": 32, "revenue": 960.00},
    --     "uber_eats": {"orders": 18, "revenue": 540.00},
    --     "doordash": {"orders": 12, "revenue": 396.00}
    --   },
    --   "by_daypart": {
    --     "lunch": {"orders": 62, "revenue": 2480.00, "avg_check": 40.00},
    --     "dinner": {"orders": 85, "revenue": 3666.00, "avg_check": 43.13}
    --   },
    --   "labour": {
    --     "total_hours": 142.5,
    --     "total_cost": 2565.00,
    --     "labour_pct": 41.7
    --   },
    --   "food_cost": {
    --     "estimated_cost": 1830.00,
    --     "food_cost_pct": 29.8
    --   },
    --   "top_items": [
    --     {"name": "Grilled Salmon", "qty": 28, "revenue": 784.00},
    --     {"name": "Caesar Salad", "qty": 35, "revenue": 490.00}
    --   ],
    --   "voids": {"count": 3, "value": 42.00},
    --   "discounts": {"count": 8, "value": 96.00}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(location_id, business_date)
);

CREATE INDEX idx_daily_summary ON daily_summary(location_id, business_date);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Location | 2 | Relational core with JSONB config/settings |
| Staff | 1 | Single table with JSONB employment & permissions (replaces 4 tables in normalised model) |
| Menu Catalogue | 2 | menu + menu_item with JSONB details (replaces 9 tables in normalised model) |
| Floor Plan & Tables | 2 | floor_plan with JSONB layout, restaurant_table with denormalised status |
| Orders & Payments | 3 | order, line_item, payment with JSONB metadata/details |
| Inventory | 3 | ingredient, stock, transaction with JSONB context |
| Labour | 1 | shift with JSONB time_tracking (replaces 3 tables in normalised model) |
| Guests & Reservations | 2 | guest with JSONB profile, reservation with JSONB details |
| Food Safety | 1 | JSONB-driven checklists (replaces 3 tables in normalised model) |
| Reporting | 1 | daily_summary with JSONB breakdown |
| Supplier | 1 | JSONB contact and terms |
| Audit | 1 | JSONB change tracking |
| **Total** | **20** | vs. 51 in normalised model |

---

## Key Design Decisions

1. **20 tables instead of 51** — JSONB consolidation eliminates junction tables (menu_item_allergen, menu_item_dietary_tag, staff_role, staff_location, menu_section_item) and collapses related entities (modifier_group + modifier into menu_item.details, schedule + time_entry into shift.time_tracking). This dramatically simplifies ORM mapping and migration management.

2. **Relational columns for things you filter/aggregate, JSONB for things you display** — order totals, dates, statuses, and foreign keys are relational columns with indexes. Modifiers, special instructions, delivery details, and checklist responses are JSONB because they are primarily read and displayed, not filtered or joined.

3. **GIN indexes on all JSONB columns** — PostgreSQL GIN indexes support containment queries (`?`, `@>`, `?|`) on JSONB, enabling queries like "find all items with milk allergen" without a junction table. Performance is comparable to junction table lookups for moderate data volumes.

4. **Location config replaces 3+ configuration tables** — tax rules, operating hours, KDS station setup, tipping policies, and fiscal compliance settings all live in `location.config` JSONB. Adding support for a new jurisdiction (e.g., France's NF 525) means adding a new config key, not a schema migration.

5. **Guest profile as a rich document** — loyalty, preferences, visit stats, and consent all live in `guest.profile` JSONB. This mirrors how OpenTable evolved their guest model: a centralised document that aggregates data from multiple sources.

6. **Menu item modifiers are embedded, not relational** — modifier groups and options live inside `menu_item.details.modifiers`. This is simpler for menus that change frequently and eliminates the 3-table join (menu_item -> modifier_group -> modifier) on every POS order screen load.

7. **Food safety checklists are schema-driven JSONB** — checklist templates are defined in `location.config.food_safety` and responses are stored as JSONB in `food_safety_log.data`. Adding a new checklist type is a configuration change, not a schema change. JSON Schema validation at the application layer ensures data quality.

8. **Shift time tracking consolidates schedule + clock data** — a single `shift` row tracks both the scheduled time and actual clock-in/out/break data in JSONB, eliminating the need for separate `schedule`, `shift`, and `time_entry` tables.

9. **Daily summary uses JSONB for flexible breakdowns** — core metrics (gross_sales, order_count) are relational for BI tool compatibility, while detailed breakdowns (by channel, by daypart, top items) are JSONB for flexibility.

10. **JSON Schema validation is mandatory at the application layer** — every JSONB column has a corresponding JSON Schema definition in the application code. This replaces database-level referential integrity with application-level structural validation, a trade-off that speeds development but requires disciplined schema governance.
