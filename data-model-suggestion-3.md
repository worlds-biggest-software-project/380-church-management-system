# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Church Management System (ChMS) · Created: 2026-05-26

## Philosophy

A church management system is fundamentally about relationships and engagement over time: when did someone first visit, how has their involvement grown, when did giving patterns change, who connected them to a small group? This model captures every interaction as an immutable event — visits, check-ins, donations, group joins, volunteer serves, pastoral contacts, and communications — and derives current state by projecting these events into read models optimised for pastoral dashboards, giving reports, volunteer schedules, and group health.

While churches rarely face the regulatory audit requirements of healthcare or corrections, the event-sourced approach offers two compelling advantages for this domain: (1) engagement analytics become trivial — "show me the journey from first visit to regular attender to small group member to volunteer" is a natural event-stream query rather than a complex multi-table join across dates; and (2) AI features like at-risk member detection, visitor follow-up timing, and volunteer burnout prediction can be trained directly on anonymised event streams rather than requiring ETL from multiple tables.

The giving event stream also provides a natural audit trail for financial transparency — every donation, refund, and statement generation is an immutable event, giving treasurers and auditors a complete financial history without relying on mutable ledger entries.

**Best for:** Churches prioritising engagement analytics, AI-powered pastoral care, and comprehensive giving audit trails, especially multi-site churches where event streams from different campuses need to merge into unified member journeys.

**Trade-offs:**
- (+) Complete member engagement journey is a natural event replay
- (+) AI training on anonymised event streams for pastoral care patterns
- (+) Financial audit trail is immutable by design
- (+) Multi-site event streams merge naturally for consolidated reporting
- (+) Temporal queries ("what was attendance like before and after the new service time?") are straightforward
- (-) Read models must be maintained and rebuilt when dashboards change
- (-) Year-end giving statements require projection logic rather than simple SQL
- (-) Higher storage than a mutable model for active churches
- (-) Eventual consistency means a donation may take moments to appear on the dashboard

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IRS Publication 1771 | Giving events carry fund, amount, method, deductibility — projected into statement read model |
| PCI DSS v4.0 | Payment events reference Stripe IDs only; no card data in events |
| ISO 4217 | Currency code on every giving event |
| RFC 5545 (iCalendar) | Event stream data supports iCal reconstruction |
| GDPR / CCPA | Privacy events (consent granted/revoked) are first-class with timestamps |
| CAN-SPAM / TCPA | Communication consent events with opt-in/opt-out timestamps |
| CloudEvents 1.0 | Event envelope follows CloudEvents specification |
| OAuth 2.0 / OIDC | Actor authentication context on every event |

---

## Infrastructure Tables

### event_store

```sql
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL CHECK (stream_type IN (
                        'church', 'person', 'household', 'group',
                        'event', 'giving', 'volunteer',
                        'communication', 'ai', 'config'
                    )),
    stream_id       UUID NOT NULL,
    sequence_num    BIGINT NOT NULL,
    event_type      TEXT NOT NULL,
    event_data      JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- CloudEvents envelope
    ce_source       TEXT NOT NULL,
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',
    ce_type         TEXT NOT NULL,
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Actor
    actor_id        UUID,
    actor_type      TEXT NOT NULL DEFAULT 'user' CHECK (actor_type IN (
                        'user', 'system', 'ai', 'integration', 'cron',
                        'member_portal', 'kiosk', 'stripe_webhook',
                        'twilio_webhook', 'sendgrid_webhook',
                        'check_in_station'
                    )),
    actor_role      TEXT,
    -- Access control
    pastoral_data   BOOLEAN NOT NULL DEFAULT false,
    financial_data  BOOLEAN NOT NULL DEFAULT false,
    -- Correlation
    correlation_id  UUID,
    causation_id    UUID,
    church_id       UUID NOT NULL,
    campus_id       UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, sequence_num)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_events_stream ON event_store (stream_type, stream_id, sequence_num);
CREATE INDEX idx_events_type ON event_store (event_type, created_at);
CREATE INDEX idx_events_actor ON event_store (actor_id, created_at);
CREATE INDEX idx_events_church ON event_store (church_id, created_at);
CREATE INDEX idx_events_campus ON event_store (campus_id, created_at) WHERE campus_id IS NOT NULL;
CREATE INDEX idx_events_correlation ON event_store (correlation_id) WHERE correlation_id IS NOT NULL;
CREATE INDEX idx_events_pastoral ON event_store (pastoral_data, created_at) WHERE pastoral_data = true;
CREATE INDEX idx_events_financial ON event_store (financial_data, created_at) WHERE financial_data = true;
```

