# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Restaurant Management Platform · Created: 2026-05-22

## Philosophy

This model follows classical third-normal-form (3NF) relational design where every real-world concept gets its own table with explicit foreign key relationships. The schema is optimised for data integrity, referential consistency, and complex cross-entity queries -- the kind of queries restaurant operators run daily ("show me labour cost as a percentage of revenue by location for the last 90 days, broken down by daypart").

Normalised relational models are the backbone of every major POS platform (Toast, Square, Aloha). They map naturally to the domain: a restaurant has locations, locations have floor plans with tables, tables receive orders, orders contain line items drawn from a menu catalogue, line items consume inventory ingredients via recipes. Each of these nouns becomes a table; each verb becomes a foreign key or junction table.

This approach is best suited for teams with strong SQL skills who need a production-grade schema that supports complex reporting, enforces business rules at the database level, and integrates cleanly with standard BI tools. It is the safest choice for a v1 MVP where correctness matters more than schema flexibility.

**Best for:** Teams prioritising data integrity, complex analytics, and regulatory compliance (PCI DSS audit trails, HACCP temperature logs, GDPR data subject access requests).

**Trade-offs:**
- Pro: Strong referential integrity prevents orphaned records and inconsistent state
- Pro: Mature tooling -- every ORM, migration tool, and BI platform works natively
- Pro: Complex analytical queries (joins across 5-8 tables) are well-optimised by PostgreSQL
- Pro: Standards-aligned field naming makes API design straightforward
- Con: Schema changes require migrations, which slow down rapid iteration
- Con: High table count (50+) increases onboarding complexity for new developers
- Con: Multi-location variation (different menus, tax rules, tip policies) requires careful modelling to avoid wide tables or excessive nullable columns
- Con: Deeply nested menu hierarchies (menu > section > item > modifier group > modifier) require multi-way joins for common queries

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Schema.org Restaurant/Menu/MenuItem | Menu hierarchy mirrors Schema.org vocabulary: `menu` > `menu_section` > `menu_item`, enabling JSON-LD export for SEO |
| PCI DSS v4.0 | Payment data stored via tokenised references only; `payment_transaction` stores gateway tokens, never raw card data |
| ISO 22000:2018 / HACCP | `food_safety_log` and `temperature_reading` tables support HACCP critical control point documentation |
| FDA Allergen Guidance (2025) | `allergen` reference table covers all 9 FDA-recognised allergens; `menu_item_allergen` junction table |
| ISO 3166-1/2 | `location.country_code` uses ISO 3166-1 alpha-2; `tax_rate.jurisdiction` uses ISO 3166-2 for state/province |
| ISO 4217 | All monetary columns reference `currency_code` using ISO 4217 three-letter codes |
| CloudEvents 1.0 | `audit_log` follows CloudEvents envelope structure for webhook dispatch |
| OpenAPI 3.1 | Table/column naming conventions designed for direct mapping to OpenAPI resource schemas |
| GDPR / CCPA | `guest` table includes `consent_status`, `data_retention_until`, and supports right-to-erasure via soft delete |

---

## Organisation & Tenant Layer

```sql
-- ============================================================
-- ORGANISATION & MULTI-TENANCY
-- ============================================================

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,           -- URL-safe identifier
    billing_email   TEXT,
    plan_tier       TEXT NOT NULL DEFAULT 'standard', -- free, standard, premium, enterprise
    settings_json   JSONB NOT NULL DEFAULT '{}',    -- org-level feature flags
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE location (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    state_province  TEXT,
    postal_code     TEXT,
    country_code    CHAR(2) NOT NULL DEFAULT 'US',  -- ISO 3166-1 alpha-2
    timezone        TEXT NOT NULL DEFAULT 'America/New_York', -- IANA timezone
    latitude        NUMERIC(9,6),
    longitude       NUMERIC(9,6),
    phone           TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    is_ghost_kitchen BOOLEAN NOT NULL DEFAULT false,
    settings_json   JSONB NOT NULL DEFAULT '{}',    -- location overrides
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, slug)
);

CREATE INDEX idx_location_org ON location(organisation_id);
CREATE INDEX idx_location_country ON location(country_code);

-- Row-Level Security policy template
ALTER TABLE location ENABLE ROW LEVEL SECURITY;
CREATE POLICY location_tenant_isolation ON location
    USING (organisation_id = current_setting('app.current_org_id')::UUID);
```

---

## Staff & Access Control

