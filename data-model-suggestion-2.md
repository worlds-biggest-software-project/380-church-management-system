# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Church Management System (ChMS) · Created: 2026-05-26

## Philosophy

This model consolidates the church management domain into a small number of wide tables where core identifiers and query-critical fields remain relational columns, while variable-depth structures — household members, group involvement, attendance history, giving details, volunteer schedules, and custom fields — live in JSONB columns. Each major aggregate (church, person, event) is a self-contained document readable in a single query.

Churches are intensely customisable environments: every congregation has different ministry structures, custom fields, milestone definitions, and form layouts. A normalized model requires schema changes for each new custom field or ministry type; this hybrid absorbs that variability in JSONB. The person record in particular benefits — a single SELECT returns a member's household, groups, giving summary, attendance history, volunteer commitments, and pastoral notes, exactly what a pastor or admin needs on a "member profile" screen.

The design keeps donations as a separate relational table because financial data requires strict auditability, Stripe webhook reconciliation, and year-end reporting queries that benefit from column-level indexing. Everything else consolidates around church, person, and event aggregates.

**Best for:** Small-to-mid churches with volunteer administrators, rapid MVP development, and deployments needing maximum custom-field flexibility without developer involvement.

**Trade-offs:**
- (+) 6 tables vs. 13 — dramatically simpler for volunteer administrators to understand
- (+) Custom fields, milestones, and ministry types are JSONB — no migrations needed
- (+) Full member profile is a single-row read
- (+) GIN indexes support fast tag and group membership queries
- (-) Financial reporting queries on embedded giving summaries are less precise than relational joins
- (-) No FK enforcement within JSONB (group membership validity is application-enforced)
- (-) Large person records for highly engaged long-term members
- (-) Household-level giving statements require aggregating across person JSONB

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IRS Publication 1771 | Donation table has dedicated columns for tax-compliant statements |
| PCI DSS v4.0 | No card data stored; Stripe references only |
| ISO 4217 | currency_code on donations for multi-currency |
| RFC 5545 (iCalendar) | Event fields support iCal export |
| RFC 6350 (vCard) | Person identity fields map to vCard |
| GDPR / CCPA | Privacy fields in persons.privacy_json |
| CAN-SPAM / TCPA | Consent tracking in persons.communication_json |
| OAuth 2.0 / OIDC | Auth config in churches.staff_json |

---

## Core Tables

### churches

```sql
CREATE TABLE churches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    campus_type     TEXT NOT NULL DEFAULT 'main' CHECK (campus_type IN (
                        'main', 'satellite', 'online', 'church_plant'
                    )),
    parent_church_id UUID REFERENCES churches(id),
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    country_code    TEXT NOT NULL DEFAULT 'US',
    currency_code   TEXT NOT NULL DEFAULT 'USD',
    identity_json   JSONB NOT NULL DEFAULT '{}',
    -- { "address": {...}, "phone": "...", "email": "...",
    --   "website": "...", "denomination": "...",
    --   "service_times": ["09:00", "11:00"] }
    staff_json      JSONB NOT NULL DEFAULT '[]',
    -- [{ "user_id": "...", "email": "...", "name": "...",
    --    "role": "senior_pastor", "mfa_enabled": true,
    --    "webauthn_registered": false, "is_active": true }]
    config_json     JSONB NOT NULL DEFAULT '{}',
    -- { "giving_funds": ["general", "missions", "building", "benevolence"],
    --   "tax_receipt_rules": "irs_pub_1771",
    --   "fiscal_year_start": "01-01",
    --   "check_in_label_format": "dymo_30252",
    --   "custom_fields": [{ "name": "...", "type": "select", "options": [...] }],
    --   "milestones": ["baptism", "membership_class", "salvation", "dedication"],
    --   "member_statuses": ["visitor", "first_time_visitor", "regular_attender", "member"],
    --   "volunteer_max_consecutive_weeks": 4 }
    groups_json     JSONB NOT NULL DEFAULT '[]',
    -- [{ "group_id": "...", "name": "Young Adults", "type": "small_group",
    --    "leader_id": "...", "leader_name": "...", "status": "active",
    --    "meeting_day": "wednesday", "meeting_time": "19:00",
    --    "location": "Room 201", "capacity": 15, "member_count": 12,
    --    "is_public": true, "enrollment_open": true }]
    integrations_json JSONB NOT NULL DEFAULT '{}',
    -- { "stripe_account_id": "acct_...",
    --   "twilio_from": "+1234567890", "twilio_sid": "...",
    --   "sendgrid_from": "church@example.org",
    --   "quickbooks_realm_id": "...",
    --   "ccli_license": "12345" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_churches_parent ON churches (parent_church_id);
CREATE INDEX idx_churches_type ON churches (campus_type);
CREATE INDEX idx_churches_groups ON churches USING GIN (groups_json);
```

