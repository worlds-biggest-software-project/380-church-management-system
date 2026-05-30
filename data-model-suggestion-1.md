# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Church Management System (ChMS) · Created: 2026-05-26

## Philosophy

This model assigns a dedicated table to every major ChMS concept — churches, people, households, groups, events, attendance, donations, pledges, volunteer schedules, check-ins, and communications — connected through explicit foreign keys. The design follows the congregational lifecycle from first visit through membership, giving, group participation, volunteering, and pastoral care.

The schema separates financial data (donations, pledges) from engagement data (attendance, groups, volunteering) at the table level, supporting both fund-accounting reporting and pastoral engagement views without cross-contamination. Multi-site architecture uses a `churches` table where each campus is a row, with people potentially attending multiple campuses.

Giving data is structured for IRS-compliant contribution statements (US), Gift Aid (UK), and equivalent jurisdictional requirements, with fund allocation as a first-class concept rather than a tag. The children's check-in system uses a dedicated table with security codes, allergy alerts, and custody flags as required columns.

**Best for:** Churches with dedicated administrative staff, multi-site campuses needing consolidated reporting, and deployments requiring strict fund-accounting separation for tax compliance.

**Trade-offs:**
- (+) Direct column-level mapping for IRS contribution statement generation
- (+) FK enforcement prevents orphaned donations or phantom group members
- (+) Clear separation of financial, engagement, and pastoral data for access control
- (+) Standard relational queries; accessible to non-technical report builders
- (-) 13 tables with junction patterns; schema changes need coordination
- (-) Custom fields per church require a config table or JSONB column
- (-) Multi-site member sharing requires careful FK management

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IRS Publication 1771 | Donation fields map directly to required contribution statement content |
| PCI DSS v4.0 | No card data stored; Stripe processor_ref only (SAQ-A scope) |
| ISO 4217 | currency_code on donations and pledges for multi-currency giving |
| RFC 5545 (iCalendar) | Events export via iCal feed using event fields |
| RFC 6350 (vCard) | People export via vCard using contact fields |
| GDPR / CCPA | Consent fields on people; pastoral notes privacy flag |
| CAN-SPAM / TCPA | Communications table tracks consent and unsubscribe |
| OAuth 2.0 / OIDC | User authentication for staff and member portal |
| W3C WebAuthn | Passkey support via users table |
| Schema.org | Event and Person fields map to Schema.org types |
| OpenAPI 3.1 | Table structure maps cleanly to REST resource definitions |

---

## Church & Staff

### churches

```sql
CREATE TABLE churches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    campus_type     TEXT NOT NULL DEFAULT 'main' CHECK (campus_type IN (
                        'main', 'satellite', 'online', 'church_plant'
                    )),
    parent_church_id UUID REFERENCES churches(id),  -- for multi-site
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    country_code    TEXT NOT NULL DEFAULT 'US',       -- ISO 3166-1
    currency_code   TEXT NOT NULL DEFAULT 'USD',      -- ISO 4217
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    state_province  TEXT,
    postal_code     TEXT,
    phone           TEXT,
    email           TEXT,
    website_url     TEXT,
    denomination    TEXT,
    config_json     JSONB NOT NULL DEFAULT '{}',
    -- { "service_times": ["09:00", "11:00"],
    --   "check_in_label_format": "dymo_30252",
    --   "giving_funds": ["general", "missions", "building", "benevolence"],
    --   "tax_receipt_rules": "irs_pub_1771",
    --   "fiscal_year_start": "01-01",
    --   "custom_fields": [{ "name": "Campus Preference", "type": "select", ... }],
    --   "stripe_account_id": "acct_...",
    --   "twilio_from": "+1234567890",
    --   "sendgrid_from": "church@example.org" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_churches_parent ON churches (parent_church_id);
CREATE INDEX idx_churches_type ON churches (campus_type);
```