```sql
-- ============================================================
-- STAFF & RBAC
-- ============================================================

CREATE TABLE staff_member (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           TEXT,
    phone           TEXT,
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    display_name    TEXT,
    pin_hash        TEXT,                           -- hashed POS login PIN
    hourly_rate     NUMERIC(10,2),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    employment_type TEXT NOT NULL DEFAULT 'hourly', -- hourly, salaried, contractor
    hire_date       DATE,
    termination_date DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_staff_org ON staff_member(organisation_id);
CREATE INDEX idx_staff_active ON staff_member(organisation_id, is_active);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,                  -- manager, server, bartender, host, cook, admin
    permissions     JSONB NOT NULL DEFAULT '[]',    -- ["pos.void_order", "inventory.adjust", ...]
    is_system       BOOLEAN NOT NULL DEFAULT false, -- true for built-in roles
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, name)
);

CREATE TABLE staff_role (
    staff_member_id UUID NOT NULL REFERENCES staff_member(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    location_id     UUID REFERENCES location(id),  -- NULL = all locations
    PRIMARY KEY (staff_member_id, role_id, location_id)
);

CREATE TABLE staff_location (
    staff_member_id UUID NOT NULL REFERENCES staff_member(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    PRIMARY KEY (staff_member_id, location_id)
);
```

---

## Menu Catalogue

```sql
-- ============================================================
-- MENU CATALOGUE (mirrors Schema.org Menu/MenuSection/MenuItem)
-- ============================================================

CREATE TABLE menu (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,                  -- "Lunch Menu", "Happy Hour", "Brunch"
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE menu_location (
    menu_id         UUID NOT NULL REFERENCES menu(id) ON DELETE CASCADE,
    location_id     UUID NOT NULL REFERENCES location(id),
    PRIMARY KEY (menu_id, location_id)
);

CREATE TABLE menu_section (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    menu_id         UUID NOT NULL REFERENCES menu(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,                  -- "Appetizers", "Mains", "Desserts"
    description     TEXT,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_menu_section_menu ON menu_section(menu_id);

CREATE TABLE menu_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    kitchen_name    TEXT,                           -- shorter name for KDS display (Square pattern)
    description     TEXT,
    sku             TEXT,
    base_price      NUMERIC(10,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    cost_price      NUMERIC(10,2),                 -- food cost for margin calculation
    tax_category    TEXT,                           -- mapped to tax rules per jurisdiction
    calories        INT,                            -- FDA menu labelling (Section 4205)
    prep_time_mins  INT,                            -- estimated prep time for KDS sequencing
    is_active       BOOLEAN NOT NULL DEFAULT true,
    is_alcoholic    BOOLEAN NOT NULL DEFAULT false,
    image_url       TEXT,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_menu_item_org ON menu_item(organisation_id);
CREATE INDEX idx_menu_item_sku ON menu_item(organisation_id, sku);

CREATE TABLE menu_section_item (
    menu_section_id UUID NOT NULL REFERENCES menu_section(id) ON DELETE CASCADE,
    menu_item_id    UUID NOT NULL REFERENCES menu_item(id),
    sort_order      INT NOT NULL DEFAULT 0,
    PRIMARY KEY (menu_section_id, menu_item_id)
);

-- Modifier groups (e.g., "Choose your protein", "Add extras")
CREATE TABLE modifier_group (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,                  -- "Temperature", "Side Choice", "Add-ons"
    selection_type  TEXT NOT NULL DEFAULT 'single', -- single, multi, quantity
    min_selections  INT NOT NULL DEFAULT 0,
    max_selections  INT,
    is_required     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE menu_item_modifier_group (
    menu_item_id      UUID NOT NULL REFERENCES menu_item(id) ON DELETE CASCADE,
    modifier_group_id UUID NOT NULL REFERENCES modifier_group(id),
    sort_order        INT NOT NULL DEFAULT 0,
    PRIMARY KEY (menu_item_id, modifier_group_id)
);

CREATE TABLE modifier (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    modifier_group_id UUID NOT NULL REFERENCES modifier_group(id) ON DELETE CASCADE,
    name              TEXT NOT NULL,
    price_adjustment  NUMERIC(10,2) NOT NULL DEFAULT 0.00,
    is_default        BOOLEAN NOT NULL DEFAULT false,
    sort_order        INT NOT NULL DEFAULT 0,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_modifier_group ON modifier(modifier_group_id);

-- Allergens (FDA 9 major allergens + custom)
CREATE TABLE allergen (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            TEXT NOT NULL UNIQUE,           -- 'milk', 'eggs', 'fish', 'shellfish', 'tree_nuts', 'peanuts', 'wheat', 'soybeans', 'sesame'
    name            TEXT NOT NULL,
    is_fda_major    BOOLEAN NOT NULL DEFAULT false,
    icon_url        TEXT
);

CREATE TABLE menu_item_allergen (
    menu_item_id    UUID NOT NULL REFERENCES menu_item(id) ON DELETE CASCADE,
    allergen_id     UUID NOT NULL REFERENCES allergen(id),
    severity        TEXT NOT NULL DEFAULT 'contains', -- contains, may_contain, trace
    PRIMARY KEY (menu_item_id, allergen_id)
);

-- Dietary tags
CREATE TABLE dietary_tag (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            TEXT NOT NULL UNIQUE,           -- 'vegetarian', 'vegan', 'gluten_free', 'halal', 'kosher'
    name            TEXT NOT NULL,
    schema_org_diet TEXT                            -- maps to Schema.org RestrictedDiet enum
);

CREATE TABLE menu_item_dietary_tag (
    menu_item_id    UUID NOT NULL REFERENCES menu_item(id) ON DELETE CASCADE,
    dietary_tag_id  UUID NOT NULL REFERENCES dietary_tag(id),
    PRIMARY KEY (menu_item_id, dietary_tag_id)
);
```

