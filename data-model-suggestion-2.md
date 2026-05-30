# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Restaurant Management Platform · Created: 2026-05-22

## Philosophy

This model treats every state change as an immutable event appended to an event store. The current state of any entity (an order, a table session, an inventory count) is derived by replaying its event stream. Read-optimised materialised views (projections) serve dashboard queries, POS screens, and kitchen displays, while the event store remains the single source of truth.

Event sourcing is a natural fit for restaurant operations because the domain is inherently event-driven: a guest arrives, a table is seated, courses are fired, items are modified, payments are split, tips are adjusted, the table is cleared. Regulations (PCI DSS, HACCP, GDPR) require detailed audit trails of who did what, when. With event sourcing, the audit trail is not a secondary concern bolted onto the data model -- it IS the data model. The open-source project `digital-restaurant` (Kotlin/Axon) demonstrates this pattern specifically for restaurant ordering with DDD aggregates and saga coordination.

This approach is best suited for teams building an AI-native platform where temporal queries ("what was the kitchen throughput at 7:15 PM last Friday?"), change pattern analysis ("which servers void the most items?"), and complete audit trails are core requirements rather than afterthoughts.

**Best for:** AI-driven analytics requiring full temporal history, regulatory environments demanding complete audit trails, and systems where "undo" and "replay" are first-class operations.

**Trade-offs:**
- Pro: Complete, immutable audit trail satisfies PCI DSS, HACCP, and GDPR requirements by design
- Pro: Temporal queries are trivial -- replay events to any point in time
- Pro: AI training data is rich -- every user action, kitchen timing, and state transition is captured
- Pro: New read models can be built retroactively from the event stream without data loss
- Pro: Natural fit for real-time event-driven integrations (kitchen displays, delivery platforms, webhooks)
- Con: Higher storage requirements -- events accumulate indefinitely (though append-only is cheap)
- Con: Eventual consistency between event store and read models requires careful design
- Con: More complex to implement than direct CRUD -- team must understand event sourcing patterns
- Con: Debugging requires event replay tooling rather than simple SELECT queries
- Con: Schema evolution of event payloads requires versioning strategy (upcasting)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | Every event in the store follows the CloudEvents envelope: `type`, `source`, `subject`, `time`, `data` |
| PCI DSS v4.0 | Payment events never contain raw card data; only tokenised references. Event immutability provides tamper-evident audit trail |
| HACCP / ISO 22000 | Temperature readings and food safety checks are events that cannot be retroactively altered |
| GDPR / CCPA | Guest data events support right-to-erasure via crypto-shredding (encrypting guest PII with per-guest keys, then destroying the key) |
| Schema.org | Read model projections for menus export as Schema.org JSON-LD, derived from menu events |
| FDA Allergen Guidance | Allergen change events create an auditable history of when allergen information was updated |
| ISO 3166 / ISO 4217 | Reference data (jurisdictions, currencies) stored in lookup tables; events reference codes |
| OpenAPI 3.1 | Read model APIs documented via OpenAPI; command endpoints documented separately |

---

## Event Store Core

