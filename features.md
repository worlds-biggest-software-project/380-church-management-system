# Church Management System (ChMS) — Feature & Functionality Survey

> Candidate #380 · Researched: 2026-05-06

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Breeze ChMS | Commercial SaaS | Proprietary, flat-rate subscription | https://www.breezechms.com/ |
| Planning Center | Commercial SaaS (modular) | Proprietary, per-product subscription | https://www.planningcenter.com/ |
| Pushpay (ChurchStaq) | Commercial SaaS | Proprietary, enterprise pricing | https://pushpay.com/ |
| ChMeetings | Commercial SaaS | Proprietary, tiered subscription | https://www.chmeetings.com/ |
| Elvanto (Tithe.ly ChMS) | Commercial SaaS | Proprietary, per-user pricing | https://www.elvanto.com/ |
| Subsplash | Commercial SaaS (mobile-first) | Proprietary, suite pricing | https://www.subsplash.com/ |
| FellowshipOne (Ministry Brands) | Enterprise SaaS | Proprietary, custom pricing | https://www.fellowshipone.com/ |
| Ministry Platform | Enterprise SaaS | Proprietary, custom pricing | https://www.ministryplatform.com/ |
| ChurchTrac | Commercial SaaS | Proprietary, low-cost subscription | https://www.churchtrac.com/ |
| Rock RMS | Open Source | MIT-style (Rock Community License) | https://www.rockrms.com/ |
| ChurchCRM | Open Source | GPL v3 (PHP) | https://churchcrm.io/ |
| Flocknote | Commercial SaaS (comms-focused) | Proprietary, per-member pricing | https://flocknote.com/ |

## Feature Analysis by Solution

### Breeze ChMS

**Core features**
- People/household profiles with photos, custom fields, tags
- Online giving with recurring donations and statements
- Event registration and check-in (kiosk and mobile)
- Email & SMS messaging to filtered groups
- Volunteer scheduling and group management
- Reports library and basic dashboards

**Differentiating features**
- Flat-rate pricing regardless of church size
- Reputation as the easiest ChMS to onboard non-technical admins
- Tag-based segmentation rather than rigid group hierarchies

**UX patterns**
- Single dashboard, minimal nav depth
- Inline editing of profiles and fields
- Onboarding wizard plus extensive video library

**Integration points**
- Native online giving (Stripe-backed)
- Mailchimp, QuickBooks, Zapier
- Public REST-style API

**Known gaps**
- Limited workflow automation
- No deep multi-site support
- Basic worship-service planning compared to Planning Center

**Licence / IP notes**
- Closed source; "Breeze" trademarked

### Planning Center

**Core features**
- Modular products: People, Services, Groups, Giving, Check-Ins, Calendar, Registrations, Publishing, Music Stand
- Service planning with order of service, song library, rehearsals
- Volunteer scheduling with conflict detection
- Recurring giving and ACH/Card processing
- Children's check-in with label printing
- Mobile apps for members and volunteers (Church Center)

**Differentiating features**
- Best-in-class worship service planning module
- Clean separation of products (buy what you need)
- Strong public-facing Church Center app for members

**UX patterns**
- Distinct app per module with consistent design system
- "Matrix" view for volunteer scheduling
- Drag-and-drop service order builder

**Integration points**
- Public API (OAuth 2.0) widely adopted by 3rd-party developers
- Webhooks for major events
- Integrations with CCLI, MultiTracks, Loop Community

**Known gaps**
- Costs add up quickly when adopting many modules
- Some redundant data entry across modules
- No native AI/automation layer

**Licence / IP notes**
- Closed source; some patents around scheduling matrix UI reportedly held

### Pushpay (ChurchStaq)

**Core features**
- Integrated giving (text, app, web, kiosk) with high conversion focus
- Resi streaming and church mobile app builder
- ChMS (formerly Church Community Builder) for people, groups, processes
- Volunteer scheduling and rotation
- Pastoral care workflows / process queues