---

## Floor Plan & Table Management

```sql
-- ============================================================
-- FLOOR PLAN & TABLE MANAGEMENT
-- ============================================================

CREATE TABLE floor_plan (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    name            TEXT NOT NULL,                  -- "Main Dining", "Patio", "Bar Area"
    is_active       BOOLEAN NOT NULL DEFAULT true,
    layout_json     JSONB,                          -- visual layout coordinates for drag-and-drop UI
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_floor_plan_location ON floor_plan(location_id);

CREATE TABLE restaurant_table (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    floor_plan_id   UUID NOT NULL REFERENCES floor_plan(id),
    table_number    TEXT NOT NULL,                  -- "A1", "Bar-3", "Patio-7"
    capacity        INT NOT NULL DEFAULT 4,
    min_covers      INT NOT NULL DEFAULT 1,
    shape           TEXT NOT NULL DEFAULT 'rectangle', -- rectangle, round, square, booth
    x_position      NUMERIC(8,2),                  -- floor plan coordinate
    y_position      NUMERIC(8,2),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    is_combinable   BOOLEAN NOT NULL DEFAULT true,  -- can be joined with adjacent tables
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(floor_plan_id, table_number)
);

CREATE INDEX idx_table_floor_plan ON restaurant_table(floor_plan_id);

-- Tracks real-time table status (one active session per table)
CREATE TABLE table_session (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_id        UUID NOT NULL REFERENCES restaurant_table(id),
    server_id       UUID REFERENCES staff_member(id),
    status          TEXT NOT NULL DEFAULT 'available', -- available, seated, ordering, served, check_presented, cleaning
    covers          INT,
    seated_at       TIMESTAMPTZ,
    cleared_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_table_session_table ON table_session(table_id);
CREATE INDEX idx_table_session_status ON table_session(table_id, status) WHERE cleared_at IS NULL;
```

---

## Orders & Payments