```sql
-- ============================================================
-- EVENT STORE — single source of truth
-- ============================================================

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL,                  -- 'Order', 'TableSession', 'InventoryItem', 'Shift', 'Reservation', 'Guest'
    stream_id       UUID NOT NULL,                  -- aggregate root ID
    event_type      TEXT NOT NULL,                  -- e.g., 'OrderCreated', 'LineItemAdded', 'PaymentCaptured'
    event_version   INT NOT NULL,                   -- monotonically increasing per stream
    -- CloudEvents envelope fields
    ce_source       TEXT NOT NULL DEFAULT '/pos',   -- '/pos', '/api', '/kds', '/online', '/webhook'
    ce_subject      TEXT,                           -- human-readable subject (order number, table number)
    -- Payload
    data            JSONB NOT NULL,                 -- event-specific payload
    metadata        JSONB NOT NULL DEFAULT '{}',    -- actor_id, ip_address, device_id, correlation_id
    -- Tenant isolation
    organisation_id UUID NOT NULL,
    location_id     UUID,
    -- Timestamps
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Optimistic concurrency: unique per stream
    UNIQUE(stream_id, event_version)
);

-- Primary query pattern: replay events for a single aggregate
CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);

-- Query pattern: all events of a type across the system (for projections)
CREATE INDEX idx_event_type ON event_store(event_type, created_at);

-- Query pattern: all events for a location in time order (for location-level replay)
CREATE INDEX idx_event_location ON event_store(location_id, created_at);

-- Query pattern: all events for an organisation (for org-wide analytics)
CREATE INDEX idx_event_org ON event_store(organisation_id, created_at);

-- Partition by month for performance at scale
-- CREATE TABLE event_store PARTITION BY RANGE (created_at);
-- CREATE TABLE event_store_2026_05 PARTITION OF event_store
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');


-- ============================================================
-- EVENT SNAPSHOTS — performance optimisation
-- ============================================================

-- Snapshots avoid replaying thousands of events for long-lived aggregates
CREATE TABLE event_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    snapshot_version INT NOT NULL,                  -- event_version at snapshot time
    state           JSONB NOT NULL,                 -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

### Example Event Payloads

```sql
-- OrderCreated event
-- data: {
--   "order_number": "L1-20260522-0042",
--   "order_type": "dine_in",
--   "channel": "pos",
--   "table_session_id": "uuid...",
--   "server_id": "uuid...",
--   "guest_count": 4,
--   "currency_code": "USD"
-- }

-- LineItemAdded event
-- data: {
--   "line_item_id": "uuid...",
--   "menu_item_id": "uuid...",
--   "name": "Grilled Salmon",
--   "kitchen_name": "SALMON GRL",
--   "quantity": 1,
--   "unit_price": 28.00,
--   "course_number": 2,
--   "seat_number": 3,
--   "modifiers": [
--     {"modifier_id": "uuid...", "name": "Medium Rare", "price_adjustment": 0.00},
--     {"modifier_id": "uuid...", "name": "Sub Mashed Potato", "price_adjustment": 2.00}
--   ],
--   "notes": "No butter, dairy allergy"
-- }

-- CourseFired event
-- data: {
--   "course_number": 2,
--   "line_item_ids": ["uuid...", "uuid...", "uuid..."],
--   "station_assignments": {
--     "uuid-line-item-1": "grill",
--     "uuid-line-item-2": "saute",
--     "uuid-line-item-3": "cold"
--   },
--   "fired_by": "uuid-server..."
-- }

-- PaymentCaptured event
-- data: {
--   "payment_id": "uuid...",
--   "payment_method": "card",
--   "amount": 142.50,
--   "tip_amount": 28.50,
--   "gateway": "stripe",
--   "gateway_txn_id": "pi_3xyz...",
--   "card_brand": "visa",
--   "card_last_four": "4242"
-- }

-- TemperatureRecorded event (HACCP)
-- data: {
--   "equipment_name": "Walk-in Cooler #1",
--   "reading_celsius": 3.2,
--   "min_threshold": 0.0,
--   "max_threshold": 4.0,
--   "is_in_range": true,
--   "recorded_by": "uuid..."
-- }
```

---

## Reference Data Tables (Non-Event-Sourced)

Slowly-changing reference data that does not benefit from event sourcing uses traditional relational tables.

```sql
-- ============================================================
-- REFERENCE DATA — traditional CRUD tables
-- ============================================================

CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan_tier       TEXT NOT NULL DEFAULT 'standard',
    settings_json   JSONB NOT NULL DEFAULT '{}',
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
    settings_json   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, slug)
);

CREATE TABLE allergen (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            TEXT NOT NULL UNIQUE,
    name            TEXT NOT NULL,
    is_fda_major    BOOLEAN NOT NULL DEFAULT false
);

CREATE TABLE dietary_tag (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            TEXT NOT NULL UNIQUE,
    name            TEXT NOT NULL,
    schema_org_diet TEXT
);