### users

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    church_id       UUID NOT NULL REFERENCES churches(id),
    email           TEXT NOT NULL UNIQUE,
    full_name       TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN (
                        'senior_pastor', 'associate_pastor', 'worship_leader',
                        'admin', 'office_staff', 'children_director',
                        'youth_director', 'group_leader', 'volunteer_coordinator',
                        'finance', 'communications', 'campus_pastor',
                        'system_admin'
                    )),
    phone           TEXT,
    mfa_enabled     BOOLEAN NOT NULL DEFAULT false,
    webauthn_registered BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_church ON users (church_id);
CREATE INDEX idx_users_role ON users (church_id, role);
```

---

## People & Households

### people

```sql
CREATE TABLE people (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    church_id           UUID NOT NULL REFERENCES churches(id),
    household_id        UUID,                -- FK added after households table
    first_name          TEXT NOT NULL,
    last_name           TEXT NOT NULL,
    nickname            TEXT,
    email               TEXT,
    phone_mobile        TEXT,
    phone_home          TEXT,
    date_of_birth       DATE,
    gender              TEXT CHECK (gender IN ('male', 'female', 'non_binary', 'prefer_not_to_say')),
    marital_status      TEXT CHECK (marital_status IN (
                            'single', 'married', 'divorced', 'widowed', 'separated'
                        )),
    -- Membership
    member_status       TEXT NOT NULL DEFAULT 'visitor' CHECK (member_status IN (
                            'visitor', 'first_time_visitor', 'regular_attender',
                            'member', 'inactive', 'transferred', 'deceased'
                        )),
    membership_date     DATE,
    first_visit_date    DATE,
    -- Campus
    primary_campus_id   UUID REFERENCES churches(id),
    -- Milestones
    baptism_date        DATE,
    membership_class_date DATE,
    salvation_date      DATE,
    -- Address
    address_line1       TEXT,
    address_line2       TEXT,
    city                TEXT,
    state_province      TEXT,
    postal_code         TEXT,
    country_code        TEXT DEFAULT 'US',
    -- Photos
    photo_url           TEXT,
    -- Pastoral
    pastoral_notes      TEXT,                -- restricted access
    pastoral_notes_private BOOLEAN NOT NULL DEFAULT true,
    -- Children-specific
    grade               TEXT,
    school              TEXT,
    allergies           TEXT[],
    medical_notes       TEXT,                -- restricted
    custody_alert       TEXT,                -- e.g., "Do not release to father"
    -- Communication preferences
    email_opt_in        BOOLEAN NOT NULL DEFAULT true,
    sms_opt_in          BOOLEAN NOT NULL DEFAULT false,  -- TCPA consent
    sms_consent_date    TIMESTAMPTZ,
    -- Privacy
    gdpr_consent        BOOLEAN,
    gdpr_consent_date   TIMESTAMPTZ,
    do_not_contact      BOOLEAN NOT NULL DEFAULT false,
    -- Portal
    portal_enabled      BOOLEAN NOT NULL DEFAULT false,
    portal_user_id      UUID REFERENCES users(id),
    -- Tags
    tags                TEXT[] DEFAULT '{}',
    custom_fields_json  JSONB NOT NULL DEFAULT '{}',
    -- Engagement tracking
    last_attendance_date DATE,
    last_giving_date    DATE,
    attendance_streak   INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_people_church ON people (church_id);
CREATE INDEX idx_people_household ON people (household_id);
CREATE INDEX idx_people_name ON people (last_name, first_name);
CREATE INDEX idx_people_email ON people (email) WHERE email IS NOT NULL;
CREATE INDEX idx_people_status ON people (church_id, member_status);
CREATE INDEX idx_people_campus ON people (primary_campus_id);
CREATE INDEX idx_people_tags ON people USING GIN (tags);
CREATE INDEX idx_people_first_visit ON people (first_visit_date)
    WHERE member_status = 'first_time_visitor';
```

### households

```sql
CREATE TABLE households (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    church_id       UUID NOT NULL REFERENCES churches(id),
    family_name     TEXT NOT NULL,
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    state_province  TEXT,
    postal_code     TEXT,
    country_code    TEXT DEFAULT 'US',
    phone_home      TEXT,
    primary_contact_id UUID,             -- FK to people
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_households_church ON households (church_id);
CREATE INDEX idx_households_name ON households (family_name);

ALTER TABLE people ADD CONSTRAINT fk_people_household
    FOREIGN KEY (household_id) REFERENCES households(id);
```

---

## Groups & Volunteering

### groups

```sql
CREATE TABLE groups (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    church_id       UUID NOT NULL REFERENCES churches(id),
    name            TEXT NOT NULL,
    group_type      TEXT NOT NULL CHECK (group_type IN (
                        'small_group', 'ministry_team', 'serving_team',
                        'class', 'committee', 'prayer_group',
                        'youth_group', 'mens_group', 'womens_group',
                        'support_group', 'choir', 'band', 'other'
                    )),
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                        'active', 'inactive', 'archived', 'forming'
                    )),
    description     TEXT,
    leader_id       UUID REFERENCES people(id),
    campus_id       UUID REFERENCES churches(id),
    meeting_day     TEXT CHECK (meeting_day IN (
                        'sunday', 'monday', 'tuesday', 'wednesday',
                        'thursday', 'friday', 'saturday'
                    )),
    meeting_time    TIME,
    meeting_location TEXT,
    capacity        INTEGER,
    member_count    INTEGER NOT NULL DEFAULT 0,
    is_public       BOOLEAN NOT NULL DEFAULT true,  -- visible in directory
    enrollment_open BOOLEAN NOT NULL DEFAULT true,
    tags            TEXT[] DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_groups_church ON groups (church_id);
CREATE INDEX idx_groups_type ON groups (church_id, group_type);
CREATE INDEX idx_groups_leader ON groups (leader_id);
CREATE INDEX idx_groups_status ON groups (status) WHERE status = 'active';
CREATE INDEX idx_groups_tags ON groups USING GIN (tags);
```

### volunteer_positions

```sql
CREATE TABLE volunteer_positions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    church_id       UUID NOT NULL REFERENCES churches(id),
    group_id        UUID REFERENCES groups(id),      -- serving team
    person_id       UUID NOT NULL REFERENCES people(id),
    position_name   TEXT NOT NULL,        -- e.g., "Greeter", "Sound Tech", "Nursery"
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                        'active', 'on_break', 'inactive', 'training'
                    )),
    -- Scheduling
    preferred_service TEXT,              -- e.g., "09:00", "11:00"
    frequency       TEXT CHECK (frequency IN (
                        'weekly', 'bi_weekly', 'monthly', 'quarterly', 'as_needed'
                    )),
    last_served_date DATE,
    next_scheduled_date DATE,
    total_times_served INTEGER NOT NULL DEFAULT 0,
    consecutive_weeks INTEGER NOT NULL DEFAULT 0,
    max_consecutive INTEGER DEFAULT 4,   -- burnout prevention
    -- Availability
    available_days  TEXT[] DEFAULT '{}',
    blackout_dates  DATE[] DEFAULT '{}',
    -- Household coordination
    household_id    UUID REFERENCES households(id),
    prefer_same_service_as_household BOOLEAN NOT NULL DEFAULT true,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_volunteer_church ON volunteer_positions (church_id);