```sql
-- ============================================================
-- ORDERS & PAYMENTS
-- ============================================================

CREATE TABLE restaurant_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    table_session_id UUID REFERENCES table_session(id),
    server_id       UUID REFERENCES staff_member(id),
    guest_id        UUID,                           -- FK to guest table (nullable for walk-ins)
    order_number    TEXT NOT NULL,                  -- human-readable, location-scoped
    order_type      TEXT NOT NULL DEFAULT 'dine_in', -- dine_in, takeout, delivery, drive_thru, online
    status          TEXT NOT NULL DEFAULT 'open',   -- open, sent_to_kitchen, in_progress, ready, served, closed, cancelled, voided
    channel         TEXT NOT NULL DEFAULT 'pos',    -- pos, online, uber_eats, doordash, grubhub, phone
    subtotal        NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    tax_total       NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    discount_total  NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    tip_total       NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    service_charge  NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    grand_total     NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    guest_count     INT,
    notes           TEXT,
    voided_by       UUID REFERENCES staff_member(id),
    void_reason     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_order_location ON restaurant_order(location_id);
CREATE INDEX idx_order_location_date ON restaurant_order(location_id, created_at);
CREATE INDEX idx_order_status ON restaurant_order(location_id, status);
CREATE INDEX idx_order_channel ON restaurant_order(location_id, channel);

CREATE TABLE order_line_item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES restaurant_order(id),
    menu_item_id    UUID REFERENCES menu_item(id),
    name            TEXT NOT NULL,                  -- snapshot of item name at order time
    kitchen_name    TEXT,
    quantity        INT NOT NULL DEFAULT 1,
    unit_price      NUMERIC(10,2) NOT NULL,
    line_total      NUMERIC(10,2) NOT NULL,
    cost_price      NUMERIC(10,2),                 -- snapshot of cost at order time
    course_number   INT DEFAULT 1,                 -- for course sequencing / fire timing
    seat_number     INT,
    status          TEXT NOT NULL DEFAULT 'pending', -- pending, sent, in_progress, ready, served, voided
    fired_at        TIMESTAMPTZ,                   -- when sent to kitchen
    ready_at        TIMESTAMPTZ,
    served_at       TIMESTAMPTZ,
    notes           TEXT,                           -- special instructions
    voided_by       UUID REFERENCES staff_member(id),
    void_reason     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_line_item_order ON order_line_item(order_id);
CREATE INDEX idx_line_item_menu_item ON order_line_item(menu_item_id);

CREATE TABLE order_line_item_modifier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    line_item_id    UUID NOT NULL REFERENCES order_line_item(id) ON DELETE CASCADE,
    modifier_id     UUID REFERENCES modifier(id),
    name            TEXT NOT NULL,                  -- snapshot
    price_adjustment NUMERIC(10,2) NOT NULL DEFAULT 0.00,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_line_modifier_item ON order_line_item_modifier(line_item_id);

CREATE TABLE order_discount (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES restaurant_order(id),
    line_item_id    UUID REFERENCES order_line_item(id), -- NULL = order-level discount
    discount_name   TEXT NOT NULL,
    discount_type   TEXT NOT NULL,                  -- percentage, fixed_amount, comp
    discount_value  NUMERIC(10,2) NOT NULL,
    applied_amount  NUMERIC(10,2) NOT NULL,
    applied_by      UUID REFERENCES staff_member(id),
    reason          TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE order_tax (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES restaurant_order(id),
    tax_name        TEXT NOT NULL,                  -- "State Sales Tax", "City Tax", "Alcohol Tax"
    tax_rate        NUMERIC(6,4) NOT NULL,          -- 0.0875 for 8.75%
    taxable_amount  NUMERIC(12,2) NOT NULL,
    tax_amount      NUMERIC(12,2) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Payment transactions store tokenised references only (PCI DSS compliance)
CREATE TABLE payment_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES restaurant_order(id),
    payment_method  TEXT NOT NULL,                  -- card, cash, gift_card, mobile_pay, split
    status          TEXT NOT NULL DEFAULT 'pending', -- pending, authorized, captured, refunded, failed, voided
    amount          NUMERIC(12,2) NOT NULL,
    tip_amount      NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    gateway         TEXT,                           -- stripe, square, toast_payments
    gateway_txn_id  TEXT,                           -- external transaction reference (tokenised)
    card_brand      TEXT,                           -- visa, mastercard, amex (no raw card data per PCI DSS)
    card_last_four  CHAR(4),
    processed_by    UUID REFERENCES staff_member(id),
    processed_at    TIMESTAMPTZ,
    refund_amount   NUMERIC(12,2),
    refund_reason   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_order ON payment_transaction(order_id);
CREATE INDEX idx_payment_gateway ON payment_transaction(gateway, gateway_txn_id);
```

---

## Inventory & Recipes