### persons

```sql
CREATE TABLE persons (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    church_id           UUID NOT NULL REFERENCES churches(id),
    first_name          TEXT NOT NULL,
    last_name           TEXT NOT NULL,
    email               TEXT,
    phone_mobile        TEXT,
    date_of_birth       DATE,
    member_status       TEXT NOT NULL DEFAULT 'visitor' CHECK (member_status IN (
                            'visitor', 'first_time_visitor', 'regular_attender',
                            'member', 'inactive', 'transferred', 'deceased'
                        )),
    primary_campus_id   UUID REFERENCES churches(id),
    tags                TEXT[] DEFAULT '{}',
    identity_json       JSONB NOT NULL DEFAULT '{}',
    -- { "nickname": "...", "gender": "male", "marital_status": "married",
    --   "photo_url": "...", "address": {...},
    --   "phone_home": "...", "country_code": "US" }
    household_json      JSONB NOT NULL DEFAULT '{}',
    -- { "household_id": "...", "family_name": "Smith",
    --   "role": "head", "members": [
    --     { "person_id": "...", "name": "Jane Smith", "relationship": "spouse" },
    --     { "person_id": "...", "name": "Tommy Smith", "relationship": "child",
    --       "grade": "3rd", "allergies": ["peanuts"], "custody_alert": null }
    --   ] }
    milestones_json     JSONB NOT NULL DEFAULT '{}',
    -- { "first_visit": "2024-03-15", "membership_date": "2024-09-01",
    --   "baptism": "2024-06-15", "membership_class": "2024-08-20",
    --   "salvation": "2024-03-15", "custom": [...] }
    groups_json         JSONB NOT NULL DEFAULT '[]',
    -- [{ "group_id": "...", "name": "Young Adults", "type": "small_group",
    --    "role": "member", "joined_date": "2024-10-01" }]
    attendance_json     JSONB NOT NULL DEFAULT '{}',
    -- { "last_date": "2026-05-25", "streak": 4,
    --   "total_services_ytd": 18, "total_events_ytd": 6,
    --   "recent": [
    --     { "event_id": "...", "name": "Sunday 11AM", "date": "2026-05-25",
    --       "type": "sunday_service" }
    --   ] }
    giving_json         JSONB NOT NULL DEFAULT '{}',
    -- { "last_date": "2026-05-20", "ytd_total_cents": 520000,
    --   "ytd_by_fund": { "general": 400000, "missions": 120000 },
    --   "lifetime_total_cents": 3500000,
    --   "recurring": { "active": true, "amount_cents": 20000,
    --     "frequency": "monthly", "fund": "general",
    --     "stripe_subscription_id": "sub_..." },
    --   "pledges": [{ "pledge_id": "...", "campaign": "Building Fund",
    --     "total": 500000, "fulfilled": 200000, "status": "active" }] }
    volunteer_json      JSONB NOT NULL DEFAULT '[]',
    -- [{ "position_id": "...", "group_id": "...", "position": "Greeter",
    --    "status": "active", "frequency": "bi_weekly",
    --    "last_served": "2026-05-18", "next_scheduled": "2026-06-01",
    --    "consecutive_weeks": 2, "total_served": 24,
    --    "preferred_service": "09:00" }]
    pastoral_json       JSONB NOT NULL DEFAULT '{}',
    -- { "notes": "...", "private": true,
    --   "last_pastoral_visit": "2026-04-10",
    --   "care_requests": [{ "date": "...", "category": "health",
    --     "description": "...", "status": "follow_up" }],
    --   "prayer_requests": [{ "date": "...", "request": "...",
    --     "category": "health", "shared_publicly": false }] }
    communication_json  JSONB NOT NULL DEFAULT '{}',
    -- { "email_opt_in": true, "sms_opt_in": false,
    --   "sms_consent_date": null, "push_enabled": false,
    --   "do_not_contact": false,
    --   "follow_up_sequence": { "active": false, "step": 0, "started": null } }
    privacy_json        JSONB NOT NULL DEFAULT '{}',
    -- { "gdpr_consent": null, "gdpr_consent_date": null,
    --   "directory_visible": true, "giving_visible_to_staff": true }
    portal_json         JSONB NOT NULL DEFAULT '{}',
    -- { "enabled": false, "user_id": null, "last_login": null }
    custom_fields_json  JSONB NOT NULL DEFAULT '{}',
    check_in_json       JSONB NOT NULL DEFAULT '{}',
    -- Children-specific check-in defaults
    -- { "allergies": ["peanuts"], "medical_notes": "...",
    --   "custody_alert": "Do not release to father",
    --   "authorized_pickups": ["...", "..."],
    --   "grade": "3rd", "school": "Lincoln Elementary" }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_persons_church ON persons (church_id);
CREATE INDEX idx_persons_name ON persons (last_name, first_name);
CREATE INDEX idx_persons_email ON persons (email) WHERE email IS NOT NULL;
CREATE INDEX idx_persons_status ON persons (church_id, member_status);
CREATE INDEX idx_persons_campus ON persons (primary_campus_id);
CREATE INDEX idx_persons_tags ON persons USING GIN (tags);
CREATE INDEX idx_persons_household ON persons USING GIN (household_json);
CREATE INDEX idx_persons_groups ON persons USING GIN (groups_json);
CREATE INDEX idx_persons_giving ON persons USING GIN (giving_json);
CREATE INDEX idx_persons_volunteer ON persons USING GIN (volunteer_json);
```