CREATE INDEX idx_volunteer_person ON volunteer_positions (person_id);
CREATE INDEX idx_volunteer_group ON volunteer_positions (group_id);
CREATE INDEX idx_volunteer_status ON volunteer_positions (status) WHERE status = 'active';
CREATE INDEX idx_volunteer_next ON volunteer_positions (next_scheduled_date)
    WHERE status = 'active';
CREATE INDEX idx_volunteer_household ON volunteer_positions (household_id);
```

---

## Events & Attendance

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
    registered_count INTEGER NOT NULL DEFAULT 0,
    attended_count  INTEGER NOT NULL DEFAULT 0,
    -- Registration
    registration_open BOOLEAN NOT NULL DEFAULT false,
    registration_deadline TIMESTAMPTZ,
    registration_fee_cents BIGINT DEFAULT 0,
    -- Recurrence
    is_recurring    BOOLEAN NOT NULL DEFAULT false,
    recurrence_rule TEXT,                -- iCal RRULE
    parent_event_id UUID REFERENCES events(id),
    -- Check-in
    check_in_enabled BOOLEAN NOT NULL DEFAULT false,
    -- Worship service specific
    sermon_title    TEXT,
    sermon_speaker  TEXT,
    sermon_notes_url TEXT,
    -- Schema.org
    schema_org_json JSONB,
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
CREATE INDEX idx_events_parent ON events (parent_event_id);
```

