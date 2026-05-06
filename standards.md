# Standards & API Reference

> Project: Church Management System (ChMS) · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 27001:2022 — Information Security Management Systems** — https://www.iso.org/standard/27001 — Baseline for protecting member data, pastoral notes, and donor records; expected by larger churches and denominations procuring software.
- **ISO/IEC 27701:2019 — Privacy Information Management** — https://www.iso.org/standard/71670.html — Extension to 27001 covering PII processing; relevant given the sensitivity of pastoral and family data.
- **ISO 20022 — Universal Financial Industry Message Scheme** — https://www.iso20022.org/ — Standard for payment-related messaging; relevant for ACH and bank-rail giving integrations.
- **ISO 4217 — Currency Codes** — https://www.iso.org/iso-4217-currency-codes.html — Required for multi-currency giving and reporting.
- **ISO 8601 — Date and Time Format** — https://www.iso.org/iso-8601-date-and-time-format.html — Calendars, events, and check-in timestamps across time zones.
- **ISO 3166 — Country Codes** — https://www.iso.org/iso-3166-country-codes.html — Member addresses, multi-region deployments.
- **ISO/IEC 18004 — QR Code** — https://www.iso.org/standard/62021.html — Check-in kiosks, text-to-give links, and event tickets.

### W3C & IETF Standards

- **RFC 9110/9111/9112 — HTTP Semantics, Caching, and HTTP/1.1** — https://www.rfc-editor.org/rfc/rfc9110 — Foundation for any REST API.
- **RFC 6749 — OAuth 2.0** — https://www.rfc-editor.org/rfc/rfc6749 — Third-party app integrations and member sign-in.
- **RFC 7519 — JSON Web Token (JWT)** — https://www.rfc-editor.org/rfc/rfc7519 — Session and API authentication.
- **RFC 5545 — iCalendar** — https://www.rfc-editor.org/rfc/rfc5545 — Event publishing and external calendar subscription.
- **RFC 6350 — vCard 4.0** — https://www.rfc-editor.org/rfc/rfc6350 — Contact export/import for member directory.
- **RFC 5321 / 5322 — SMTP and Internet Message Format** — https://www.rfc-editor.org/rfc/rfc5322 — Bulk and transactional email.
- **RFC 8058 — One-Click Unsubscribe** — https://www.rfc-editor.org/rfc/rfc8058 — Required for compliant bulk email per Gmail/Yahoo 2024 sender rules.
- **RFC 8617 — ARC (Authenticated Received Chain)** — https://www.rfc-editor.org/rfc/rfc8617 — Improves email deliverability for forwarded messages.
- **W3C WCAG 2.2** — https://www.w3.org/TR/WCAG22/ — Accessibility for member-facing web and mobile experiences.
- **W3C Web Authentication (WebAuthn) Level 3** — https://www.w3.org/TR/webauthn-3/ — Passkey login for members and staff.
- **W3C Push API** — https://www.w3.org/TR/push-api/ — Browser push notifications for the member PWA.
- **W3C Payment Request API** — https://www.w3.org/TR/payment-request/ — Standardised giving flows on the web.

### Data Model & API Specifications

- **OpenAPI Specification 3.1** — https://spec.openapis.org/oas/v3.1.0 — Recommended for the public REST API surface.
- **AsyncAPI 3.0** — https://www.asyncapi.com/docs/reference/specification/v3.0.0 — Documentation for webhooks and event streams.
- **JSON Schema 2020-12** — https://json-schema.org/specification — Validation of custom fields, forms, and import/export payloads.
- **JSON:API 1.1** — https://jsonapi.org/format/ — Optional convention for resource-oriented endpoints.
- **GraphQL (October 2021)** — https://spec.graphql.org/October2021/ — Alternative API shape used by some modern ChMS integrations.
- **SCIM 2.0 (RFC 7643/7644)** — https://www.rfc-editor.org/rfc/rfc7644 — User provisioning from denominational SSO directories.
- **SAML 2.0 / OpenID Connect Core 1.0** — https://openid.net/specs/openid-connect-core-1_0.html — Single sign-on for staff and multi-site campuses.
- **Schema.org Person, Event, Organization** — https://schema.org/Person — Structured data for public-facing church events and member directories.

