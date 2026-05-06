# Standards & API Reference

> Project: Restaurant Management Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 22000:2018 — Food Safety Management Systems**
- URL: https://www.iso.org/iso-22000-food-safety-management.html
- The primary international food safety standard applicable to restaurants and food service establishments. Defines requirements for a food safety management system covering HACCP principles, prerequisite programs, and continual improvement. Restaurant management platforms increasingly embed digital checklists and audit trails to support ISO 22000 compliance.

**ISO 22002:2025 — Prerequisite Programmes on Food Safety (revised series)**
- URL: https://www.sgs.com/en-sa/news/2026/01/iso-22002-2025-explained-a-new-framework-for-sector-specific-food-safety
- A completely revised series published July 2025 that introduces a new common foundation (Part 100) and sector-specific parts. Addresses globalized supply chains, food fraud, food defense, and sustainability. Relevant to supplier management and traceability modules in restaurant management platforms.

**ISO 9001:2015 — Quality Management Systems**
- URL: https://www.iso.org/standard/62085.html
- Widely adopted quality management framework applicable to restaurant chains seeking to standardise operations across multiple locations. Relevant to SOPs, audit trails, and operational consistency modules.

**ISO 14001:2015 — Environmental Management Systems**
- URL: https://www.iso.org/standard/60857.html
- Increasingly adopted by restaurants with sustainability goals. Covers waste management, energy consumption, and resource tracking — features increasingly expected in modern restaurant management platforms.

**ISO 45001:2018 — Occupational Health and Safety**
- URL: https://www.iso.org/standard/63787.html
- Applicable to staff safety management, incident logging, and workplace risk assessments within restaurant workforce management modules.

---

### W3C & IETF Standards

**Schema.org Restaurant / Menu / MenuItem Vocabulary (W3C Community Group)**
- URL: https://schema.org/Restaurant · https://schema.org/Menu · https://schema.org/MenuItem
- De facto structured data standard for restaurant web presence. JSON-LD implementation (recommended by Google) allows restaurant management platforms to expose menus, hours, location, cuisine type, and nutrition information to search engines. NutritionInformation type supports allergen and dietary requirement markup.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Standard token format used across restaurant SaaS APIs for authentication and authorisation, including multi-tenant access control between corporate and franchise-level accounts.

**RFC 7617 — The 'Basic' HTTP Authentication Scheme**
- URL: https://datatracker.ietf.org/doc/html/rfc7617
- Baseline HTTP authentication referenced in numerous restaurant POS API integrations.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The authorisation standard underlying SSO and third-party integrations (delivery platforms, loyalty programmes, accounting systems) across all major restaurant management platforms.

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Foundational REST API standard governing HTTP method usage (GET, POST, PUT, DELETE, PATCH) across restaurant POS and management APIs.

---

### Data Model & API Specifications

**OpenAPI Specification 3.x**
- URL: https://spec.openapis.org/oas/latest.html
- Industry standard for describing RESTful APIs. Major restaurant platforms (Toast, Square, Lightspeed) publish OpenAPI-compatible documentation for their developer portals. An AI-native restaurant management platform should publish an OpenAPI 3.1 schema as its primary API contract.

**JSON Schema (draft-07 / 2020-12)**
- URL: https://json-schema.org/
- Standard for validating JSON data structures. Applicable to menu data models, order payloads, reservation objects, and staff scheduling data exchanged between services.

**Webhooks / CloudEvents Specification 1.0**
- URL: https://cloudevents.io/
- Standard envelope format for event-driven architecture. Relevant for order status updates, inventory alerts, and reservation notifications that restaurant management platforms must emit to integrated systems.

**GraphQL Specification**
- URL: https://spec.graphql.org/
- Increasingly adopted as an alternative to REST for flexible data querying in restaurant analytics dashboards. Relevant for multi-outlet reporting where clients request only the fields they need.

---

### Security & Authentication Standards