### attendance

```sql
CREATE TABLE attendance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL REFERENCES events(id),
    person_id       UUID NOT NULL REFERENCES people(id),
    church_id       UUID NOT NULL REFERENCES churches(id),
    status          TEXT NOT NULL DEFAULT 'registered' CHECK (status IN (
                        'registered', 'attended', 'no_show', 'cancelled'
                    )),
    checked_in_at   TIMESTAMPTZ,
    checked_out_at  TIMESTAMPTZ,
    -- Check-in specific (children)
    is_child_check_in BOOLEAN NOT NULL DEFAULT false,
    security_code   TEXT,                -- pickup code
    label_printed   BOOLEAN NOT NULL DEFAULT false,
    checked_in_by   UUID REFERENCES people(id),  -- parent/guardian
    checked_out_to  UUID REFERENCES people(id),  -- authorized pickup
    allergy_alert_shown BOOLEAN NOT NULL DEFAULT false,
    custody_alert_shown BOOLEAN NOT NULL DEFAULT false,
    room_assignment TEXT,
    -- Volunteer serving
    is_serving      BOOLEAN NOT NULL DEFAULT false,
    position_name   TEXT,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attendance_event ON attendance (event_id);
CREATE INDEX idx_attendance_person ON attendance (person_id);
CREATE INDEX idx_attendance_church ON attendance (church_id, checked_in_at);
CREATE UNIQUE INDEX idx_attendance_unique ON attendance (event_id, person_id);
CREATE INDEX idx_attendance_child ON attendance (is_child_check_in, security_code)
    WHERE is_child_check_in = true;
```

---

## Giving & Finance

### donations

```sql
CREATE TABLE donations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    church_id       UUID NOT NULL REFERENCES churches(id),
    person_id       UUID REFERENCES people(id),      -- null for anonymous
    household_id    UUID REFERENCES households(id),
    amount_cents    BIGINT NOT NULL,
    currency_code   TEXT NOT NULL DEFAULT 'USD',      -- ISO 4217
    fund            TEXT NOT NULL DEFAULT 'general',  -- fund allocation
    donation_type   TEXT NOT NULL CHECK (donation_type IN (
                        'one_time', 'recurring', 'pledge_payment'
                    )),
    payment_method  TEXT NOT NULL CHECK (payment_method IN (
                        'card', 'ach', 'cash', 'check', 'text_to_give',
                        'kiosk', 'stock', 'other'
                    )),
    -- Stripe (PCI-compliant: no card data)
    stripe_payment_id TEXT,              -- pi_xxx
    stripe_charge_id TEXT,               -- ch_xxx
    stripe_subscription_id TEXT,         -- sub_xxx for recurring
    -- Check/cash specifics
    check_number    TEXT,
    batch_id        TEXT,                -- for cash/check entry batches
    -- Status
    status          TEXT NOT NULL DEFAULT 'completed' CHECK (status IN (
                        'pending', 'completed', 'failed', 'refunded', 'disputed'
                    )),
    -- Tax receipt
    tax_deductible  BOOLEAN NOT NULL DEFAULT true,
    receipt_sent    BOOLEAN NOT NULL DEFAULT false,
    receipt_sent_at TIMESTAMPTZ,
    -- Pledge linkage
    pledge_id       UUID,               -- FK added after pledges table
    -- Dates
    donation_date   DATE NOT NULL,
    fiscal_year     TEXT NOT NULL,       -- e.g., "2026"
    -- Memo
    memo            TEXT,
    is_anonymous    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_donations_church ON donations (church_id);
CREATE INDEX idx_donations_person ON donations (person_id);
CREATE INDEX idx_donations_household ON donations (household_id);
CREATE INDEX idx_donations_date ON donations (donation_date);
CREATE INDEX idx_donations_fund ON donations (church_id, fund, donation_date);
CREATE INDEX idx_donations_fiscal ON donations (church_id, fiscal_year);
CREATE INDEX idx_donations_stripe ON donations (stripe_payment_id)
    WHERE stripe_payment_id IS NOT NULL;
CREATE INDEX idx_donations_batch ON donations (batch_id) WHERE batch_id IS NOT NULL;
CREATE INDEX idx_donations_pledge ON donations (pledge_id) WHERE pledge_id IS NOT NULL;
```