**Differentiating features**
- Industry-leading donor conversion analytics
- Custom branded mobile app per church
- Heavy investment in pastoral "process" automation

**UX patterns**
- Donor-facing flows are conversion-optimised (Apple Pay, one-tap)
- Process queues route members through staff workflows
- Admin UI is more enterprise-feeling than Breeze

**Integration points**
- Resi (streaming), Church Community Builder API
- Accounting (Sage Intacct, QuickBooks)
- Salesforce-style data exports

**Known gaps**
- Pricing opaque and high
- Steeper learning curve
- Locked ecosystem; switching cost is significant

**Licence / IP notes**
- Closed source; multiple acquisitions consolidated under Pushpay (CCB, Resi)

### ChMeetings

**Core features**
- Member database with custom fields
- Event creation with RSVP/registration
- SMS and email blasts
- Attendance tracking
- Group and ministry management
- Mobile app for members

**Differentiating features**
- Strong international (multi-language, multi-currency) support
- Affordable for small/mid churches
- Quick setup (cloud-only, no install)

**UX patterns**
- Tile-based dashboard
- Mobile-first layouts even on web
- Localisation across 20+ languages

**Integration points**
- Stripe, PayPal, Square for giving
- Zoom for online events
- Limited public API

**Known gaps**
- Reporting depth is shallow
- No service planning module
- Limited workflow automation

**Licence / IP notes**
- Closed source

### Elvanto (Tithe.ly ChMS)

**Core features**
- People, groups, services, volunteer rosters
- Forms and registrations
- Reporting and custom queries
- Check-in for kids and events
- File and song library

**Differentiating features**
- Strong roster/availability handling
- Demographic reporting and queries
- Popular outside North America (AU/NZ/UK roots)

**UX patterns**
- Configurable dashboards per role
- Custom report builder
- Volunteer self-service portal

**Integration points**
- Tithe.ly Giving (post-acquisition)
- Zapier, Mailchimp
- API with REST + webhooks

**Known gaps**
- Mobile app less polished than Planning Center
- UI feels dated in places post-acquisition
- Roadmap velocity has slowed

**Licence / IP notes**
- Closed source; Tithe.ly owns the product

### Subsplash

**Core features**
- Branded mobile and web apps
- Media hosting (sermons, podcasts, video)
- Giving (one-time, recurring, text)
- Groups and messaging
- Push notifications

**Differentiating features**
- Mobile app quality is a primary selling point
- Tight media + giving integration
- Custom branded experience for each church

**UX patterns**
- Member experience optimised over admin experience
- Content-first navigation (sermons, articles)
- Push-driven engagement loops

**Integration points**
- Sermon ingestion from YouTube/Vimeo
- ChMS data sync via API to Planning Center, CCB
- ProPresenter integrations

**Known gaps**
- Lighter on traditional ChMS depth (people, volunteers)
- Often used alongside another ChMS rather than replacing it
- Reporting is media/engagement focused

**Licence / IP notes**
- Closed source; mobile app builder uses proprietary template engine

### FellowshipOne

**Core features**
- Enterprise people and household management
- Contributions and pledge tracking
- Multi-site/campus support
- Workflows ("Activities") for staff processes
- Reporting and ad-hoc query builder

**Differentiating features**
- Designed for very large multi-campus churches
- Deep customisation through staff-built workflows
- Long-tenured product with strong fund-accounting maturity

**UX patterns**
- Traditional desktop-style admin UI
- Heavy use of saved searches and queues
- Steep but powerful query builder

**Integration points**
- Public REST API
- SSO via SAML
- Marketplace of partner integrations

**Known gaps**
- Heavy and expensive for small churches
- Modernisation lags newer SaaS competitors
- Implementation typically requires consultants

**Licence / IP notes**
- Closed source; owned by Ministry Brands

### Ministry Platform

**Core features**
- Highly relational data model (think CRM-on-rails for churches)
- Workflows, processes, communications
- Giving with deep fund accounting
- Multi-site support with security per campus
- Reporting via SSRS-style report packages