### stream_snapshots

```sql
CREATE TABLE stream_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL,
    stream_id       UUID NOT NULL,
    sequence_num    BIGINT NOT NULL,
    snapshot_data   JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_snapshots_stream ON stream_snapshots (stream_type, stream_id, sequence_num);
```

### projection_checkpoints

```sql
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_sequence   BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Taxonomy

### Church Stream
| Event | Key Fields |
|-------|------------|
| `church.created` | name, campus_type, country_code, currency_code |
| `church.config_updated` | giving_funds, service_times, custom_fields |
| `church.campus_added` | campus_id, name, type |
| `church.staff_added` | user_id, role, name |
| `church.staff_role_changed` | user_id, old_role, new_role |
| `church.integration_configured` | provider (stripe/twilio/sendgrid), config |
| `church.group_created` | group_id, name, type, leader_id |
| `church.group_updated` | group_id, changes |
| `church.group_archived` | group_id, reason |

### Person Stream
| Event | Key Fields |
|-------|------------|
| `person.registered` | name, email, phone, member_status |
| `person.first_visit` | date, campus_id, invited_by |
| `person.status_changed` | old_status, new_status (visitor→member) |
| `person.milestone_reached` | type (baptism/membership_class/salvation), date |
| `person.profile_updated` | changed_fields |
| `person.photo_uploaded` | photo_url |
| `person.tag_added` | tag |
| `person.tag_removed` | tag |
| `person.custom_field_set` | field_name, value |
| `person.campus_changed` | old_campus, new_campus |
| `person.transferred` | to_church, reason |
| `person.marked_inactive` | reason, last_attendance |
| `person.portal_enabled` | user_id |

### Household Stream
| Event | Key Fields |
|-------|------------|
| `household.created` | family_name, address |
| `household.member_added` | person_id, relationship |
| `household.member_removed` | person_id, reason |
| `household.address_updated` | old_address, new_address |
| `household.primary_contact_changed` | person_id |

### Group Stream
| Event | Key Fields |
|-------|------------|
| `group.member_joined` | person_id, role (member/leader/co_leader) |
| `group.member_left` | person_id, reason |
| `group.leader_changed` | old_leader_id, new_leader_id |
| `group.meeting_held` | date, attendees, topic |
| `group.meeting_cancelled` | date, reason |

### Event Stream
| Event | Key Fields |
|-------|------------|
| `event.created` | name, type, date, campus_id, capacity |
| `event.registration_opened` | deadline, fee_cents |
| `event.person_registered` | person_id |
| `event.person_attended` | person_id, checked_in_at |
| `event.person_no_show` | person_id |
| `event.child_checked_in` | child_id, security_code, room, checked_in_by, allergy_alert, custody_alert |
| `event.child_checked_out` | child_id, checked_out_to, checked_out_at |
| `event.volunteer_served` | person_id, position |
| `event.completed` | total_attended, total_first_time, total_children |
| `event.sermon_recorded` | title, speaker, notes_url |

### Giving Stream
| Event | Key Fields |
|-------|------------|
| `giving.donation_received` | person_id, amount_cents, currency, fund, method, stripe_id (financial_data=true) |
| `giving.recurring_started` | person_id, amount_cents, frequency, fund, stripe_sub_id |
| `giving.recurring_updated` | person_id, old_amount, new_amount |
| `giving.recurring_cancelled` | person_id, reason |
| `giving.refunded` | donation_event_id, amount_cents, reason |
| `giving.pledge_created` | person_id, campaign, total_cents, frequency, start, end |
| `giving.pledge_payment` | person_id, pledge_id, amount_cents |
| `giving.pledge_completed` | person_id, pledge_id |
| `giving.pledge_cancelled` | person_id, pledge_id, reason |
| `giving.batch_entered` | batch_id, donations_count, total_cents |
| `giving.statement_generated` | person_id, fiscal_year, total_cents |
| `giving.statement_sent` | person_id, fiscal_year, method (email/mail) |

### Volunteer Stream
| Event | Key Fields |
|-------|------------|
| `volunteer.position_assigned` | person_id, group_id, position, frequency |
| `volunteer.served` | person_id, event_id, position, date |
| `volunteer.availability_updated` | person_id, available_days, blackout_dates |
| `volunteer.scheduled` | person_id, event_id, position, date |
| `volunteer.declined` | person_id, event_id, reason |
| `volunteer.break_started` | person_id, position, reason, return_date |
| `volunteer.break_ended` | person_id, position |
| `volunteer.burnout_flagged` | person_id, consecutive_weeks, total_recent |
| `volunteer.position_deactivated` | person_id, position, reason |

### Communication Stream
| Event | Key Fields |
|-------|------------|
| `communication.email_opt_in` | person_id |
| `communication.email_opt_out` | person_id |
| `communication.sms_consent_given` | person_id, consent_date |
| `communication.sms_consent_revoked` | person_id |
| `communication.campaign_sent` | channel, type, recipient_count, subject |
| `communication.email_delivered` | person_id, campaign_id |
| `communication.email_opened` | person_id, campaign_id |
| `communication.email_bounced` | person_id, campaign_id, reason |
| `communication.sms_delivered` | person_id, campaign_id |
| `communication.push_sent` | person_id, title |
| `communication.follow_up_step_sent` | person_id, step_number, channel |
| `communication.follow_up_completed` | person_id, total_steps |

### AI Stream
| Event | Key Fields |
|-------|------------|
| `ai.suggestion_generated` | type, person_id, confidence, explanation |
| `ai.suggestion_accepted` | suggestion_id, accepted_by, modifications |
| `ai.suggestion_rejected` | suggestion_id, rejected_by, reason |
| `ai.visitor_follow_up_drafted` | person_id, message, channel |
| `ai.at_risk_member_detected` | person_id, risk_factors, confidence |
| `ai.burnout_warning_raised` | person_id, position, consecutive_weeks |
| `ai.sermon_guide_generated` | sermon_event_id, guide_text |
| `ai.prayer_categorised` | person_id, category, request_summary |
| `ai.segment_suggested` | criteria, member_count, purpose |

---

## Read Models

### rm_member_360

```sql
CREATE TABLE rm_member_360 (
    person_id           UUID PRIMARY KEY,
    church_id           UUID NOT NULL,
    full_name           TEXT NOT NULL,
    email               TEXT,
    phone               TEXT,
    member_status       TEXT NOT NULL,
    primary_campus      TEXT,
    tags                TEXT[] NOT NULL DEFAULT '{}',
    -- Household
    household_id        UUID,
    family_name         TEXT,
    household_members   JSONB NOT NULL DEFAULT '[]',
    -- Milestones
    first_visit_date    DATE,
    membership_date     DATE,
    milestones          JSONB NOT NULL DEFAULT '{}',
    -- Engagement
    last_attendance     DATE,
    attendance_streak   INTEGER NOT NULL DEFAULT 0,
    services_ytd        INTEGER NOT NULL DEFAULT 0,
    events_ytd          INTEGER NOT NULL DEFAULT 0,
    -- Groups
    groups              JSONB NOT NULL DEFAULT '[]',
    -- [{ group_id, name, type, role, joined }]
    -- Giving
    last_giving_date    DATE,
    giving_ytd_cents    BIGINT NOT NULL DEFAULT 0,
    giving_by_fund      JSONB NOT NULL DEFAULT '{}',
    has_recurring       BOOLEAN NOT NULL DEFAULT false,
    active_pledges      JSONB NOT NULL DEFAULT '[]',
    -- Volunteering
    volunteer_positions JSONB NOT NULL DEFAULT '[]',
    -- [{ position, group, status, last_served, next_scheduled, consecutive }]
    total_times_served  INTEGER NOT NULL DEFAULT 0,
    -- Pastoral
    pastoral_notes      TEXT,
    last_pastoral_contact DATE,
    care_requests       JSONB NOT NULL DEFAULT '[]',
    prayer_requests     JSONB NOT NULL DEFAULT '[]',
    -- Communication
    email_opt_in        BOOLEAN NOT NULL DEFAULT true,
    sms_opt_in          BOOLEAN NOT NULL DEFAULT false,
    follow_up_active    BOOLEAN NOT NULL DEFAULT false,
    follow_up_step      INTEGER NOT NULL DEFAULT 0,
    -- AI
    pending_suggestions INTEGER NOT NULL DEFAULT 0,
    risk_level          TEXT,            -- AI-detected engagement risk
    -- Check-in (children)
    allergies           TEXT[],
    custody_alert       TEXT,
    -- Metadata
    last_event_at       TIMESTAMPTZ NOT NULL,
    last_event_id       UUID NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_member_church ON rm_member_360 (church_id);
CREATE INDEX idx_rm_member_status ON rm_member_360 (church_id, member_status);
CREATE INDEX idx_rm_member_name ON rm_member_360 (full_name);
CREATE INDEX idx_rm_member_email ON rm_member_360 (email) WHERE email IS NOT NULL;
CREATE INDEX idx_rm_member_tags ON rm_member_360 USING GIN (tags);
CREATE INDEX idx_rm_member_household ON rm_member_360 (household_id);
CREATE INDEX idx_rm_member_risk ON rm_member_360 (risk_level)
    WHERE risk_level IS NOT NULL;
```

### rm_giving_dashboard

```sql
CREATE TABLE rm_giving_dashboard (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    church_id           UUID NOT NULL,
    campus_id           UUID,
    period_type         TEXT NOT NULL CHECK (period_type IN (
                            'weekly', 'monthly', 'quarterly', 'yearly'
                        )),
    period_key          TEXT NOT NULL,    -- e.g., '2026-W21', '2026-05', '2026-Q2', '2026'
    -- Totals
    total_cents         BIGINT NOT NULL DEFAULT 0,
    donation_count      INTEGER NOT NULL DEFAULT 0,
    unique_givers       INTEGER NOT NULL DEFAULT 0,
    -- By fund
    by_fund             JSONB NOT NULL DEFAULT '{}',
    -- { "general": 1200000, "missions": 350000, ... }
    -- By method
    by_method           JSONB NOT NULL DEFAULT '{}',
    -- { "card": 800000, "ach": 500000, "cash": 150000, "check": 100000 }
    -- Trends
    first_time_givers   INTEGER NOT NULL DEFAULT 0,
    recurring_total     BIGINT NOT NULL DEFAULT 0,
    recurring_count     INTEGER NOT NULL DEFAULT 0,
    average_gift_cents  BIGINT NOT NULL DEFAULT 0,
    -- Pledges
    pledges_active      INTEGER NOT NULL DEFAULT 0,
    pledges_fulfilled_cents BIGINT NOT NULL DEFAULT 0,
    pledges_outstanding_cents BIGINT NOT NULL DEFAULT 0,
    -- Year-end
    statements_generated INTEGER NOT NULL DEFAULT 0,
    statements_sent     INTEGER NOT NULL DEFAULT 0,
    -- Metadata
    last_event_at       TIMESTAMPTZ NOT NULL,
    last_event_id       UUID NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_rm_giving_period ON rm_giving_dashboard (church_id, campus_id, period_type, period_key);
CREATE INDEX idx_rm_giving_church ON rm_giving_dashboard (church_id, period_type);
```

### rm_group_health

```sql
CREATE TABLE rm_group_health (
    group_id            UUID PRIMARY KEY,
    church_id           UUID NOT NULL,
    group_name          TEXT NOT NULL,
    group_type          TEXT NOT NULL,
    leader_name         TEXT,
    status              TEXT NOT NULL,
    -- Membership
    member_count        INTEGER NOT NULL DEFAULT 0,
    capacity            INTEGER,
    members             JSONB NOT NULL DEFAULT '[]',
    -- [{ person_id, name, role, joined, last_attended }]
    -- Activity
    meetings_held_ytd   INTEGER NOT NULL DEFAULT 0,
    avg_attendance      NUMERIC(5, 2),
    avg_attendance_pct  NUMERIC(5, 2),
    last_meeting_date   DATE,
    -- Growth
    members_joined_qtd  INTEGER NOT NULL DEFAULT 0,
    members_left_qtd    INTEGER NOT NULL DEFAULT 0,
    -- Metadata
    last_event_at       TIMESTAMPTZ NOT NULL,
    last_event_id       UUID NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_group_church ON rm_group_health (church_id);
CREATE INDEX idx_rm_group_type ON rm_group_health (church_id, group_type);
```

### rm_event_attendance

```sql
CREATE TABLE rm_event_attendance (
    event_id            UUID PRIMARY KEY,
    church_id           UUID NOT NULL,
    campus_id           UUID,
    event_name          TEXT NOT NULL,
    event_type          TEXT NOT NULL,
    event_date          TIMESTAMPTZ NOT NULL,
    -- Counts
    total_attended      INTEGER NOT NULL DEFAULT 0,
    total_registered    INTEGER NOT NULL DEFAULT 0,
    total_first_time    INTEGER NOT NULL DEFAULT 0,
    total_children      INTEGER NOT NULL DEFAULT 0,
    total_volunteers    INTEGER NOT NULL DEFAULT 0,
    -- Check-in
    children_checked_in INTEGER NOT NULL DEFAULT 0,
    children_checked_out INTEGER NOT NULL DEFAULT 0,
    allergy_alerts      INTEGER NOT NULL DEFAULT 0,
    -- Worship
    sermon_title        TEXT,
    sermon_speaker      TEXT,
    -- Comparison
    vs_previous_week_pct NUMERIC(5, 2),
    vs_same_week_last_year_pct NUMERIC(5, 2),
    -- Metadata
    last_event_at       TIMESTAMPTZ NOT NULL,
    last_event_id       UUID NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_attendance_church ON rm_event_attendance (church_id, event_date);
CREATE INDEX idx_rm_attendance_campus ON rm_event_attendance (campus_id, event_date);
CREATE INDEX idx_rm_attendance_type ON rm_event_attendance (event_type, event_date);
```

### rm_volunteer_board

```sql
CREATE TABLE rm_volunteer_board (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    church_id           UUID NOT NULL,
    campus_id           UUID,
    event_date          DATE NOT NULL,
    service_time        TEXT,
    -- Coverage
    positions_needed    INTEGER NOT NULL DEFAULT 0,
    positions_filled    INTEGER NOT NULL DEFAULT 0,
    coverage_pct        NUMERIC(5, 2),
    -- By position
    positions           JSONB NOT NULL DEFAULT '[]',
    -- [{ position: "Greeter", needed: 4, filled: 3,
    --    scheduled: [{ person_id, name, status }],
    --    gaps: 1 }]
    -- Burnout alerts
    burnout_warnings    JSONB NOT NULL DEFAULT '[]',
    -- [{ person_id, name, position, consecutive_weeks }]
    -- Household coordination
    household_conflicts JSONB NOT NULL DEFAULT '[]',
    -- [{ household_id, family_name, issue: "different services" }]
    -- Metadata
    last_event_at       TIMESTAMPTZ NOT NULL,
    last_event_id       UUID NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_volunteer_church ON rm_volunteer_board (church_id, event_date);
CREATE INDEX idx_rm_volunteer_campus ON rm_volunteer_board (campus_id, event_date);
CREATE INDEX idx_rm_volunteer_upcoming ON rm_volunteer_board (event_date)
    WHERE event_date >= CURRENT_DATE;
```

---

## Example Event Replay

### Reconstruct a member's engagement journey

```sql
SELECT ce_time, event_type,
       event_data->>'summary' AS summary,
       stream_type
FROM event_store
WHERE church_id = $1
  AND (
    (stream_type = 'person' AND stream_id = $2)
    OR (stream_type = 'giving' AND event_data->>'person_id' = $2::text)
    OR (stream_type = 'volunteer' AND event_data->>'person_id' = $2::text)
    OR (stream_type = 'event' AND event_data->>'person_id' = $2::text)
    OR (stream_type = 'group' AND event_data->>'person_id' = $2::text)
  )
ORDER BY ce_time ASC;
```

### Financial audit trail for a fiscal year

```sql
SELECT ce_time, event_type,
       (event_data->>'amount_cents')::bigint AS amount,
       event_data->>'fund' AS fund,
       event_data->>'payment_method' AS method,
       event_data->>'stripe_payment_id' AS stripe_id,
       actor_type
FROM event_store
WHERE stream_type = 'giving'
  AND church_id = $1
  AND financial_data = true
  AND ce_time BETWEEN '2026-01-01' AND '2026-12-31'
ORDER BY ce_time;
```

### Communication consent timeline (GDPR/TCPA audit)

```sql
SELECT ce_time, event_type,
       event_data->>'person_id' AS person_id,
       event_data->>'channel' AS channel,
       actor_type
FROM event_store
WHERE stream_type = 'communication'
  AND church_id = $1
  AND event_type IN (
      'communication.email_opt_in', 'communication.email_opt_out',
      'communication.sms_consent_given', 'communication.sms_consent_revoked'
  )
ORDER BY ce_time;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Infrastructure | 3 | Partitioned event store, snapshots, projection checkpoints |
| Read Models | 5 | Member 360, giving dashboard, group health, event attendance, volunteer board |
| **Total** | **8** | |

---

## Key Design Decisions

1. **10 stream types covering the congregational lifecycle** — church, person, household, group, event, giving, volunteer, communication, ai, config. The giving stream is separate from person because financial events need independent audit and access control.

2. **Engagement journey as natural replay** — the member's path from first visit through membership, group participation, giving, and volunteering is a chronological event replay. This is the data that AI models use for at-risk detection and pastoral care prompting.

3. **Financial events with `financial_data = true`** — every giving event carries this flag, enabling treasurers to query the complete financial audit trail with a single index scan. Year-end statements are projections of these events filtered by fiscal year.

4. **Communication consent as events** — opt-in/opt-out for email, SMS, and push are immutable events with timestamps. This provides the consent audit trail required by GDPR, CCPA, CAN-SPAM, and TCPA without relying on mutable boolean fields.

5. **Volunteer burnout detection from event stream** — `volunteer.served` events are counted by the `rm_volunteer_board` projection to calculate consecutive weeks and raise burnout warnings. The AI stream generates `ai.burnout_warning_raised` events that appear as suggestions to volunteer coordinators.

6. **Multi-campus event streams** — every event carries both `church_id` and `campus_id`, enabling campus-level dashboards (attendance, giving, volunteers at this campus) and consolidated church-wide reporting from the same event store.

7. **Stripe/Twilio/SendGrid as actor types** — webhook-generated events from payment processors and communication providers carry their own actor types (`stripe_webhook`, `twilio_webhook`, `sendgrid_webhook`), providing complete provenance for externally originated events.

8. **Giving dashboard as multi-granularity projection** — `rm_giving_dashboard` stores weekly, monthly, quarterly, and yearly aggregations in the same table, keyed by `period_type` and `period_key`. This supports the "giving trends" view that church treasurers and boards review regularly.

9. **Correlation chains for visitor follow-up** — `correlation_id` links a visitor's first-visit event to the AI-generated follow-up suggestion, to the communication sent, to the email opened, to the second visit. This chain is the engagement funnel that pastoral staff care about most.

10. **Pastoral data flag** — events touching pastoral notes, care requests, and prayer requests carry `pastoral_data = true`. Read model projections that include pastoral data can enforce access control at the projection level, ensuring only authorized staff see sensitive pastoral information.