CREATE TABLE tax_rate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    rate            NUMERIC(6,4) NOT NULL,
    jurisdiction    TEXT,
    applies_to      TEXT NOT NULL DEFAULT 'all',
    is_inclusive     BOOLEAN NOT NULL DEFAULT false,
    effective_from  DATE,
    effective_until DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE kitchen_station (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id     UUID NOT NULL REFERENCES location(id),
    name            TEXT NOT NULL,
    display_order   INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE supplier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    contact_name    TEXT,
    email           TEXT,
    phone           TEXT,
    payment_terms   TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Read Model Projections (Materialised Views)

These tables are rebuilt from the event store. They can be dropped and recreated at any time without data loss.

```sql
-- ============================================================
-- READ MODEL: Current Order State
-- ============================================================

CREATE TABLE rm_order (
    order_id        UUID PRIMARY KEY,               -- same as stream_id
    location_id     UUID NOT NULL,
    organisation_id UUID NOT NULL,
    order_number    TEXT NOT NULL,
    order_type      TEXT NOT NULL,
    channel         TEXT NOT NULL,
    status          TEXT NOT NULL,
    server_id       UUID,
    table_session_id UUID,
    guest_id        UUID,
    guest_count     INT,
    subtotal        NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    tax_total       NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    discount_total  NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    tip_total       NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    grand_total     NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    line_item_count INT NOT NULL DEFAULT 0,
    last_event_version INT NOT NULL,                -- tracks projection freshness
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_order_location ON rm_order(location_id, created_at);
CREATE INDEX idx_rm_order_status ON rm_order(location_id, status);

CREATE TABLE rm_order_line_item (
    line_item_id    UUID PRIMARY KEY,
    order_id        UUID NOT NULL REFERENCES rm_order(order_id),
    menu_item_id    UUID,
    name            TEXT NOT NULL,
    kitchen_name    TEXT,
    quantity        INT NOT NULL,
    unit_price      NUMERIC(10,2) NOT NULL,
    line_total      NUMERIC(10,2) NOT NULL,
    course_number   INT,
    seat_number     INT,
    status          TEXT NOT NULL,
    modifiers_json  JSONB,
    notes           TEXT,
    fired_at        TIMESTAMPTZ,
    ready_at        TIMESTAMPTZ,
    served_at       TIMESTAMPTZ
);

CREATE INDEX idx_rm_line_item_order ON rm_order_line_item(order_id);

-- ============================================================
-- READ MODEL: Kitchen Display
-- ============================================================

CREATE TABLE rm_kitchen_ticket (
    ticket_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL,
    station_id      UUID NOT NULL,
    location_id     UUID NOT NULL,
    order_number    TEXT NOT NULL,
    order_type      TEXT NOT NULL,
    table_number    TEXT,
    server_name     TEXT,
    course_number   INT,
    status          TEXT NOT NULL,                  -- pending, in_progress, ready, served
    items_json      JSONB NOT NULL,                 -- [{name, kitchen_name, qty, mods, notes, seat}]
    priority        INT NOT NULL DEFAULT 0,         -- higher = more urgent
    fired_at        TIMESTAMPTZ,
    target_time     TIMESTAMPTZ,                   -- expected completion
    completed_at    TIMESTAMPTZ,
    elapsed_seconds INT                            -- for speed-of-service reporting
);

CREATE INDEX idx_rm_kitchen_station ON rm_kitchen_ticket(station_id, status);
CREATE INDEX idx_rm_kitchen_location ON rm_kitchen_ticket(location_id, fired_at);

-- ============================================================
-- READ MODEL: Table Status
-- ============================================================

CREATE TABLE rm_table_status (
    table_id        UUID PRIMARY KEY,
    location_id     UUID NOT NULL,
    floor_plan_id   UUID NOT NULL,
    table_number    TEXT NOT NULL,
    capacity        INT NOT NULL,
    status          TEXT NOT NULL,                  -- available, reserved, seated, check_presented, cleaning
    current_session_id UUID,
    server_id       UUID,
    server_name     TEXT,
    covers          INT,
    seated_at       TIMESTAMPTZ,
    elapsed_minutes INT,                           -- for colour-coded timer display
    active_order_id UUID,
    order_total     NUMERIC(12,2)
);

CREATE INDEX idx_rm_table_location ON rm_table_status(location_id, status);

-- ============================================================
-- READ MODEL: Inventory Levels
-- ============================================================

CREATE TABLE rm_inventory_level (
    location_id     UUID NOT NULL,
    ingredient_id   UUID NOT NULL,
    ingredient_name TEXT NOT NULL,
    category        TEXT,
    quantity_on_hand NUMERIC(12,4) NOT NULL,
    unit_of_measure TEXT NOT NULL,
    par_level       NUMERIC(10,2),
    is_below_par    BOOLEAN NOT NULL DEFAULT false,
    last_received_at TIMESTAMPTZ,
    last_counted_at TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (location_id, ingredient_id)
);

CREATE INDEX idx_rm_inv_below_par ON rm_inventory_level(location_id, is_below_par)
    WHERE is_below_par = true;

-- ============================================================
-- READ MODEL: Menu Catalogue (for POS and online ordering)
-- ============================================================

CREATE TABLE rm_menu_catalogue (
    menu_item_id    UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    name            TEXT NOT NULL,
    kitchen_name    TEXT,
    description     TEXT,
    base_price      NUMERIC(10,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    calories        INT,
    prep_time_mins  INT,
    is_active       BOOLEAN NOT NULL,
    is_available    BOOLEAN NOT NULL DEFAULT true,  -- false when key ingredient is out of stock
    allergens_json  JSONB,                          -- ["milk", "eggs"]
    dietary_tags_json JSONB,                        -- ["vegetarian", "gluten_free"]
    modifiers_json  JSONB,                          -- modifier group summaries
    image_url       TEXT,
    menus_json      JSONB,                          -- which menus/sections this item appears in
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_menu_org ON rm_menu_catalogue(organisation_id, is_active);

-- ============================================================
-- READ MODEL: Guest Profile
-- ============================================================

CREATE TABLE rm_guest_profile (
    guest_id        UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    display_name    TEXT,
    email           TEXT,
    phone           TEXT,
    total_visits    INT NOT NULL DEFAULT 0,
    total_spend     NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    avg_check       NUMERIC(10,2),
    last_visit_at   TIMESTAMPTZ,
    favourite_items_json JSONB,                     -- [{menu_item_id, name, order_count}]
    allergens_json  JSONB,
    notes           TEXT,
    is_vip          BOOLEAN NOT NULL DEFAULT false,
    loyalty_points  INT NOT NULL DEFAULT 0,
    loyalty_tier    TEXT,
    consent_status  TEXT NOT NULL DEFAULT 'none',
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_guest_org ON rm_guest_profile(organisation_id);

-- ============================================================
-- READ MODEL: Daily Summary (aggregated from events)
-- ============================================================

CREATE TABLE rm_daily_summary (
    location_id     UUID NOT NULL,
    business_date   DATE NOT NULL,
    gross_sales     NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    net_sales       NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    tax_collected   NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    tips_collected  NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    discounts_given NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    order_count     INT NOT NULL DEFAULT 0,
    guest_count     INT NOT NULL DEFAULT 0,
    avg_check       NUMERIC(10,2),
    avg_ticket_time_seconds INT,                   -- kitchen performance
    labour_hours    NUMERIC(10,2),
    labour_cost     NUMERIC(12,2),
    updated_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (location_id, business_date)
);

-- ============================================================
-- READ MODEL: Staff Schedule
-- ============================================================

CREATE TABLE rm_staff_schedule (
    shift_id        UUID PRIMARY KEY,
    staff_member_id UUID NOT NULL,
    location_id     UUID NOT NULL,
    staff_name      TEXT NOT NULL,
    role_name       TEXT,
    start_time      TIMESTAMPTZ NOT NULL,
    end_time        TIMESTAMPTZ NOT NULL,
    break_minutes   INT NOT NULL DEFAULT 0,
    status          TEXT NOT NULL,
    clock_in        TIMESTAMPTZ,
    clock_out       TIMESTAMPTZ,
    actual_minutes  INT,
    hourly_rate     NUMERIC(10,2),
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_schedule_location ON rm_staff_schedule(location_id, start_time);
CREATE INDEX idx_rm_schedule_staff ON rm_staff_schedule(staff_member_id, start_time);
```

---

## Event Catalogue

The following event types define the complete domain event vocabulary:

```sql
-- ============================================================
-- EVENT TYPE REGISTRY (documentation / validation)
-- ============================================================

CREATE TABLE event_type_registry (
    event_type      TEXT PRIMARY KEY,
    stream_type     TEXT NOT NULL,
    description     TEXT NOT NULL,
    schema_version  INT NOT NULL DEFAULT 1,
    payload_schema  JSONB,                          -- JSON Schema for validation
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Seed the registry
INSERT INTO event_type_registry (event_type, stream_type, description) VALUES
-- Order lifecycle
('OrderCreated',           'Order',          'New order opened on POS, online, or delivery channel'),
('LineItemAdded',          'Order',          'Menu item added to order with modifiers and notes'),
('LineItemModified',       'Order',          'Line item quantity, modifiers, or notes changed'),
('LineItemVoided',         'Order',          'Line item removed from order with reason'),
('CourseFired',            'Order',          'Course sent to kitchen stations'),
('LineItemReady',          'Order',          'Kitchen station marks item as ready'),
('LineItemServed',         'Order',          'Item delivered to guest'),
('DiscountApplied',        'Order',          'Discount applied to order or line item'),
('DiscountRemoved',        'Order',          'Discount removed from order'),
('ServiceChargeAdded',     'Order',          'Service charge or gratuity added'),
('OrderSplit',             'Order',          'Order split into multiple checks'),
('OrderMerged',            'Order',          'Multiple orders merged into one'),
('PaymentAuthorized',      'Order',          'Payment card authorized'),
('PaymentCaptured',        'Order',          'Payment captured / cash tendered'),
('PaymentRefunded',        'Order',          'Payment partially or fully refunded'),
('TipAdjusted',            'Order',          'Tip amount modified after initial payment'),
('OrderClosed',            'Order',          'Order fully paid and closed'),
('OrderVoided',            'Order',          'Entire order voided by manager'),
('OrderCancelled',         'Order',          'Order cancelled before completion'),
-- Table session lifecycle
('TableSeated',            'TableSession',   'Party seated at table'),
('TableServerAssigned',    'TableSession',   'Server assigned to table'),
('TableStatusChanged',     'TableSession',   'Table status updated (ordering, served, etc.)'),
('TableCleared',           'TableSession',   'Table cleared and available'),
('TablesCombined',         'TableSession',   'Multiple tables joined for large party'),
('TablesSeparated',        'TableSession',   'Combined tables split back'),
-- Reservation lifecycle
('ReservationCreated',     'Reservation',    'New reservation booked'),
('ReservationConfirmed',   'Reservation',    'Reservation confirmed by guest or auto'),
('ReservationModified',    'Reservation',    'Reservation details changed (time, party size)'),
('ReservationCancelled',   'Reservation',    'Reservation cancelled'),
('ReservationSeated',      'Reservation',    'Reservation party has been seated'),
('ReservationNoShow',      'Reservation',    'Guest did not arrive'),
-- Inventory lifecycle
('InventoryReceived',      'Inventory',      'Stock received from supplier'),
('InventoryUsed',          'Inventory',      'Stock consumed (linked to order)'),
('InventoryWasted',        'Inventory',      'Stock wasted or spoiled'),
('InventoryTransferred',   'Inventory',      'Stock transferred between locations'),
('InventoryCounted',       'Inventory',      'Physical count recorded'),
('InventoryAdjusted',      'Inventory',      'Manual adjustment with reason'),
('PurchaseOrderCreated',   'Inventory',      'New purchase order drafted'),
('PurchaseOrderSubmitted', 'Inventory',      'PO sent to supplier'),
('PurchaseOrderReceived',  'Inventory',      'PO marked as received'),
-- Staff lifecycle
('ShiftScheduled',         'Shift',          'New shift added to schedule'),
('ShiftModified',          'Shift',          'Shift time or role changed'),
('ShiftCancelled',         'Shift',          'Shift removed from schedule'),
('StaffClockedIn',         'Shift',          'Employee clocked in'),
('StaffClockedOut',        'Shift',          'Employee clocked out'),
('BreakStarted',           'Shift',          'Employee started break'),
('BreakEnded',             'Shift',          'Employee ended break'),
-- Guest lifecycle
('GuestCreated',           'Guest',          'New guest profile created'),
('GuestUpdated',           'Guest',          'Guest profile information changed'),
('GuestConsentChanged',    'Guest',          'GDPR consent status changed'),
('GuestDataErased',        'Guest',          'Guest data erased per GDPR request (crypto-shredding)'),
('LoyaltyPointsEarned',   'Guest',          'Loyalty points credited from order'),
('LoyaltyPointsRedeemed', 'Guest',          'Loyalty points redeemed for reward'),
-- Menu lifecycle
('MenuItemCreated',        'Menu',           'New menu item added to catalogue'),
('MenuItemUpdated',        'Menu',           'Menu item details changed'),
('MenuItemPriceChanged',   'Menu',           'Menu item price updated'),
('MenuItemDeactivated',    'Menu',           'Menu item marked as inactive'),
('AllergenUpdated',        'Menu',           'Menu item allergen information changed'),
('MenuPublished',          'Menu',           'Menu version published to locations'),
-- Food safety
('TemperatureRecorded',    'FoodSafety',     'Equipment temperature reading logged'),
('SafetyCheckCompleted',   'FoodSafety',     'Food safety checklist completed'),
('SafetyIssueRaised',      'FoodSafety',     'Food safety issue flagged with corrective action');
```

---

## Command Handlers (Application Layer)

Commands are the write side of CQRS. Each command validates business rules and emits events.

```sql
-- Example: the application layer validates and emits events.
-- This is pseudocode showing the command -> event flow.

-- Command: CreateOrder {location_id, order_type, server_id, table_session_id}
-- Validates: location exists, server is clocked in, table is seated
-- Emits: OrderCreated

-- Command: AddLineItem {order_id, menu_item_id, quantity, modifiers[], notes}
-- Validates: order is open, menu item is active and available, modifiers are valid
-- Emits: LineItemAdded

-- Command: FireCourse {order_id, course_number}
-- Validates: order is open, course items exist, kitchen stations are configured
-- Emits: CourseFired (one per station)

-- Command: CapturePayment {order_id, payment_method, amount, tip, gateway_token}
-- Validates: order total matches, no double payment
-- Emits: PaymentCaptured

-- Command: RecordTemperature {location_id, equipment, reading, recorded_by}
-- Validates: location exists, equipment is registered
-- Emits: TemperatureRecorded (with auto-computed is_in_range)
```

---

## Projection Rebuilder

```sql
-- Example: rebuilding the rm_order read model from events

-- Step 1: Clear the projection
-- TRUNCATE rm_order, rm_order_line_item;

-- Step 2: Replay events in order
-- SELECT * FROM event_store
-- WHERE stream_type = 'Order'
-- ORDER BY stream_id, event_version;

-- Step 3: For each event, apply the state change
-- OrderCreated -> INSERT INTO rm_order
-- LineItemAdded -> INSERT INTO rm_order_line_item, UPDATE rm_order totals
-- PaymentCaptured -> UPDATE rm_order payment totals
-- OrderClosed -> UPDATE rm_order SET status = 'closed'

-- In production, this runs as a background worker that processes events
-- from a "last processed event_id" cursor, enabling incremental projection updates.

-- ============================================================
-- PROJECTION CHECKPOINT — tracks rebuild progress
-- ============================================================

CREATE TABLE projection_checkpoint (
    projection_name TEXT PRIMARY KEY,               -- 'rm_order', 'rm_kitchen_ticket', etc.
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    lag_seconds     INT,                           -- how far behind real-time
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Temporal Query Examples

```sql
-- "What was this order's state at 7:15 PM?"
-- Replay events up to that timestamp:
SELECT * FROM event_store
WHERE stream_id = '<<order-uuid>>'
  AND created_at <= '2026-05-22 19:15:00-04'
ORDER BY event_version;

-- "Which servers void the most items in the last 30 days?"
SELECT 
    metadata->>'actor_id' AS server_id,
    COUNT(*) AS void_count,
    SUM((data->>'unit_price')::NUMERIC * (data->>'quantity')::INT) AS void_value
FROM event_store
WHERE event_type = 'LineItemVoided'
  AND created_at >= now() - INTERVAL '30 days'
  AND organisation_id = '<<org-uuid>>'
GROUP BY metadata->>'actor_id'
ORDER BY void_count DESC;

-- "Average time from CourseFired to LineItemReady per station, last week"
WITH fired AS (
    SELECT stream_id, 
           (data->>'course_number')::INT AS course,
           created_at AS fired_at
    FROM event_store
    WHERE event_type = 'CourseFired'
      AND created_at >= now() - INTERVAL '7 days'
),
ready AS (
    SELECT stream_id,
           data->>'line_item_id' AS item_id,
           created_at AS ready_at
    FROM event_store
    WHERE event_type = 'LineItemReady'
      AND created_at >= now() - INTERVAL '7 days'
)
SELECT 
    AVG(EXTRACT(EPOCH FROM ready.ready_at - fired.fired_at)) AS avg_seconds
FROM fired
JOIN ready ON fired.stream_id = ready.stream_id
WHERE ready.ready_at > fired.fired_at;

-- "Show me the complete audit trail for order #42"
SELECT event_type, ce_source, metadata->>'actor_id' AS actor,
       data, created_at
FROM event_store
WHERE stream_id = '<<order-uuid>>'
ORDER BY event_version;
```

---

## GDPR Crypto-Shredding Pattern

```sql
-- ============================================================
-- GDPR: CRYPTO-SHREDDING SUPPORT
-- ============================================================

-- Each guest's PII in events is encrypted with a per-guest key.
-- To exercise right-to-erasure, destroy the key — events become unreadable.

CREATE TABLE guest_encryption_key (
    guest_id        UUID PRIMARY KEY,
    encryption_key  BYTEA NOT NULL,                -- AES-256 key, encrypted at rest
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    destroyed_at    TIMESTAMPTZ                    -- set when GDPR erasure requested
);

-- When a guest requests erasure:
-- 1. UPDATE guest_encryption_key SET encryption_key = NULL, destroyed_at = now()
--    WHERE guest_id = '<<guest-uuid>>';
-- 2. The GuestDataErased event is emitted (without PII)
-- 3. Read model projections clear the guest's PII
-- 4. Events referencing the guest remain but PII fields are now undecryptable
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | event_store, event_snapshot, projection_checkpoint |
| Event Registry | 1 | event_type_registry (documentation/validation) |
| Reference Data | 7 | organisation, location, allergen, dietary_tag, tax_rate, kitchen_station, supplier |
| Read Model: Orders | 2 | rm_order, rm_order_line_item |
| Read Model: Kitchen | 1 | rm_kitchen_ticket |
| Read Model: Tables | 1 | rm_table_status |
| Read Model: Inventory | 1 | rm_inventory_level |
| Read Model: Menu | 1 | rm_menu_catalogue |
| Read Model: Guests | 1 | rm_guest_profile |
| Read Model: Reporting | 1 | rm_daily_summary |
| Read Model: Staff | 1 | rm_staff_schedule |
| GDPR Support | 1 | guest_encryption_key |
| **Total** | **21** | Plus ~60 event types in the registry |

---

## Key Design Decisions

1. **Single event_store table with stream_type discrimination** — rather than separate event tables per aggregate, all events go into one partitioned table. This simplifies cross-aggregate projections and global event subscriptions.

2. **CloudEvents envelope built into every event** — `ce_source`, `ce_subject`, and structured `metadata` make every event webhook-ready and compatible with the CloudEvents specification for external dispatch.

3. **Optimistic concurrency via (stream_id, event_version) unique constraint** — prevents conflicting writes to the same aggregate, critical for concurrent POS terminal operations.

4. **Read models are disposable** — all `rm_*` tables can be dropped and rebuilt from the event store at any time. This means read models can be optimised per use case without fear of data loss.

5. **Separate read model for kitchen display** — `rm_kitchen_ticket` is a highly specialised projection optimised for the KDS screen: one row per station per course, with denormalised item details. It updates within milliseconds of CourseFired events.

6. **Event snapshots for long-lived aggregates** — guest profiles and inventory items accumulate thousands of events over months. Snapshots at configurable intervals (every 100 events) prevent slow replay.

7. **GDPR crypto-shredding instead of event deletion** — immutability is preserved while still supporting right-to-erasure. PII in event payloads is encrypted with per-guest keys; destroying the key makes the data unrecoverable.

8. **Time-partitioned event store for scale** — at high-volume locations processing 500+ orders/day, each generating 10-20 events, the event store grows by 10,000+ rows daily per location. Monthly partitioning keeps queries fast and enables cost-effective cold storage of old partitions.

9. **Event type registry as living documentation** — the registry table serves as both documentation and validation reference. Projection builders can query it to discover which events they should subscribe to.

10. **Projection checkpoint enables exactly-once processing** — each projection tracks its last processed event, enabling incremental updates and crash recovery without replaying the entire event history.
