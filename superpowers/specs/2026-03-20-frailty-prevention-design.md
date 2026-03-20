# Frailty Prevention Feature — NHS Wales App

## Design Specification

**Date:** 2026-03-20
**Status:** Draft
**Decision supported:** Should DHCW invest in building a frailty/falls prevention feature in the NHS Wales app?

---

## 1. Overview

A standalone frailty prevention feature in the NHS Wales app that delivers guided strength and balance exercise programmes (digitising existing face-to-face services) to people already identified as frail. The goal is to slow progression and prevent falls/hospitalisations. Available via self-discovery and clinician signposting. Addresses patchy geographic coverage of existing falls prevention services across Welsh health boards.

**Top design constraint:** Accessibility for older, less digitally confident users.

---

## 2. Target Population

People already identified as frail — estimated 100,000+ in Wales based on eFI prevalence data. The feature supplements (not replaces) existing face-to-face falls prevention services, extending reach to underserved areas.

**Discovery routes:**
- Self-discovery within the NHS Wales app
- Clinician signposting (GPs, community nurses, falls teams) via toolkit materials (leaflet, QR code, talking points)

---

## 3. Feature Design

### 3.1 My Exercise Programme

A structured, progressive 12-week plan based on Otago-style exercises.

- Each day presents a simple screen: "Today's exercises" with 3-5 short video demonstrations
- Large play buttons, minimal navigation
- Users mark exercises as done
- Programme gradually increases difficulty at set intervals (e.g. week 5 introduces ankle weights) — no adaptive logic, a clinically validated progression schedule
- Video library of 20-30 exercises, filmed with real older adults demonstrating each movement (seated and standing variations)
- Each video: 30-90 seconds, front-on camera angle, clear verbal instruction with captions
- Fallback: each exercise also available as an illustrated step-by-step card (static images + text) for low-bandwidth or preference

**Programme content model:** weeks -> days -> exercises, with progression rules. Allows clinical teams to update the programme without app code changes.

### 3.2 Home Safety Check

A guided, room-by-room walkthrough helping users identify and address common falls hazards in their home environment.

- Simple checklist format: "Do you have loose rugs or mats?" -> yes/no -> if yes, practical advice ("Remove loose rugs or secure them with non-slip tape")
- Covers: flooring (rugs, mats, trailing cables), lighting (dark hallways, stairs), bathroom (grab rails, non-slip mats), footwear, and clutter
- Produces a personalised summary of actions the user can take or share with family/carers
- Can be revisited and updated over time
- Authored as structured data: rooms -> hazards -> yes/no -> advice text
- Easily updateable by content editors without developer involvement
- Summary stored locally on device

### 3.3 Local Classes Near Me

A searchable directory of community-based exercise classes (e.g. falls prevention groups, Age Cymru sessions, health board-run programmes).

- Users enter their postcode or allow location access
- Results show: class name, provider, type (e.g. Otago, tai chi, chair-based), location, day/time, contact details, accessibility info
- Searchable by postcode with radius filter
- Flagged as "check with provider before attending" to manage schedule change expectations
- Managed dataset updated periodically (e.g. quarterly) by central DHCW team or via health board submissions — not a live API or web scrape

---

## 4. Accessibility-First Design Principles

- Extra-large text and touch targets throughout
- High contrast, minimal visual clutter
- Video captions and audio descriptions
- No gestures beyond tap and scroll
- Simple linear navigation (no nested menus)
- Option for text-and-image-only mode (illustrated cards) for users who can't stream video
- Text resizable via system accessibility settings
- Clear "need help?" route — phone number for local falls team or NHS 111 Wales
- WCAG 2.1 AA minimum compliance
- Works on iOS and Android, including devices 3-4 years old
- Usable without creating a separate account
- Loads and functions acceptably on 3G/slow connections (video can buffer, cards must work offline)

---

## 5. Data, Privacy & Welsh Language

**Data storage:**
- All user progress data (exercises completed, checklist results) stored locally on device
- No personally identifiable data transmitted to backend systems
- Aggregated, anonymised usage analytics only (programme starts, completion rates, most-viewed exercises) to support ongoing value reporting