**PCI DSS v4.0 (Payment Card Industry Data Security Standard)**
- URL: https://www.pcisecuritystandards.org/standards/
- Mandatory for any restaurant management platform that stores, processes, or transmits payment card data. PCI DSS 4.0 (fully enforced from March 2025) requires stronger encryption, multi-factor authentication for all staff accessing payment systems, and quarterly vulnerability scans. Four compliance levels based on annual transaction volume. Applies to POS integrations, online ordering, and gift card systems.

**EMV (Europay, Mastercard, Visa) — Chip & Contactless Standards**
- URL: https://www.emvco.com/
- Hardware-level payment standard mandatory at physical POS terminals. Restaurant management platforms must support EMV-compliant payment terminal integrations.

**OAuth 2.0 + OpenID Connect (OIDC)**
- URL: https://openid.net/connect/
- Standard identity and authorisation layer for multi-tenant restaurant SaaS. Enables SSO across corporate portals, franchise dashboards, and staff-facing mobile apps. Token isolation per tenant is required for franchise multi-location deployments.

**OWASP Application Security Verification Standard (ASVS)**
- URL: https://owasp.org/www-project-application-security-verification-standard/
- Framework for secure web application development. Particularly relevant for online ordering portals, customer loyalty data handling, and admin dashboards in restaurant management platforms.

**NIST Cybersecurity Framework (CSF) 2.0**
- URL: https://www.nist.gov/cyberframework
- Widely adopted risk management framework applicable to enterprise restaurant operators and franchisors who must demonstrate security governance to partners and insurers.

---

### Food Safety & Regulatory Standards

**HACCP (Hazard Analysis and Critical Control Points) — FDA/USDA**
- URL: https://www.fda.gov/food/hazard-analysis-critical-control-point-haccp/haccp-principles-application-guidelines
- Foundational food safety framework requiring identification of biological, chemical, and physical hazards with critical control points. Digital restaurant management platforms support HACCP compliance via timestamped temperature logs, cleaning schedule checklists, and audit trail exports. Compliance standards include FSMA, BRC, GMP, SQF, GFSI, and ISO 22000.

**FDA Food Safety Modernization Act (FSMA)**
- URL: https://www.fda.gov/food/food-safety-modernization-act-fsma
- US federal law governing food safety from supply chain to service. Relevant to supplier traceability, allergen management, and menu labelling compliance modules.

**FDA Menu Labelling Requirements (Section 4205 of the ACA)**
- URL: https://foodlabelmaker.com/regulatory-hub/fda/fda-menu-labelling-a-compliance-guide-for-the-fb-industry/
- Requires chain restaurants with 20+ locations to display calorie information on menus and menu boards. Digital menus must include a statement about suggested caloric intake and additional nutritional information on request.

**FDA Food Allergen Labelling Guidance (Edition 5, January 2025)**
- URL: https://www.fda.gov/food/nutrition-food-labeling-and-critical-foods/food-allergies
- Updated January 2025, this guidance covers the nine major food allergens (milk, eggs, fish, crustacean shellfish, tree nuts, peanuts, wheat, soybeans, sesame) with revised definitions. Restaurant management platforms must support allergen flagging per menu item with structured data export.

**GDPR (General Data Protection Regulation) — EU Regulation 2016/679**
- URL: https://gdpr.eu/
- Applies to any restaurant management platform processing EU customer data (loyalty programmes, reservations, online ordering). Requires explicit consent, data minimisation, right to erasure, and third-party supplier compliance. Fines up to 4% of annual global revenue or €20 million.

**CCPA (California Consumer Privacy Act)**
- URL: https://oag.ca.gov/privacy/ccpa
- US state-level data privacy law applicable to restaurant loyalty programmes and customer data collected via online ordering or CRM modules.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic**
- URL: https://modelcontextprotocol.io/
- An open standard enabling AI assistants to interact with external data sources and tools. An AI-native restaurant management platform could expose MCP-compatible server endpoints for AI agents to query inventory levels, generate staff schedules, analyse sales trends, or respond to natural-language operations queries. Highly relevant for the AI-augmentation layer of this project.

---