### Security & Authentication Standards

- **OWASP Application Security Verification Standard (ASVS) v4.0.3** — https://owasp.org/www-project-application-security-verification-standard/ — Security baseline for the application.
- **OWASP Top 10 (2021)** — https://owasp.org/www-project-top-ten/ — Minimum vulnerability coverage.
- **NIST SP 800-63B — Digital Identity Guidelines (Authentication)** — https://pages.nist.gov/800-63-3/sp800-63b.html — Password and MFA requirements for staff accounts.
- **PCI DSS v4.0** — https://www.pcisecuritystandards.org/ — Required for handling card data; ideally outsourced to Stripe/Adyen via tokenisation to remain SAQ-A scope.
- **GDPR (EU Regulation 2016/679)** — https://gdpr-info.eu/ — Pastoral notes and member data fall within scope for EU/UK churches.
- **CCPA / CPRA** — https://oag.ca.gov/privacy/ccpa — California members' personal information rights.
- **CAN-SPAM Act** — https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business — Email marketing rules in the US.
- **TCPA (US)** — https://www.fcc.gov/general/telemarketing-and-robocall-rules — Consent requirements for SMS to congregants.
- **HIPAA (where applicable)** — https://www.hhs.gov/hipaa/ — Counselling/care ministries that touch protected health information may trigger HIPAA scope.
- **IRS Publication 1771 — Charitable Contributions Substantiation** — https://www.irs.gov/pub/irs-pdf/p1771.pdf — Required content of contribution statements for US donors.
- **CRA / HMRC / ATO equivalents** — Country-specific donation receipt rules for Canada, UK Gift Aid, and Australia DGR/PAF reporting.

### MCP Server Specifications

- **Model Context Protocol Specification** — https://spec.modelcontextprotocol.io/ — Relevant if exposing church data to AI agents (e.g., a pastor-facing assistant) via a controlled MCP server interface.
- **MCP Reference Servers** — https://github.com/modelcontextprotocol/servers — Useful patterns for read-only data exposure with audit logging.

## Similar Products — Developer Documentation & APIs

### Planning Center
- **Description:** Modular SaaS suite (People, Services, Groups, Giving, Check-Ins, Calendar, Registrations) with the most widely adopted ChMS API.
- **API Documentation:** https://developer.planning.center/docs/
- **SDKs/Libraries:** Community libraries (Ruby, Python, JS); no official SDKs — see https://developer.planning.center/docs/#/overview/libraries
- **Developer Guide:** https://developer.planning.center/docs/#/overview/getting-started
- **Standards:** JSON:API 1.0 conventions over REST
- **Authentication:** OAuth 2.0 (Authorization Code) and Personal Access Tokens

### Pushpay / Church Community Builder (CCB)
- **Description:** Enterprise giving + ChMS platform with REST APIs across donor and church data.
- **API Documentation:** https://pushpay.dev/ and https://docs.ccbchurch.com/
- **SDKs/Libraries:** None official; HTTP examples in documentation
- **Developer Guide:** https://pushpay.dev/docs/getting-started
- **Standards:** REST/JSON
- **Authentication:** OAuth 2.0; legacy CCB APIs use API key + Basic auth

### Breeze ChMS
- **Description:** Easy-to-use ChMS with a simple HTTP API for people, tags, contributions, events, and forms.
- **API Documentation:** https://app.breezechms.com/api
- **SDKs/Libraries:** Community Python `breeze-chms-api` and similar; no official SDKs
- **Developer Guide:** Linked from the API page above
- **Standards:** REST/JSON
- **Authentication:** Per-subdomain API key in `Api-Key` header

### Tithe.ly / Elvanto
- **Description:** Combined giving and ChMS suite with a long-running Elvanto API and newer Tithe.ly Giving API.
- **API Documentation:** https://www.elvanto.com/api/ and https://tithely.developer.tithely.com/
- **SDKs/Libraries:** Community PHP and Python wrappers
- **Developer Guide:** https://www.elvanto.com/api/getting-started/
- **Standards:** REST/JSON
- **Authentication:** OAuth 2.0 and API key