### events

```sql
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    church_id       UUID NOT NULL REFERENCES churches(id),
    campus_id       UUID REFERENCES churches(id),
    name            TEXT NOT NULL,
    event_type      TEXT NOT NULL CHECK (event_type IN (
                        'sunday_service', 'midweek_service', 'special_service',
                        'conference', 'retreat', 'vbs', 'youth_event',
                        'small_group_meeting', 'class', 'meeting',
                        'outreach', 'fundraiser', 'social', 'other'
                    )),
    status          TEXT NOT NULL DEFAULT 'scheduled' CHECK (status IN (
                        'draft', 'scheduled', 'in_progress', 'completed', 'cancelled'
                    )),
    starts_at       TIMESTAMPTZ NOT NULL,
    ends_at         TIMESTAMPTZ,
    location        TEXT,
    description     TEXT,
    capacity        INTEGER,
    is_recurring    BOOLEAN NOT NULL DEFAULT false,
    recurrence_rule TEXT,
    parent_event_id UUID REFERENCES events(id),
    registration_json JSONB NOT NULL DEFAULT '{}',
    -- { "open": true, "deadline": "...", "fee_cents": 0,
    --   "registered_count": 45, "waitlist_count": 0 }
    attendance_json JSONB NOT NULL DEFAULT '{}',
    -- { "total_attended": 120, "total_first_time": 3,
    --   "total_children": 22, "total_volunteers_serving": 15,
    --   "attendees": [
    --     { "person_id": "...", "name": "...", "status": "attended",
    --       "checked_in_at": "...", "is_child": false, "is_serving": true }
    --   ] }
    check_in_json   JSONB NOT NULL DEFAULT '{}',
    -- { "enabled": true, "rooms": ["Nursery", "K-2nd", "3rd-5th"],
    --   "checked_in": [
    --     { "person_id": "...", "child_name": "Tommy Smith",
    --       "security_code": "A7B3", "room": "K-2nd",
    --       "checked_in_by": "...", "checked_in_at": "...",
    --       "checked_out_to": null, "checked_out_at": null,
    --       "allergy_alert": "peanuts", "custody_alert": null,
    --       "label_printed": true }
    --   ] }
    worship_json    JSONB NOT NULL DEFAULT '{}',
    -- { "sermon_title": "...", "speaker": "...", "notes_url": "...",
    --   "songs": ["Amazing Grace", "How Great Is Our God"],
    --   "order_of_service": [...] }
    tags            TEXT[] DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_events_church ON events (church_id);
CREATE INDEX idx_events_campus ON events (campus_id);
CREATE INDEX idx_events_date ON events (starts_at);
CREATE INDEX idx_events_type ON events (church_id, event_type);
CREATE INDEX idx_events_upcoming ON events (starts_at)
    WHERE status = 'scheduled' AND starts_at > now();
CREATE INDEX idx_events_attendance ON events USING GIN (attendance_json);
CREATE INDEX idx_events_checkin ON events USING GIN (check_in_json);
```