### pledges

```sql
CREATE TABLE pledges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    church_id       UUID NOT NULL REFERENCES churches(id),
    person_id       UUID NOT NULL REFERENCES people(id),
    household_id    UUID REFERENCES households(id),
    fund            TEXT NOT NULL DEFAULT 'general',
    total_amount_cents BIGINT NOT NULL,
    currency_code   TEXT NOT NULL DEFAULT 'USD',
    frequency       TEXT NOT NULL CHECK (frequency IN (
                        'one_time', 'weekly', 'bi_weekly', 'monthly',
                        'quarterly', 'annually'
                    )),
    start_date      DATE NOT NULL,
    end_date        DATE,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                        'active', 'completed', 'cancelled', 'paused'
                    )),
    amount_fulfilled_cents BIGINT NOT NULL DEFAULT 0,
    payments_made   INTEGER NOT NULL DEFAULT 0,
    campaign_name   TEXT,                -- e.g., "Building Fund 2026"
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pledges_church ON pledges (church_id);
CREATE INDEX idx_pledges_person ON pledges (person_id);
CREATE INDEX idx_pledges_fund ON pledges (church_id, fund);
CREATE INDEX idx_pledges_status ON pledges (status) WHERE status = 'active';
CREATE INDEX idx_pledges_campaign ON pledges (campaign_name) WHERE campaign_name IS NOT NULL;

ALTER TABLE donations ADD CONSTRAINT fk_donations_pledge
    FOREIGN KEY (pledge_id) REFERENCES pledges(id);
```

---

## Communications

### communications

```sql
CREATE TABLE communications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    church_id       UUID NOT NULL REFERENCES churches(id),
    channel         TEXT NOT NULL CHECK (channel IN ('email', 'sms', 'push')),
    comm_type       TEXT NOT NULL CHECK (comm_type IN (
                        'broadcast', 'follow_up', 'automated_sequence',
                        'event_reminder', 'giving_receipt', 'birthday',
                        'pastoral', 'volunteer_reminder', 'announcement'
                    )),
    status          TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
                        'draft', 'scheduled', 'sending', 'sent', 'failed', 'cancelled'
                    )),
    subject         TEXT,
    body            TEXT NOT NULL,
    -- Targeting
    recipient_filter_json JSONB,
    -- { "member_status": ["member", "regular_attender"],
    --   "tags": ["young_adults"], "campus_id": "...",
    --   "group_id": "...", "custom_query": "..." }
    recipient_count INTEGER NOT NULL DEFAULT 0,
    delivered_count INTEGER NOT NULL DEFAULT 0,
    opened_count    INTEGER NOT NULL DEFAULT 0,
    clicked_count   INTEGER NOT NULL DEFAULT 0,
    unsubscribed_count INTEGER NOT NULL DEFAULT 0,
    bounced_count   INTEGER NOT NULL DEFAULT 0,
    -- Scheduling
    scheduled_at    TIMESTAMPTZ,
    sent_at         TIMESTAMPTZ,
    -- Provider
    provider        TEXT,                -- 'sendgrid', 'twilio', etc.
    provider_id     TEXT,                -- external message/campaign ID
    -- Compliance
    includes_unsubscribe BOOLEAN NOT NULL DEFAULT true,  -- CAN-SPAM / RFC 8058
    sent_by_id      UUID REFERENCES users(id),
    -- AI
    ai_generated    BOOLEAN NOT NULL DEFAULT false,
    ai_model_id     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_communications_church ON communications (church_id);
CREATE INDEX idx_communications_type ON communications (comm_type);
CREATE INDEX idx_communications_status ON communications (status);
CREATE INDEX idx_communications_scheduled ON communications (scheduled_at)
    WHERE status = 'scheduled';
```