### Subsplash
- **Description:** Branded mobile app, media, and giving platform with developer APIs for content and giving.
- **API Documentation:** https://developers.subsplash.com/
- **SDKs/Libraries:** None official; webhooks documented
- **Developer Guide:** https://developers.subsplash.com/docs
- **Standards:** REST/JSON
- **Authentication:** OAuth 2.0 client credentials

### FellowshipOne
- **Description:** Enterprise multi-site ChMS with mature REST API and partner ecosystem.
- **API Documentation:** https://developer.fellowshipone.com/
- **SDKs/Libraries:** Sample code in C#, Java, PHP, Python on the developer portal
- **Developer Guide:** https://developer.fellowshipone.com/docs/
- **Standards:** REST/JSON and XML
- **Authentication:** OAuth 1.0a (legacy) and 2.0

### Rock RMS
- **Description:** Open source ChMS with a comprehensive REST API and Lava templating engine.
- **API Documentation:** https://community.rockrms.com/Rock/API
- **SDKs/Libraries:** Built-in C# client; community .NET and Node libraries
- **Developer Guide:** https://community.rockrms.com/learn
- **Standards:** OData v3-style REST + custom endpoints
- **Authentication:** API keys, JWT, and cookie-based session

### ChurchCRM
- **Description:** GPL-licensed open source ChMS in PHP/MySQL with a small REST surface.
- **API Documentation:** https://demo.churchcrm.io/api/ (Swagger UI on instances)
- **SDKs/Libraries:** None; OpenAPI document is published per instance
- **Developer Guide:** https://docs.churchcrm.io/
- **Standards:** OpenAPI 2.0 (Swagger)
- **Authentication:** Session cookie / API token

### Stripe (Giving Backbone)
- **Description:** Payment infrastructure widely used by ChMS giving modules.
- **API Documentation:** https://docs.stripe.com/api
- **SDKs/Libraries:** Official SDKs for Node, Python, Ruby, PHP, Java, Go, .NET
- **Developer Guide:** https://docs.stripe.com/payments
- **Standards:** REST/JSON; webhooks; PCI DSS SAQ-A via Elements/Checkout
- **Authentication:** Restricted/secret API keys; OAuth for Connect

### Twilio (Communications Backbone)
- **Description:** SMS, voice, and WhatsApp APIs commonly used for member messaging.
- **API Documentation:** https://www.twilio.com/docs/api
- **SDKs/Libraries:** Official SDKs for Node, Python, Ruby, PHP, Java, Go, .NET
- **Developer Guide:** https://www.twilio.com/docs
- **Standards:** REST/JSON; TwiML; webhooks
- **Authentication:** Account SID + Auth Token; API Key/Secret

### SendGrid / Postmark (Email Backbone)
- **Description:** Transactional and bulk email APIs used by ChMS communications.
- **API Documentation:** https://docs.sendgrid.com/api-reference and https://postmarkapp.com/developer
- **SDKs/Libraries:** Official SDKs in major languages
- **Standards:** REST/JSON; SMTP; SPF/DKIM/DMARC
- **Authentication:** API key (Bearer)

## Notes

- **Emerging area — AI/MCP exposure of church data:** No standardised schema exists yet for exposing congregational data to LLMs. Building on MCP with strict role-aware filtering (especially for pastoral notes) is an open design space.
- **Email deliverability rules tightened in 2024:** Gmail and Yahoo now require SPF, DKIM, DMARC alignment, and one-click unsubscribe (RFC 8058) for bulk senders above 5,000 messages/day — directly relevant for any ChMS communications module.
- **Donation receipt rules vary by jurisdiction:** US (IRS Pub 1771), UK Gift Aid, Canada CRA, Australia DGR — there is no global standard. The data model needs jurisdiction-aware receipting.
- **No formal interoperability standard exists for ChMS data:** Migrations are typically CSV-based. A community "ChMS Interchange Format" would be a valuable contribution but does not yet exist.
- **Children's check-in patent landscape is unclear:** Several large vendors implement near-identical pickup-code/label-print flows. Original implementations should be designed from first principles and documented to avoid disputes.