### donations

```sql
CREATE TABLE donations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    church_id       UUID NOT NULL REFERENCES churches(id),
    person_id       UUID REFERENCES persons(id),
    amount_cents    BIGINT NOT NULL,
    currency_code   TEXT NOT NULL DEFAULT 'USD',
    fund            TEXT NOT NULL DEFAULT 'general',
    donation_type   TEXT NOT NULL CHECK (donation_type IN (
                        'one_time', 'recurring', 'pledge_payment'
                    )),
    payment_method  TEXT NOT NULL CHECK (payment_method IN (
                        'card', 'ach', 'cash', 'check', 'text_to_give',
                        'kiosk', 'stock', 'other'
                    )),
    stripe_payment_id TEXT,
    stripe_subscription_id TEXT,
    check_number    TEXT,
    batch_id        TEXT,
    status          TEXT NOT NULL DEFAULT 'completed' CHECK (status IN (
                        'pending', 'completed', 'failed', 'refunded', 'disputed'
                    )),
    tax_deductible  BOOLEAN NOT NULL DEFAULT true,
    receipt_sent    BOOLEAN NOT NULL DEFAULT false,
    pledge_id       TEXT,                -- reference to pledge in person's giving_json
    donation_date   DATE NOT NULL,
    fiscal_year     TEXT NOT NULL,
    is_anonymous    BOOLEAN NOT NULL DEFAULT false,
    memo            TEXT,
    household_name  TEXT,                -- denormalized for statement generation
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_donations_church ON donations (church_id);
CREATE INDEX idx_donations_person ON donations (person_id);
CREATE INDEX idx_donations_date ON donations (donation_date);
CREATE INDEX idx_donations_fund ON donations (church_id, fund, donation_date);
CREATE INDEX idx_donations_fiscal ON donations (church_id, fiscal_year);
CREATE INDEX idx_donations_stripe ON donations (stripe_payment_id)
    WHERE stripe_payment_id IS NOT NULL;
CREATE INDEX idx_donations_batch ON donations (batch_id) WHERE batch_id IS NOT NULL;
```

---

## AI & Audit

### ai_suggestions

```sql
CREATE TABLE ai_suggestions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    church_id       UUID NOT NULL REFERENCES churches(id),
    person_id       UUID,
    suggestion_type TEXT NOT NULL CHECK (suggestion_type IN (
                        'visitor_follow_up', 'pastoral_outreach',
                        'volunteer_slot', 'burnout_warning',
                        'at_risk_member', 'giving_pattern',
                        'sermon_discussion_guide', 'prayer_categorisation',
                        'communication_segment', 'report_query',
                        'group_health', 'event_recommendation'
                    )),
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                        'pending', 'accepted', 'rejected', 'modified', 'expired'
                    )),
    model_id        TEXT NOT NULL,
    suggestion_json JSONB NOT NULL,
    confidence      NUMERIC(4, 3),
    explanation     TEXT,
    accepted_by     TEXT,
    accepted_at     TIMESTAMPTZ,
    feedback        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_suggestions_church ON ai_suggestions (church_id);
CREATE INDEX idx_ai_suggestions_type ON ai_suggestions (suggestion_type);
CREATE INDEX idx_ai_suggestions_pending ON ai_suggestions (status) WHERE status = 'pending';
```

### audit_log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    church_id       UUID NOT NULL REFERENCES churches(id),
    user_id         UUID,
    actor_type      TEXT NOT NULL DEFAULT 'user' CHECK (actor_type IN (
                        'user', 'system', 'ai', 'integration',
                        'cron', 'member_portal', 'kiosk'
                    )),
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    changes_json    JSONB,
    ip_address      INET,
    session_id      TEXT,
    pastoral_data   BOOLEAN NOT NULL DEFAULT false,
    financial_data  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_log_church ON audit_log (church_id, created_at);