## Similar Products — Developer Documentation & APIs

### Toast POS
- **Description:** Purpose-built restaurant POS and management platform with ordering, kitchen display, payroll, and analytics modules.
- **API Documentation:** https://doc.toasttab.com/doc/devguide/apiOverview.html
- **API Reference:** https://toastintegrations.redoc.ly/
- **Developer Guide:** https://doc.toasttab.com/doc/devguide/index.html
- **SDKs/Libraries:** REST-based; no official SDK; integrators use standard HTTP clients
- **Standards:** REST/JSON, OAuth 2.0, Webhooks
- **Authentication:** OAuth 2.0 (client credentials); Standard API access provides read-only credentials configurable per restaurant/store
- **Notes:** Supports both read and write operations. Simphony Transaction Services Gen2 and Business Intelligence API provide near real-time transactional data. 250+ certified third-party integrations available.

### Square for Restaurants
- **Description:** Restaurant POS with table management, online ordering, and kitchen display built on Square's developer platform. Accessible pricing with free tier.
- **API Documentation:** https://developer.squareup.com/docs
- **API Reference:** https://developer.squareup.com/reference/square
- **SDKs/Libraries:** Python, Node.js, Ruby, PHP, Java, .NET — https://developer.squareup.com/docs/sdks
- **Developer Guide:** https://developer.squareup.com/us/en
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, Webhooks
- **Authentication:** OAuth 2.0; supports API keys for simple integrations
- **Notes:** Rewritten Python and Node.js SDKs released 2025. Supports orders, catalog, inventory, and reservation management through unified API.

### Lightspeed Restaurant (K-Series)
- **Description:** Cloud POS targeting full-service restaurants with strong analytics and multi-outlet reporting capabilities.
- **API Documentation:** https://api-docs.lsk.lightspeed.app/
- **Developer Portal:** https://api-portal.lsk.lightspeed.app/
- **SDKs/Libraries:** REST-based; standard HTTP clients; community Ruby gem available on GitHub
- **Developer Guide:** https://api-portal.lsk.lightspeed.app/quick-start/intro
- **Standards:** REST/JSON, OAuth 2.0
- **Authentication:** OAuth 2.0; access restricted to Lightspeed partners and approved merchants; scopes configured per API client
- **Notes:** APIs in continuous development. O-Series also has separate API documentation at Lightspeed support portal. K-Series and X-Series (retail) have separate API portals.

### OpenTable
- **Description:** Leading restaurant reservation and guest management platform with 60,000+ restaurant partners globally.
- **API Documentation:** https://docs.opentable.com/
- **Partner API Page:** https://www.opentable.com/restaurant-solutions/api-partners/
- **SDKs/Libraries:** REST-based; no public SDK
- **Standards:** REST/JSON; API Sandbox available for partner testing
- **Authentication:** Affiliate programme required; not an open API — access requires application
- **Notes:** API exposes restaurant name, address, coordinates, phone number, reservation links, and real-time availability. Sandbox environment available for development testing.

### Uber Eats Marketplace API
- **Description:** Delivery marketplace API enabling programmatic management of stores, menus, and orders on the Uber Eats platform.
- **API Documentation:** https://developer.uber.com/docs/eats/introduction
- **Developer Guide:** https://developer.uber.com/docs/eats/guides/getting-started
- **Standards:** REST/JSON, OAuth 2.0, Webhooks
- **Authentication:** OAuth 2.0
- **Notes:** Supports store management (online/offline status, hours), menu synchronisation (items, pricing, availability), and order processing (receive, accept, fulfil). Requires formal partner approval.

### DoorDash Drive API
- **Description:** DoorDash's delivery fulfilment API for requesting deliveries from DoorDash's Dasher fleet, distinct from the merchant marketplace API.
- **API Documentation:** https://developer.doordash.com/en-US/docs/drive/tutorials/get_started/
- **Standards:** REST/JSON, JWT authentication
- **Authentication:** JWT-based; Sandbox environment available without incurring real costs
- **Notes:** Focused on last-mile delivery fulfilment. Separate from DoorDash Merchant API for storefront management.

