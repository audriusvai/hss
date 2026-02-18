# ManoLaikas — Client Retention & Scheduling for Beauty Professionals

## Problem Statement

Independent beauty professionals (hairdressers, manicurists, cosmetologists) lose revenue because clients forget to rebook, cancellations leave empty slots, and manual scheduling via text messages wastes hours each week. Without an administrator or system, these solo professionals have no reliable way to keep their calendar full and their clients returning on time.

## Key Hypothesis

We believe automated SMS-based recall reminders and a self-service booking link will increase client retention and reduce empty slots for solo beauty professionals.
We'll know we're right when professionals report measurably fewer no-shows and more consistent rebookings during a hands-on pilot.

## Users

**Primary User**: Self-employed beauty specialist (hairdresser, manicurist, cosmetologist) who manages their own client flow without an administrator. Typically has 50-200 regular clients.

**Job to Be Done**: When I finish a client's appointment, I want their next visit to happen automatically at the right interval, so I can keep my calendar full without chasing people over text.

**Non-Users**: Salon chains with receptionists/admins, clients searching for new professionals (this is NOT a marketplace), large multi-location businesses.

## Solution

A mobile-first web application that gives each professional a public booking link (synced with their personal calendar), automatically tracks each client's visit cycle, and sends SMS reminders when it's time to rebook. When cancellations happen, a one-tap "burning slot" feature notifies waitlisted clients instantly. The tool replaces manual text coordination with hands-off automation while keeping the personal specialist-client relationship intact.

### MVP Scope

| Priority | Capability | Rationale |
|----------|------------|-----------|
| Must | **Dynamic calendar** — public booking link showing only available slots, synced with Google/iCloud calendar | Core interaction point; eliminates back-and-forth scheduling messages |
| Must | **Smart Recall (Loyalty algorithm)** — track each client's visit cycle, auto-send SMS 3 days before their next expected visit if not yet booked | Primary retention mechanism; the key differentiator |
| Must | **SMS automation** — appointment reminders (day-before) + "return" messages with direct booking link | Reduces no-shows and drives rebookings |
| Must | **Client card** — visit history with photos (last cut/color) and formulas/notes | Professionals need context for each client; replaces paper notes |
| Should | **Deposit function** — request small deposit from new clients at booking | No-show protection; important but not blocking for initial validation |
| Should | **Burning slot** — one-tap SMS blast to waitlisted clients when a cancellation opens a slot | High-value for revenue recovery; depends on SMS infra being solid first |
| Won't | **Marketplace/discovery** — clients searching for new professionals | Explicitly not a catalog; focus is on existing relationships |
| Won't | **Multi-location / team management** — admin dashboards for salons | Target is solo professionals only |
| Won't | **Full payment processing** — beyond simple deposit collection | Keeps scope manageable; professionals handle payments themselves |

## Success Metrics

| Metric | Target | How Measured |
|--------|--------|--------------|
| Client rebooking rate | Increase vs. baseline (target TBD after pilot) | Compare rebook frequency before/after adoption per professional |
| Empty slot reduction | Fewer unfilled hours per week | Professional self-report + calendar occupancy data |
| No-show rate | Reduction vs. baseline | Track booked vs. attended appointments |
| SMS recall conversion | >20% of recall messages result in a booking | Clicks on booking link from recall SMS |
| Professional retention | Professionals keep using it after 1 month | Active usage tracking |

## Decisions

- **SMS provider**: Local Lithuanian provider (to be selected later; not required for MVP build)
- **Pricing model**: 15 EUR/month subscription + SMS fees passed through to professional
- **Calendar sync**: Two-way sync with Google/iCloud (preferred)
- **Photo storage**: Hosted on the web application server
- **Deposit payments**: Paysera (Lithuanian payment provider)
- **Target market**: Lithuania only. Start in English, transition to Lithuanian language
- **Pilot size**: 2 professionals for initial validation

## Open Questions

- [ ] Legal/GDPR — client consent flow for SMS marketing in Lithuania/EU
- [ ] Specific local SMS provider selection and per-message cost
- [ ] Photo storage limits per professional (disk budget on server)

## Implementation Phases

| # | Phase | Description | Status | Depends |
|---|-------|-------------|--------|---------|
| 1 | **Core booking engine** | Dynamic calendar with public booking link + Google/iCloud calendar sync | pending | - |
| 2 | **Client management** | Client cards with visit history, photos, notes, and visit cycle tracking | pending | 1 |
| 3 | **SMS automation** | Appointment reminders (day-before) + Smart Recall cycle-based messages | pending | 1, 2 |
| 4 | **Burning slot** | Last-minute cancellation notification to waitlisted clients | pending | 3 |
| 5 | **Deposit system** | Payment collection for new client bookings | pending | 1 |
| 6 | **Pilot & feedback** | Deploy to test group of professionals, gather feedback, iterate | pending | 1, 2, 3 |

---

*Generated: 2026-02-18*
*Status: DRAFT - needs validation with target professionals*