CREATE INDEX idx_audit_log_user ON audit_log (user_id, created_at);
CREATE INDEX idx_audit_log_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_log_pastoral ON audit_log (pastoral_data, created_at)
    WHERE pastoral_data = true;
```

---

## Example Queries

### First-time visitors from last month not yet contacted

```sql
SELECT id, first_name, last_name, email, phone_mobile,
       milestones_json->>'first_visit' AS first_visit,
       communication_json->>'follow_up_sequence' AS follow_up
FROM persons
WHERE church_id = $1
  AND member_status = 'first_time_visitor'
  AND (milestones_json->>'first_visit')::date
      BETWEEN (now() - interval '30 days')::date AND now()::date
  AND (communication_json->'follow_up_sequence'->>'active')::boolean IS NOT TRUE
ORDER BY (milestones_json->>'first_visit')::date;
```

### Year-end giving statement per household

```sql
SELECT p.household_json->>'family_name' AS family_name,
       d.fund,
       SUM(d.amount_cents) AS total_cents,
       COUNT(*) AS donation_count
FROM donations d
JOIN persons p ON p.id = d.person_id
WHERE d.church_id = $1
  AND d.fiscal_year = '2026'
  AND d.status = 'completed'
  AND d.tax_deductible = true
GROUP BY p.household_json->>'household_id', p.household_json->>'family_name', d.fund
ORDER BY family_name, fund;
```

### Volunteers approaching burnout threshold

```sql
SELECT id, first_name, last_name,
       v->>'position' AS position,
       (v->>'consecutive_weeks')::int AS consecutive,
       v->>'next_scheduled' AS next_date
FROM persons,
     jsonb_array_elements(volunteer_json) AS v
WHERE church_id = $1
  AND (v->>'status') = 'active'
  AND (v->>'consecutive_weeks')::int >= 3
ORDER BY (v->>'consecutive_weeks')::int DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Church | 1 | Staff, groups, config, integrations all in JSONB |
| Person | 1 | Central aggregate: household, groups, attendance, giving, volunteering, pastoral |
| Event | 1 | Registration, attendance, check-in, worship all in JSONB |
| Donations | 1 | Relational for financial auditability and Stripe reconciliation |
| AI & Audit | 2 | Suggestions and partitioned audit log |
| **Total** | **6** | |

---

## Key Design Decisions

1. **Person as central aggregate** — all member-related data (household, groups, milestones, attendance, giving summary, volunteering, pastoral notes, communication preferences, privacy) lives in one row. The "member profile" screen is a single SELECT.

2. **Donations remain relational** — financial transactions need column-level indexing for fiscal-year statements, fund reporting, Stripe webhook reconciliation, and audit. The person's `giving_json` holds summary/aggregate data updated by application triggers, not the source of truth.

3. **Groups at church level** — `churches.groups_json` holds the group directory. Individual membership lives in `persons.groups_json`. This avoids a separate groups table while maintaining the church-level directory view and per-person involvement view.

4. **Children's check-in embedded in events** — `events.check_in_json` contains the security codes, room assignments, and allergy/custody alerts for each checked-in child. `persons.check_in_json` stores the child's default allergies and custody alerts. This gives each event its own check-in state while person-level defaults carry forward.

5. **Giving summary on persons** — `persons.giving_json` provides quick lookups for pastoral care (has this person's giving declined?) and portal display (show my giving YTD). It's a denormalized summary updated by application logic when donations are created.

6. **Pastoral notes isolated in JSONB** — `pastoral_json` is a separate JSONB key, enabling middleware to enforce access control on this specific path. The audit log flags `pastoral_data = true` for any access.

7. **Follow-up sequences in communication_json** — automated visitor follow-up state (current step, started date, active flag) lives on the person record, making it easy to query "who needs the next follow-up step" without a separate workflow table.

8. **Custom fields as JSONB** — `persons.custom_fields_json` and the custom field definitions in `churches.config_json` support unlimited church-specific fields without schema changes.