**Welsh language:**
- Not required at launch, but content model supports bilingual content from day one so it can be added without restructuring
- Video captions are the main dependency — plan for Welsh-captioned versions in a future phase

---

## 6. Economic Case (Health Economist Perspective)

### 6.1 Counterfactual (Do Nothing)

- Existing face-to-face falls prevention services remain patchy across health boards
- Geographic inequality in access persists
- No national-scale engagement data to inform commissioning decisions
- Patients in underserved areas rely on ad-hoc advice, leaflets, or nothing

### 6.2 Cost Framing

| Item | Estimate range | Notes |
|------|---------------|-------|
| Video content commissioning | £15,000-£30,000 | Filming, captioning, clinical review |
| App development (3 features) | £40,000-£80,000 | Depends on existing app architecture |
| Local classes data setup | £5,000-£10,000 | Initial curation and process design |
| Annual maintenance | £10,000-£20,000 | Content updates, classes directory, bug fixes |
| **Total Year 1** | **£60,000-£120,000** | |

### 6.3 Benefit Framing (Cost-Avoidance)

- Average NHS cost of a fall requiring A&E attendance: ~£2,000-£4,000
- Average cost of a hip fracture (acute + rehab + social care): ~£25,000-£30,000
- Otago programme evidence: 35% reduction in falls rate among participants (Campbell et al., BMJ)
- Even with conservative digital adherence assumptions (lower than face-to-face), preventing **5-10 hip fractures per year** across Wales would cover the full build and running cost
- At 1,000 active users (modest for a national app), even a 10% reduction in falls events generates meaningful cost-avoidance

### 6.4 Non-Financial Value (Cost-Consequence Framing)

- Equalised access across all health boards — directly supports "once for Wales"
- Patient activation and self-management confidence
- Home hazard reduction benefits extend to other household members
- National engagement data enables evidence-based commissioning of face-to-face services where digital is not sufficient
- Bridge into local classes supports existing community providers rather than displacing them

### 6.5 Key Uncertainties

- Digital adherence rates for this demographic are not well-established — face-to-face Otago evidence may not transfer directly
- Uptake depends heavily on clinician signposting behaviour, which is outside DHCW's direct control
- The local classes directory requires sustained data maintenance commitment

### 6.6 Summary Assessment

Low-cost, low-risk investment with asymmetric upside. Even under pessimistic adoption assumptions, the cost-per-user is favourable compared to face-to-face alternatives. Under optimistic assumptions, it materially reduces falls-related acute costs while strengthening DHCW's credibility as a national digital enabler.

---

## 7. Delivery Assessment (App Lead Perspective)

### 7.1 Build Effort

| Feature | Effort | Complexity |
|---------|--------|------------|
| Exercise programme (UI, progress tracking, video player) | 6-8 weeks | Medium |
| Home Safety Check (questionnaire, summary, local storage) | 2-3 weeks | Low |
| Local Classes directory (search, postcode lookup, data model) | 3-4 weeks | Low-Medium |
| Accessibility testing and refinement | 2-3 weeks | Medium |
| Content integration (videos, checklist text, classes data) | 2-3 weeks | Low |
| **Total estimated build** | **12-16 weeks** | Sequential; parallel tracks could compress |

### 7.2 Delivery Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Video content not ready when development completes | High | Delays launch | Commission content early; develop with placeholder videos |
| Poor video performance on older/low-end devices | Medium | Undermines accessibility | Provide illustrated card fallback; test on low-spec devices from sprint 1 |
| Local classes data incomplete or stale at launch | Medium | Erodes user trust | Launch with partial coverage, clearly label; define health board submission process early |
| Low clinician signposting uptake | Medium | Low adoption | Provide signposting toolkit for clinical teams |
| Accessibility gaps missed in testing | Low-Medium | Excludes target users | Involve older adult testers early; WCAG 2.1 AA; test with screen readers |

### 7.3 User Requirements

- Works on iOS and Android, including devices 3-4 years old
- Usable without creating a separate account
- Loads and functions acceptably on 3G/slow connections
- Text resizable via system accessibility settings
- Clear "need help?" route to falls team or NHS 111 Wales

### 7.4 Population Reach