**Differentiating features**
- Most data-model-flexible enterprise ChMS
- Power users can build custom pages and reports
- Strong fit for churches with internal IT

**UX patterns**
- Table-and-form heavy admin UX
- Configurable security at field level
- Power-user oriented; consultant ecosystem

**Integration points**
- REST API
- SQL access for on-prem deployments historically
- Partners: Text-In-Church, Gloo, etc.

**Known gaps**
- Requires technical resource to operate well
- UI not friendly to volunteer admins
- Mobile experience trails native-mobile competitors

**Licence / IP notes**
- Closed source; owned by ACS Technologies

### ChurchTrac

**Core features**
- People, attendance, giving, accounting
- Built-in fund accounting (rare at this price)
- Email and forms
- Pledges and statements

**Differentiating features**
- Includes proper double-entry accounting
- Very low price point
- Originally desktop, now full SaaS

**UX patterns**
- Compact UI focused on data entry speed
- Accountant-friendly layouts for funds and ledgers

**Integration points**
- Stripe for online giving
- CSV import/export
- Limited third-party integrations

**Known gaps**
- No mobile app for members
- Limited automation/workflow
- Visually utilitarian

**Licence / IP notes**
- Closed source

### Rock RMS

**Core features**
- People, families, groups, attendance
- Giving with batches and statements
- Workflows engine (highly customisable)
- Communications (email, SMS, push)
- Check-in with label printing
- Content/CMS module

**Differentiating features**
- Open source under permissive Rock Community License
- Built-in CMS (no need for separate website tool)
- Visual workflow designer rivals enterprise CRMs

**UX patterns**
- Block-based page composition
- Workflow designer with drag-drop activities
- Admin UI designed for power users

**Integration points**
- REST API + Lava templating engine
- Webhook-based integrations
- Active partner ecosystem (Spark Development Network)

**Known gaps**
- Self-hosting requires Windows/.NET expertise historically
- Steep learning curve for non-technical staff
- Hosted offerings exist but at additional cost

**Licence / IP notes**
- Rock Community License (MIT-style with naming/branding restrictions)
- Self-host or use partner-hosted

### ChurchCRM

**Core features**
- Member and family management
- Donations and pledges
- Sunday school and group management
- Email queues
- Calendar and events

**Differentiating features**
- Fully open source (GPL v3)
- PHP/MySQL stack — easy to self-host
- Free for any size congregation

**UX patterns**
- Bootstrap-based traditional CRUD UI
- Form-driven workflows
- Minimal onboarding assistance

**Integration points**
- Stripe/PayPal donation plugins
- iCal feed for calendar
- REST API (limited)

**Known gaps**
- UX dated and inconsistent
- Mobile experience minimal
- Reporting is basic
- No built-in service planning

**Licence / IP notes**
- GPL v3 — derivative works must remain GPL

### Flocknote

**Core features**
- Email + SMS communications to congregation
- Group/segment management
- Sign-up forms
- Notes (long-form messages with replies)

**Differentiating features**
- Specialised in communications, not full ChMS
- Strong adoption in Catholic parishes
- Reply-to-sender SMS workflows

**UX patterns**
- Two-way messaging UI like a consumer messaging app
- Group sign-ups via short codes

**Integration points**
- ParishSOFT, basic ChMS data sync
- Limited API

**Known gaps**
- Not a full ChMS — needs companion system
- No giving, no check-in, no service planning

**Licence / IP notes**
- Closed source

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Member/household profiles with custom fields and tags
- Online giving (one-time, recurring, ACH, card, text)
- Year-end contribution statements
- Email + SMS bulk communications with segmentation
- Event registration and RSVP
- Children's check-in with label printing and security tags
- Small group/ministry team management
- Volunteer scheduling and self-service availability
- Attendance tracking
- Mobile app or PWA for members
- Role-based access control with pastoral note privacy
- Standard reports (giving, attendance, group health)

