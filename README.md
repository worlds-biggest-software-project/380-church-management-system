# Church Management System (ChMS)

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform unifying membership, giving, small groups, events, and volunteer coordination for churches and faith communities.

Church Management System (ChMS) is a unified platform that manages the full congregational lifecycle — from first visit through deepening involvement, giving, group participation, and volunteer service. It is built for churches and faith communities whose small administrative teams currently juggle spreadsheets, accounting software, group rosters, and consumer sign-up apps. The core problem it solves is fragmentation: pastoral staff lack a unified view of congregational engagement, volunteers are under- or over-committed, and giving statements require manual reconciliation at year end.

---

## Why Church Management System (ChMS)?

- The ChMS market is dominated by closed-source SaaS (Breeze, Planning Center, Pushpay, Subsplash, FellowshipOne), with pricing ranging from $9/month for entry-level platforms to opaque enterprise pricing for multi-site churches.
- Existing open-source alternatives have UX gaps: Rock RMS requires Windows/.NET expertise and has a steep learning curve; ChurchCRM has a dated Bootstrap CRUD UI, minimal mobile experience, and basic reporting.
- Modular incumbents like Planning Center accumulate cost quickly when adopting multiple modules and force redundant data entry across them.
- Multi-site campus consolidation, pastoral workflow automation, and donor-conversion mobile flows are gated behind enterprise-tier pricing, leaving mid-sized churches underserved.
- No incumbent offers a native AI/automation layer for pastoral care prompting, first-time-visitor follow-up, volunteer burnout prediction, or natural-language reporting.

---

## Key Features

### Membership & People Management

- Member and household profiles with photos, custom fields, and tags
- Relationship mapping across families and small group connections
- Attendance history and group involvement tracking
- Custom fields for pastoral notes and milestones (baptism, membership class)
- Role-based access control with pastoral note privacy

### Giving & Financial Management

- Online giving via Stripe with one-time and recurring donations
- Multiple giving methods (card, ACH, text-to-give, kiosk)
- Fund allocation and year-end contribution statements
- Pledges and tax-compliant substantiation (IRS rules in the US)
- Integration paths to accounting tools (QuickBooks, Xero)

### Groups, Volunteers & Events

- Small group and ministry team directory with roster management
- Volunteer serving positions, rotation scheduling, and self-service availability
- Household scheduling to coordinate family members on the same service days
- Event creation, RSVP, capacity management, and post-event reporting
- Children's check-in with label printing, allergy and custody alerts, and pickup codes

### Communications & Reporting

- Bulk email and SMS to filtered membership segments
- Automated follow-up sequences for first-time visitors
- Push notifications via mobile-friendly member portal (PWA)
- Standard reports for attendance, giving, group health, and volunteer coverage
- Mobile-first member experience for giving, sign-up, and rosters

---

## AI-Native Advantage

AI capabilities are first-class rather than bolted on. The platform drafts personalised first-time-visitor follow-up messages, suggests volunteer slots based on availability and rotation history, and detects at-risk members from declining attendance or giving patterns for pastoral outreach. Natural-language search and report querying ("show me first-time givers from last quarter who haven't been contacted") put data within reach of non-technical staff. Sermon transcription can automatically generate small group discussion guides, and prayer requests are auto-categorised and tagged.

---

## Tech Stack & Deployment

The platform is designed as a self-hostable, cloud-deployable system with a mobile-friendly PWA member portal. Online giving is Stripe-backed. A public REST API plus webhooks supports ecosystem integrations (Mailchimp, QuickBooks, Zapier, Twilio). Multi-site architecture supports shared directories and consolidated giving reporting while preserving campus-level visibility. Pastoral data is governed by strict access controls aligned with GDPR/CCPA where applicable, and giving records meet audit-trail and fund-accounting requirements needed for tax-deductible contribution statements.

---

## Market Context

The ChMS market is mature and competitive. Entry-level platforms such as ChurchTrac start near $9/month, mid-market platforms like Breeze offer flat-rate pricing, and enterprise platforms (Pushpay, FellowshipOne, Ministry Platform) use opaque custom pricing for large multi-site churches. Primary buyers are church administrators — often part-time or volunteer staff — with low technical sophistication and high cost sensitivity, making ease of use a genuine competitive differentiator.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