- ~100,000+ people in Wales living with frailty
- NHS Wales app install base determines initial ceiling
- Clinician signposting is the main adoption lever for this demographic
- Digital exclusion is real for this group; the app is positioned as complementary to face-to-face services

---

## 8. Success Criteria and Evaluation

**Adoption targets:**
- 3 months post-launch: 500 programme starts
- 6 months: 1,500 programme starts, 300 programme completions (12-week cycle)
- 12 months: 3,000 programme starts, 800 completions

**Key metrics to instrument:**
- Programme starts, weekly adherence rates, and 12-week completion rates
- Home Safety Check completions and action summaries generated
- Local Classes searches and click-throughs to contact details
- Feature discovery route (self-found vs. clinician signposted, captured via optional "how did you hear about this?" prompt)

**Programme completion target:** 25-30% of starters completing the full 12 weeks would indicate acceptable digital adherence (benchmarked against typical digital health programme completion rates). Below 15% would trigger a review of the user experience and content.

**Cost-avoidance estimation:** Since falls data sits in clinical systems the app does not connect to, cost-avoidance will be estimated indirectly using: app engagement data (active users, programme adherence) combined with published Otago effect sizes to model expected falls prevented. This will be reported as a range, not a point estimate, with assumptions stated transparently.

**Decision points:**
- At 6 months: if adoption is below 50% of target and clinician signposting toolkit has been distributed, commission user research to diagnose barriers before further investment
- At 12 months: if completion rates are below 15%, reconsider the standalone digital-only model and explore hybrid approaches (e.g. app + remote physiotherapist check-ins)

---

## 9. Clinician Signposting Toolkit

Since clinician signposting is the primary adoption lever, a toolkit must be ready at or before app launch. Owned by DHCW communications team in collaboration with the falls prevention clinical lead.

**Contents:**
- A4 patient-facing leaflet: feature overview, QR code to download/open, what to expect
- Brief clinician guide (one page): what the feature does, which patients to recommend it to, how to bring it up in a consultation (suggested talking points)
- Digital assets: email-ready summary for health board dissemination, social media graphics for Age Cymru / community partners

**Timeline:** Content drafted during app build phase (weeks 4-8), reviewed by clinical leads, printed/distributed 2 weeks before launch. Digital assets available at launch.

---

## 10. Content Governance

**Exercise programme content:** Owned by DHCW clinical lead for digital services, with clinical review and sign-off required for any changes to exercises, progression schedules, or safety messaging. Changes follow a documented review-approve-publish process before being pushed to the app content model.

**Home Safety Check content:** Owned by DHCW content team. Updates to hazard advice text require review by a falls prevention clinician. Room/hazard structure changes require clinical lead sign-off.

**Local Classes directory:** Owned by DHCW data team. Health boards submit updates via a standard form (class name, provider, location, schedule, contact, accessibility info). DHCW validates and publishes quarterly. Health boards are contacted during the build phase to confirm participation and establish submission contacts. Classes can be flagged or removed between cycles if reported as discontinued.

**Safety messaging:** All exercise content must include appropriate safety disclaimers (e.g. "stop if you feel pain or dizziness", "consult your GP before starting if you have had a recent fall or surgery"). These are included in video introductions and on the programme landing screen.

---

## 11. Local Classes Data Pipeline

**Submission process:**
1. DHCW data team provides each health board with a standard submission spreadsheet (fields: class name, provider, type, address, postcode, day/time, contact phone, contact email, accessibility notes, start/end dates if seasonal)
2. Health boards nominate a single point of contact responsible for submissions
3. Submissions collected quarterly; DHCW validates, deduplicates, and publishes to the app dataset
4. Between quarterly cycles, health boards can flag closures or changes via email to a dedicated DHCW inbox

**Pre-launch requirements:**
- Agreement from at least 4 of 7 health boards to participate at launch
- Initial dataset seeded from publicly available class listings (Age Cymru, local authority websites) and validated with health board contacts
- App clearly labels coverage: "Showing classes we know about in your area. More are being added regularly."

**Ongoing maintenance:** Estimated 1-2 days per quarter for DHCW data team to process submissions and update the dataset