### 7shifts (Staff Scheduling)
- **Description:** Restaurant-specific employee scheduling, labour management, and payroll platform used by 50,000+ restaurants.
- **API Documentation:** https://developers.7shifts.com/reference/introduction
- **Standards:** REST/JSON, OAuth 2.0, Webhooks
- **Authentication:** OAuth 2.0 (v2 API)
- **Notes:** Webhooks support real-time notifications for shift changes, time-off requests, and schedule updates. Integrates natively with Toast, Square, and Gusto for payroll sync.

### Deliverect
- **Description:** Integration middleware connecting restaurant POS systems with 1000+ delivery and online ordering platforms through a single API.
- **API Documentation:** https://developers.deliverect.com/
- **Postman Collection:** https://www.postman.com/deliverect/api-team/documentation/edifiys/deliverect-public-api
- **Standards:** REST/JSON, Webhooks
- **Authentication:** API Key + OAuth 2.0
- **Notes:** Syncs menus, pricing, store status, and orders across all channels in real-time. Acts as aggregation middleware — particularly relevant for multi-channel restaurant management architectures.

### Xero Accounting API
- **Description:** Cloud accounting platform widely integrated with restaurant POS systems for financial data synchronisation and reporting.
- **API Documentation:** https://developer.xero.com/documentation/api/accounting/overview
- **SDKs/Libraries:** .NET, Java, Node.js, PHP, Python, Ruby — via Xero developer portal
- **Standards:** REST/JSON, OAuth 2.0, OpenAPI 3.0
- **Authentication:** OAuth 2.0 (PKCE flow for web apps)
- **Notes:** Exposes chart of accounts, invoices, payments, journal entries, bank transactions, tax rates, and contacts. Essential integration target for restaurant financial reporting modules.

### Stripe Payments API
- **Description:** Full-stack payment processing API supporting card present (Terminal), online ordering, and subscription billing relevant to restaurant SaaS platforms.
- **API Documentation:** https://docs.stripe.com/api
- **SDKs/Libraries:** Node.js, Python, Ruby, PHP, Java, Go, .NET — https://docs.stripe.com/sdks
- **Developer Guide:** https://docs.stripe.com/development
- **Standards:** REST/JSON, OpenAPI, Webhooks, TLS 1.2+
- **Authentication:** API keys (publishable + secret); Restricted keys for scoped access
- **Notes:** Supports PaymentIntents, Terminal (card-present), Connect (multi-party payments for franchise models), and webhook events for async order/payment state tracking. PCI DSS SAQ A compliant when using Stripe-hosted elements.

---

## Notes

**Emerging Standards & Gaps**

- **Unified Restaurant Data Model**: There is no single open industry standard for restaurant data interchange (menu items, orders, reservations, staff shifts). Each major platform uses proprietary schemas, creating significant integration overhead. An AI-native platform that publishes an open, schema.org-compatible JSON-LD data model could become a de facto standard.

- **KDS Integration**: Kitchen Display System (KDS) integration lacks formal standardisation beyond UnifiedPOS (OMG UPOS 1.15 — https://www.omg.org/spec/UPOS/1.15/About-UPOS). Most KDS integrations rely on proprietary middleware or POS vendor-specific protocols.

- **AI / MCP Layer**: The Model Context Protocol is nascent but highly relevant. Building MCP-compatible tool endpoints for inventory queries, labour forecasting, and menu optimisation would differentiate an AI-native platform and integrate with emerging AI agent frameworks.

- **Fiscal / Tax Compliance**: Country-specific fiscal requirements (e.g., Italy's RT Registers, France's NF 525 anti-fraud law, Germany's KassenSichV) are not covered by international standards and require localised compliance modules — a known gap in most international restaurant platforms.

- **Yelp Reservations API**: https://docs.developer.yelp.com/docs/reservation — supports finding businesses with reservation availability, placing holds, and completing bookings. Relevant for reservation aggregation alongside OpenTable.