```sql
-- ============================================================
-- INVENTORY & RECIPE MANAGEMENT
-- ============================================================

CREATE TABLE ingredient (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    category        TEXT,                           -- produce, protein, dairy, dry_goods, beverages
    unit_of_measure TEXT NOT NULL,                  -- kg, g, l, ml, each, oz, lb
    cost_per_unit   NUMERIC(10,4),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    par_level       NUMERIC(10,2),                 -- minimum stock before reorder
    shelf_life_days INT,
    storage_temp_min NUMERIC(5,1),                 -- HACCP: min storage temperature (Celsius)
    storage_temp_max NUMERIC(5,1),                 -- HACCP: max storage temperature
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ingredient_org ON ingredient(organisation_id);

CREATE TABLE recipe (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    menu_item_id    UUID NOT NULL REFERENCES menu_item(id),
    yield_quantity  NUMERIC(10,2) NOT NULL DEFAULT 1,
    yield_unit      TEXT NOT NULL DEFAULT 'serving',
    prep_time_mins  INT,
    cook_time_mins  INT,
    instructions    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE recipe_ingredient (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recipe_id       UUID NOT NULL REFERENCES recipe(id) ON DELETE CASCADE,
    ingredient_id   UUID NOT NULL REFERENCES ingredient(id),
    quantity        NUMERIC(10,4) NOT NULL,
    unit_of_measure TEXT NOT NULL,
    waste_factor    NUMERIC(5,4) DEFAULT 0.00,     -- e.g., 0.10 for 10% prep waste
    is_optional     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_recipe_ingredient_recipe ON recipe_ingredient(recipe_id);

CREATE TABLE inventory_stock (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    ingredient_id   UUID NOT NULL REFERENCES ingredient(id),
    quantity_on_hand NUMERIC(12,4) NOT NULL DEFAULT 0,
    unit_of_measure TEXT NOT NULL,
    last_counted_at TIMESTAMPTZ,
    last_received_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(location_id, ingredient_id)
);

CREATE INDEX idx_stock_location ON inventory_stock(location_id);

CREATE TABLE inventory_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    ingredient_id   UUID NOT NULL REFERENCES ingredient(id),
    transaction_type TEXT NOT NULL,                 -- received, used, wasted, transferred, counted, adjusted
    quantity         NUMERIC(12,4) NOT NULL,        -- positive = in, negative = out
    unit_of_measure  TEXT NOT NULL,
    unit_cost        NUMERIC(10,4),
    reference_type   TEXT,                          -- order, purchase_order, waste_log, transfer, count
    reference_id     UUID,
    performed_by     UUID REFERENCES staff_member(id),
    notes            TEXT,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_inv_txn_location ON inventory_transaction(location_id);
CREATE INDEX idx_inv_txn_ingredient ON inventory_transaction(ingredient_id);
CREATE INDEX idx_inv_txn_date ON inventory_transaction(location_id, created_at);

CREATE TABLE supplier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    contact_name    TEXT,
    email           TEXT,
    phone           TEXT,
    address         TEXT,
    payment_terms   TEXT,                           -- net_30, cod, prepaid
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE purchase_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    po_number       TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',  -- draft, submitted, confirmed, received, cancelled
    ordered_at      TIMESTAMPTZ,
    expected_at     TIMESTAMPTZ,
    received_at     TIMESTAMPTZ,
    total_amount    NUMERIC(12,2),
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    notes           TEXT,
    created_by      UUID REFERENCES staff_member(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE purchase_order_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id UUID NOT NULL REFERENCES purchase_order(id) ON DELETE CASCADE,
    ingredient_id   UUID NOT NULL REFERENCES ingredient(id),
    quantity_ordered NUMERIC(12,4) NOT NULL,
    quantity_received NUMERIC(12,4),
    unit_of_measure TEXT NOT NULL,
    unit_cost       NUMERIC(10,4) NOT NULL,
    line_total      NUMERIC(12,2) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Labour & Scheduling

```sql
-- ============================================================
-- LABOUR & SCHEDULING
-- ============================================================

CREATE TABLE schedule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    week_start_date DATE NOT NULL,                 -- Monday of the schedule week
    status          TEXT NOT NULL DEFAULT 'draft',  -- draft, published, locked
    published_at    TIMESTAMPTZ,
    published_by    UUID REFERENCES staff_member(id),
    labour_budget   NUMERIC(12,2),                 -- target labour cost for the week
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(location_id, week_start_date)
);