### Differentiating Features
- Worship service planning (order of service, song library, rehearsals) — Planning Center's moat
- Branded mobile app (Subsplash, Pushpay)
- Workflow/process automation engine (Rock, Pushpay, Ministry Platform)
- Multi-site campus consolidation with per-campus security
- Integrated streaming (Pushpay/Resi)
- Pastoral care workflows ("process queues")
- Donor analytics and conversion optimisation
- Built-in CMS / website (Rock RMS)
- Built-in fund accounting (ChurchTrac)
- Localisation and multi-currency (ChMeetings)

### Underserved Areas / Opportunities
- Genuinely modern open-source option with polished UX (Rock and ChurchCRM both have UX gaps)
- AI-assisted pastoral care prompting (who haven't we connected with recently?)
- Automated first-time-visitor follow-up sequences with personalised content
- Volunteer burnout prediction and load balancing across households
- Sermon-to-content automation (sermon → blog → social posts → small group discussion guide)
- Natural-language report querying for non-technical staff
- Privacy-first pastoral notes with on-device or end-to-end encryption
- Cross-platform data portability (most platforms lock data in)
- Affordable multi-site for mid-sized churches (currently enterprise-priced)
- Integrated benevolence/care request tracking

### AI-Augmentation Candidates
- Drafting personalised follow-up messages to first-time visitors
- Summarising pastoral visit notes from voice memos
- Suggesting volunteer slots based on availability, skills, and rotation history
- Generating small group discussion questions from sermon transcripts
- Detecting at-risk members (declining attendance/giving patterns) for pastoral outreach
- Auto-categorising and tagging incoming prayer requests
- Natural-language search across people, events, and giving ("show me first-time givers from last quarter who haven't been contacted")
- Automated transcription and translation of sermons/announcements
- Smart segmentation suggestions for communications
- AI-assisted form/process builder

## Legal & IP Summary

The ChMS market is dominated by closed-source SaaS with proprietary trademarks and brand-protected mobile apps. Two open-source alternatives exist with different licence implications: **Rock RMS** uses a permissive Rock Community License (MIT-style with brand restrictions) suitable for derivative work and SaaS reuse; **ChurchCRM** is GPL v3, meaning any derivative product would also need to be GPL-licensed and source-disclosed. Public APIs from Planning Center, Pushpay/CCB, FellowshipOne, and Elvanto can be consumed for integration, but their schemas and workflows are not redistributable. Children's check-in workflows (label printing, custody alerts, pickup codes) and donor-conversion mobile flows (one-tap Apple Pay giving) may have implementation patents held by Pushpay or Planning Center; original implementations should be designed independently. Year-end contribution statements must comply with IRS substantiation rules (US) and equivalent tax-receipt rules in other jurisdictions — this is regulatory, not IP, but must be implemented correctly. No patent or copyright concerns prevent building a new AI-native ChMS from scratch using publicly documented standards.

## Recommended Feature Scope

**Must-have (MVP)**
- Member/household profiles with custom fields and tags
- Online giving (Stripe-backed) with recurring donations and contribution statements
- Email + SMS communications with segmentation
- Event registration and RSVP with capacity controls
- Children's check-in with label printing and pickup codes
- Small groups and volunteer rosters with self-service availability
- Mobile-friendly member portal (PWA)
- Role-based access control with pastoral note privacy

**Should-have (v1.1)**
- AI-drafted first-time-visitor follow-up sequences
- Natural-language query for reports and people search
- Volunteer burnout/load balancing suggestions
- Worship service planning module (order of service, song library)
- Multi-site campus support with per-campus security
- Public REST API + webhooks for ecosystem integrations
- Sermon transcription and small-group discussion guide generation

**Nice-to-have (backlog)**
- Branded mobile app builder
- Built-in fund accounting / QuickBooks deep sync
- Integrated streaming (Resi alternative or YouTube/Vimeo bridge)
- Workflow/process automation engine with visual designer
- Built-in CMS for church website
- End-to-end encrypted pastoral notes
- Multi-language and multi-currency for international congregations