---

## AI & Audit

### ai_suggestions

```sql
CREATE TABLE ai_suggestions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    church_id       UUID NOT NULL REFERENCES churches(id),
    person_id       UUID REFERENCES people(id),
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
    accepted_by_id  UUID REFERENCES users(id),
    accepted_at     TIMESTAMPTZ,
    feedback        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_suggestions_church ON ai_suggestions (church_id);
CREATE INDEX idx_ai_suggestions_person ON ai_suggestions (person_id);
CREATE INDEX idx_ai_suggestions_type ON ai_suggestions (suggestion_type);
CREATE INDEX idx_ai_suggestions_pending ON ai_suggestions (status) WHERE status = 'pending';
```

### audit_log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    church_id       UUID NOT NULL REFERENCES churches(id),
    user_id         UUID REFERENCES users(id),
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
CREATE INDEX idx_audit_log_financial ON audit_log (financial_data, created_at)
    WHERE financial_data = true;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Church & Staff | 2 | Multi-site with self-referencing campus hierarchy |
| People & Households | 2 | Member lifecycle with pastoral notes and privacy |
| Groups & Volunteering | 2 | Groups with membership; volunteer scheduling with burnout prevention |
| Events & Attendance | 2 | Events with iCal; attendance with children's check-in |
| Giving & Finance | 2 | Donations (Stripe-backed) and pledges with fund accounting |
| Communications | 1 | Email/SMS/push with CAN-SPAM compliance |
| AI & Audit | 2 | Pastoral AI suggestions; partitioned audit log |
| **Total** | **13** | |

---

## Key Design Decisions

1. **People and households as separate tables** — a person belongs to one household via `household_id`. This supports household-level giving statements (IRS requires per-household aggregation for married couples filing jointly) and household-level volunteer coordination.

2. **Attendance doubles as check-in** — the `attendance` table handles both regular service attendance and children's check-in. Child check-ins add security codes, allergy alerts, custody alerts, and room assignments. This avoids a separate check-in table while keeping the child-safety fields available.

3. **Fund as a text column, not a separate table** — giving funds (general, missions, building, benevolence) are configured in `churches.config_json` and referenced as text on donations. This keeps fund management simple for small churches while supporting multi-fund allocation.

4. **Volunteer burnout prevention** — `volunteer_positions` tracks `consecutive_weeks` and `max_consecutive` to detect and prevent volunteer burnout. The AI suggestion system uses these fields for `burnout_warning` suggestions.

5. **Stripe-only payment references** — no card numbers, CVVs, or bank account numbers are stored. All payment data is Stripe references (`stripe_payment_id`, `stripe_charge_id`), keeping the system PCI DSS SAQ-A compliant.

6. **Pastoral notes with access control** — `people.pastoral_notes` is flagged `pastoral_notes_private = true` by default. The audit log tracks `pastoral_data = true` for any access to these fields, enabling churches to demonstrate responsible data handling.

7. **Multi-site via self-referencing churches** — satellite campuses reference the main church via `parent_church_id`. People have a `primary_campus_id` but can attend events at any campus. Financial reporting can roll up to the parent church.

8. **Communication compliance built in** — `sms_opt_in` with `sms_consent_date` satisfies TCPA, `email_opt_in` satisfies CAN-SPAM, and `gdpr_consent` with `gdpr_consent_date` satisfies GDPR. The communications table enforces `includes_unsubscribe = true`.

9. **Tags as TEXT[] with GIN index** — tags on people, groups, and events use PostgreSQL arrays with GIN indexes, enabling fast containment queries (`WHERE tags @> ARRAY['young_adults']`) without a junction table.

10. **Fiscal year on donations** — `fiscal_year` enables year-end contribution statements without date-range calculations, supporting churches with non-calendar fiscal years.