CREATE TABLE shift (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schedule_id     UUID NOT NULL REFERENCES schedule(id),
    staff_member_id UUID NOT NULL REFERENCES staff_member(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    role_id         UUID REFERENCES role(id),
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    break_minutes   INT NOT NULL DEFAULT 0,
    is_published     BOOLEAN NOT NULL DEFAULT false,
    status          TEXT NOT NULL DEFAULT 'scheduled', -- scheduled, confirmed, no_show, completed
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_shift_schedule ON shift(schedule_id);
CREATE INDEX idx_shift_staff ON shift(staff_member_id);
CREATE INDEX idx_shift_location_date ON shift(location_id, start_time);

CREATE TABLE time_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    staff_member_id UUID NOT NULL REFERENCES staff_member(id),
    location_id     UUID NOT NULL REFERENCES location(id),
    shift_id        UUID REFERENCES shift(id),
    clock_in        TIMESTAMPTZ NOT NULL,
    clock_out       TIMESTAMPTZ,
    break_start     TIMESTAMPTZ,
    break_end       TIMESTAMPTZ,
    total_minutes   INT,                           -- computed on clock_out
    overtime_minutes INT DEFAULT 0,
    hourly_rate     NUMERIC(10,2) NOT NULL,
    total_pay       NUMERIC(10,2),                 -- computed
    tips_earned     NUMERIC(10,2) DEFAULT 0.00,
    status          TEXT NOT NULL DEFAULT 'active', -- active, completed, adjusted
    adjusted_by     UUID REFERENCES staff_member(id),
    adjustment_note TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_time_entry_staff ON time_entry(staff_member_id);
CREATE INDEX idx_time_entry_location ON time_entry(location_id, clock_in);
```

---

## Guests, Reservations & Loyalty

```sql
-- ============================================================
-- GUESTS, RESERVATIONS & LOYALTY
-- ============================================================

CREATE TABLE guest (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           TEXT,
    phone           TEXT,
    first_name      TEXT,
    last_name       TEXT,
    display_name    TEXT,
    notes           TEXT,                           -- dietary preferences, VIP notes
    consent_status  TEXT NOT NULL DEFAULT 'none',   -- none, opted_in, opted_out (GDPR)
    consent_date    TIMESTAMPTZ,
    data_retention_until TIMESTAMPTZ,              -- GDPR: auto-purge date
    total_visits    INT NOT NULL DEFAULT 0,
    total_spend     NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    last_visit_at   TIMESTAMPTZ,
    is_vip          BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_guest_org ON guest(organisation_id);
CREATE INDEX idx_guest_email ON guest(organisation_id, email);
CREATE INDEX idx_guest_phone ON guest(organisation_id, phone);

CREATE TABLE guest_allergen (
    guest_id        UUID NOT NULL REFERENCES guest(id) ON DELETE CASCADE,
    allergen_id     UUID NOT NULL REFERENCES allergen(id),
    notes           TEXT,
    PRIMARY KEY (guest_id, allergen_id)
);

CREATE TABLE reservation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    guest_id        UUID REFERENCES guest(id),
    table_id        UUID REFERENCES restaurant_table(id),
    party_size      INT NOT NULL,
    reserved_at     TIMESTAMPTZ NOT NULL,           -- date/time of the reservation
    duration_mins   INT NOT NULL DEFAULT 90,
    status          TEXT NOT NULL DEFAULT 'confirmed', -- pending, confirmed, seated, completed, cancelled, no_show
    source          TEXT NOT NULL DEFAULT 'direct',  -- direct, opentable, resy, yelp, phone, walk_in
    external_ref    TEXT,                           -- OpenTable/Resy booking reference
    guest_name      TEXT,                           -- for walk-ins without guest profile
    guest_phone     TEXT,
    guest_email     TEXT,
    special_requests TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reservation_location ON reservation(location_id, reserved_at);
CREATE INDEX idx_reservation_guest ON reservation(guest_id);
CREATE INDEX idx_reservation_status ON reservation(location_id, status, reserved_at);

CREATE TABLE loyalty_program (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    points_per_dollar NUMERIC(6,2) NOT NULL DEFAULT 1.00,
    redemption_threshold INT NOT NULL DEFAULT 100,
    reward_value    NUMERIC(10,2) NOT NULL DEFAULT 10.00,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE loyalty_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    loyalty_program_id UUID NOT NULL REFERENCES loyalty_program(id),
    guest_id        UUID NOT NULL REFERENCES guest(id),
    points_balance  INT NOT NULL DEFAULT 0,
    lifetime_points INT NOT NULL DEFAULT 0,
    tier            TEXT NOT NULL DEFAULT 'standard', -- standard, silver, gold, platinum
    enrolled_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(loyalty_program_id, guest_id)
);

CREATE TABLE loyalty_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    loyalty_account_id UUID NOT NULL REFERENCES loyalty_account(id),
    order_id        UUID REFERENCES restaurant_order(id),
    transaction_type TEXT NOT NULL,                 -- earn, redeem, adjust, expire
    points          INT NOT NULL,                   -- positive = earn, negative = redeem
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_loyalty_txn_account ON loyalty_transaction(loyalty_account_id);
```

---

## Food Safety & Compliance

```sql
-- ============================================================
-- FOOD SAFETY & COMPLIANCE (HACCP / FSMA)
-- ============================================================

CREATE TABLE food_safety_checklist (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,                  -- "Opening Checklist", "Closing Checklist", "Line Check"
    frequency       TEXT NOT NULL,                  -- daily, per_shift, weekly
    items_json      JSONB NOT NULL,                 -- checklist item definitions
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE food_safety_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    checklist_id    UUID REFERENCES food_safety_checklist(id),
    completed_by    UUID NOT NULL REFERENCES staff_member(id),
    shift_date      DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'complete', -- complete, incomplete, issue_flagged
    responses_json  JSONB NOT NULL,                -- completed checklist responses
    notes           TEXT,
    completed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_safety_log_location ON food_safety_log(location_id, shift_date);

CREATE TABLE temperature_reading (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    equipment_name  TEXT NOT NULL,                  -- "Walk-in Cooler #1", "Freezer", "Hot Hold Station"
    reading_celsius NUMERIC(5,1) NOT NULL,
    is_in_range     BOOLEAN NOT NULL,
    min_threshold   NUMERIC(5,1) NOT NULL,
    max_threshold   NUMERIC(5,1) NOT NULL,
    recorded_by     UUID REFERENCES staff_member(id),
    corrective_action TEXT,                        -- if out of range, what was done
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_temp_reading_location ON temperature_reading(location_id, created_at);
```

---

## Kitchen Display & Online Ordering

```sql
-- ============================================================
-- KITCHEN DISPLAY SYSTEM
-- ============================================================

CREATE TABLE kitchen_station (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    name            TEXT NOT NULL,                  -- "Grill", "Saute", "Fry", "Cold", "Dessert", "Bar", "Expo"
    display_order   INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE menu_item_station (
    menu_item_id    UUID NOT NULL REFERENCES menu_item(id),
    station_id      UUID NOT NULL REFERENCES kitchen_station(id),
    PRIMARY KEY (menu_item_id, station_id)
);

-- ============================================================
-- ONLINE ORDERING & DELIVERY
-- ============================================================

CREATE TABLE online_order_config (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    channel         TEXT NOT NULL,                  -- first_party, uber_eats, doordash, grubhub
    is_active       BOOLEAN NOT NULL DEFAULT true,
    external_store_id TEXT,                         -- store ID on the delivery platform
    commission_rate NUMERIC(5,4),                   -- platform commission (e.g., 0.30 for 30%)
    settings_json   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(location_id, channel)
);

CREATE TABLE delivery_fulfillment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES restaurant_order(id),
    channel         TEXT NOT NULL,                  -- first_party, uber_eats, doordash
    external_order_id TEXT,
    delivery_address TEXT,
    delivery_instructions TEXT,
    estimated_pickup TIMESTAMPTZ,
    estimated_delivery TIMESTAMPTZ,
    actual_pickup   TIMESTAMPTZ,
    actual_delivery TIMESTAMPTZ,
    driver_name     TEXT,
    driver_phone    TEXT,
    status          TEXT NOT NULL DEFAULT 'pending', -- pending, accepted, preparing, ready_for_pickup, picked_up, delivered, cancelled
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_delivery_order ON delivery_fulfillment(order_id);
```

---

## Reporting & Audit

```sql
-- ============================================================
-- DAILY RECONCILIATION & REPORTING
-- ============================================================

CREATE TABLE daily_summary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    business_date   DATE NOT NULL,
    gross_sales     NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    net_sales       NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    tax_collected   NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    tips_collected  NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    discounts_given NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    refunds_issued  NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    order_count     INT NOT NULL DEFAULT 0,
    guest_count     INT NOT NULL DEFAULT 0,
    avg_check       NUMERIC(10,2),
    labour_cost     NUMERIC(12,2),
    food_cost       NUMERIC(12,2),
    labour_pct      NUMERIC(5,2),                  -- labour cost / net sales
    food_cost_pct   NUMERIC(5,2),                  -- food cost / net sales
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(location_id, business_date)
);

CREATE INDEX idx_daily_summary_date ON daily_summary(location_id, business_date);

-- ============================================================
-- AUDIT LOG (CloudEvents-aligned)
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    location_id     UUID REFERENCES location(id),
    actor_id        UUID,                           -- staff_member or system
    actor_type      TEXT NOT NULL DEFAULT 'staff',  -- staff, system, api_key, webhook
    event_type      TEXT NOT NULL,                  -- order.created, payment.captured, inventory.adjusted, staff.clocked_in
    event_source    TEXT NOT NULL DEFAULT 'pos',    -- pos, api, webhook, system, backoffice
    entity_type     TEXT NOT NULL,                  -- order, payment, menu_item, staff_member, etc.
    entity_id       UUID NOT NULL,
    changes_json    JSONB,                          -- {field: {old: x, new: y}}
    metadata_json   JSONB,                          -- additional context
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organisation_id, created_at);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_actor ON audit_log(actor_id, created_at);
CREATE INDEX idx_audit_event_type ON audit_log(event_type, created_at);
```

---

## Tax Configuration

```sql
-- ============================================================
-- TAX CONFIGURATION
-- ============================================================

CREATE TABLE tax_rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,                  -- "State Sales Tax", "City Tax", "Alcohol Tax"
    rate            NUMERIC(6,4) NOT NULL,          -- 0.0875 for 8.75%
    jurisdiction    TEXT,                           -- ISO 3166-2 code (e.g., US-NY)
    applies_to      TEXT NOT NULL DEFAULT 'all',    -- all, food, alcohol, non_food
    is_inclusive    BOOLEAN NOT NULL DEFAULT false,  -- true for VAT-style taxes
    is_active       BOOLEAN NOT NULL DEFAULT true,
    effective_from  DATE,
    effective_until DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE location_tax_rate (
    location_id     UUID NOT NULL REFERENCES location(id),
    tax_rate_id     UUID NOT NULL REFERENCES tax_rate(id),
    PRIMARY KEY (location_id, tax_rate_id)
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Tenancy | 2 | organisation, location |
| Staff & RBAC | 4 | staff_member, role, staff_role, staff_location |
| Menu Catalogue | 9 | menu through dietary_tag junction tables |
| Floor Plan & Tables | 3 | floor_plan, restaurant_table, table_session |
| Orders & Payments | 6 | restaurant_order through payment_transaction |
| Inventory & Recipes | 7 | ingredient through purchase_order_line |
| Labour & Scheduling | 3 | schedule, shift, time_entry |
| Guests & Loyalty | 6 | guest through loyalty_transaction |
| Food Safety | 3 | checklist, log, temperature_reading |
| Kitchen & Delivery | 4 | kitchen_station, menu_item_station, online_order_config, delivery_fulfillment |
| Reporting & Audit | 2 | daily_summary, audit_log |
| Tax Configuration | 2 | tax_rate, location_tax_rate |
| **Total** | **51** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** -- enables multi-location sync, offline operation, and distributed ID generation without coordination. Critical for POS terminals that may operate offline.

2. **Organisation > Location hierarchy with RLS** -- every tenant-scoped table includes `organisation_id` as the leading column in composite indexes. PostgreSQL Row-Level Security policies enforce data isolation automatically.

3. **Menu items are organisation-scoped, menus are location-assigned** -- a single menu item catalogue is maintained at the org level, but menus (collections of sections and items) are assigned to specific locations via `menu_location`. This supports ghost kitchens sharing items across virtual brands.

4. **Order line items snapshot pricing** -- `unit_price`, `name`, and `cost_price` are copied to the line item at order time, not looked up via FK. This preserves historical accuracy when menu prices change.

5. **Inventory uses a transaction ledger** -- `inventory_stock` holds current quantity; `inventory_transaction` records every movement. Theoretical vs. actual comparison is a query joining `order_line_item` (via recipes) against `inventory_transaction`.

6. **Allergens and dietary tags are first-class entities** -- not JSONB arrays -- enabling reliable filtering, reporting, and FDA compliance auditing across the entire menu catalogue.

7. **Audit log follows CloudEvents structure** -- `event_type`, `event_source`, `entity_type`, `entity_id`, and `changes_json` map directly to CloudEvents fields, enabling webhook dispatch and event-driven integrations.

8. **Tax rates are temporal** -- `effective_from` / `effective_until` columns support jurisdiction-specific tax changes without breaking historical order records (which snapshot their own tax amounts).

9. **Reservation source tracking** -- the `source` field on reservations distinguishes direct bookings from OpenTable, Resy, Yelp, and phone, enabling attribution analytics and commission tracking.

10. **Daily summary as a materialised aggregate** -- rather than computing reporting metrics from raw orders every time, `daily_summary` is populated by a nightly job (or trigger), optimising dashboard query performance.
