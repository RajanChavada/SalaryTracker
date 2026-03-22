# AGENT.md — Arc: Salary & Career Progression Tracker
### System Architecture · Functional & Non-Functional Requirements

---

## 1. Project Overview

**App name:** Arc
**Platform:** iOS (SwiftUI)
**Target audience:** Any working adult who has ever tracked salary in a spreadsheet — universal, not niche
**Core value proposition:** A private, beautiful, on-device record of your career and salary progression — visualised in a way that makes you feel proud of how far you've come
**Revenue model:** Freemium subscription — Trajectory ($3.99/mo · $27.99/yr) and Wealth ($6.99/mo · $49.99/yr)
**Architecture principle:** Local-first — all salary data stays on device by default
**V1 launch scope:** Core logging, timeline, growth chart, milestones, Year in Review, and two paid tiers

---

## 2. Non-Functional Requirements

### 2.1 Platform & Environment

| Requirement | Specification |
|---|---|
| Platform | iOS 16+ (SwiftUI) |
| Minimum deployment target | iOS 16.0 |
| Device support | iPhone only (V1) |
| Orientation | Portrait only |
| Appearance | Dark mode primary, light mode supported |
| Localisation | English (V1). Currency and date formatting localised. |
| Supported currencies | CAD, USD (V1). GBP, EUR (V2). |

### 2.2 Performance

| Metric | Requirement |
|---|---|
| App cold start | < 1 second |
| Chart render (≤50 salary entries) | < 300ms |
| Chart render (>50 salary entries) | < 500ms (pre-computed on background thread) |
| Entry save + immediate chart update | < 200ms visible latency |
| Year in Review card generation | < 2 seconds |
| CloudKit sync on app open | Background only — never blocks UI |
| Milestone evaluation on save | < 100ms (synchronous, on background thread) |
| Animation frame rate | 60fps, no dropped frames |

### 2.3 Security & Privacy

| Requirement | Specification |
|---|---|
| Salary data storage | On-device (Core Data) — never transmitted to a server in raw form |
| CloudKit sync | Encrypted end-to-end via CloudKit private database — Apple cannot read it, developer cannot read it |
| Market benchmarking data | Device sends only anonymised, bucketed range (e.g., "$90k–$110k, tech, Canada") — never a raw number |
| Authentication tokens | Stored in iOS Keychain exclusively |
| App lock | Optional Face ID / Touch ID lock — presented as a feature during onboarding |
| Year in Review sharing | Never includes absolute salary figures — % growth, career stages, and milestones only |
| Account deletion | Full data deletion available in-app (App Store requirement) — Core Data purged, CloudKit records deleted |
| Apple privacy manifest | Required — declare all data types. Salary data stays on-device and is not declared as "collected" |

### 2.4 Accessibility

| Requirement | Specification |
|---|---|
| VoiceOver | All charts have `.accessibilityLabel` with prose description of the data |
| Dynamic Type | All text — including chart labels — scales with system font size |
| Reduce Motion | All chart animations and milestone animations have static fallbacks |
| Colour contrast | WCAG AA minimum across both dark and light themes |
| Tabular figures | All salary numbers use `.monospacedDigit()` modifier — columns align |
| Tap target size | Minimum 44×44pt for all interactive elements |

### 2.5 Reliability

| Requirement | Specification |
|---|---|
| Offline | Full functionality offline — Core Data is the primary store |
| CloudKit sync conflict | Last-write-wins with visible "synced" indicator; no silent data loss |
| Data recovery | If app deleted without CloudKit — data is gone. This risk must be surfaced at the "skip account" prompt |
| Error handling | All save, compute, and sync operations have explicit error states |

---

## 3. Tech Stack

### 3.1 Client

| Layer | Technology |
|---|---|
| UI Framework | SwiftUI |
| Charts | Swift Charts (native — no third-party library) |
| Local storage | Core Data |
| Sync | CloudKit (private database — opt-in) |
| Keychain | KeychainAccess or native Security framework |
| Auth | Sign in with Apple (StoreKit identity) |
| Subscriptions | StoreKit 2 |
| Push notifications | APNs (local notifications primarily; remote for future market alerts) |
| PDF generation | PDFKit (native iOS — Wealth tier export) |

### 3.2 Backend (Minimal)

| Layer | Technology | Purpose |
|---|---|---|
| Auth validation | Apple Sign In server-side token validation | Verify Apple identity tokens |
| Subscription status | RevenueCat or App Store Server API | Validate and serve subscription status |
| Market benchmarks | Static JSON bundled in app binary | Anonymised salary range data by field + region — updated with each app release |
| Push delivery | APNs via lightweight server | Quarterly check-in and future market pulse notifications |

**What the backend never sees:**
- Raw salary amounts
- Individual salary entries
- Any personally identifiable career data

### 3.3 Data Flow Architecture

```
User enters salary entry
        ↓
SalaryNormalisationService (on-device)
    – Converts raw → normalised_annual
    – Derives all computed fields
        ↓
Core Data (on-device primary store)
        ↓
MilestoneEvaluationService (on-device background thread)
    – Checks all milestone thresholds
    – Fires milestone unlock if triggered
        ↓
Swift Charts ViewModel (on-device background thread)
    – Pre-computes chart data arrays
    – Caches derived values (CAGR, forecast, lifetime earnings)
        ↓
CloudKit Private DB (background sync — opt-in)
    – Encrypted Core Data records
    – No salary amounts readable server-side
```

---

## 4. Core Data Schema

### 4.1 `UserProfile`

```swift
// Core Data Entity
id: UUID
displayName: String
currency: String              // "CAD" | "USD"
defaultHoursPerWeek: Int16   // default 40 — used for hourly normalisation
subscriptionTier: String      // "free" | "trajectory" | "wealth"
subscriptionExpiresAt: Date?
isFaceIDEnabled: Bool
notificationQuarterlyEnabled: Bool
notificationMilestoneEnabled: Bool
createdAt: Date
lastOpenedAt: Date
```

### 4.2 `SalaryEntry`

```swift
// Core Data Entity
id: UUID
profile: UserProfile          // relationship
entryType: String             // "job" | "side_income" | "one_time"
jobTitle: String
company: String?
employmentType: String        // "full_time" | "part_time" | "contract" | "self_employed"
rawAmount: Decimal
rawType: String               // "hourly" | "monthly" | "annual"
currency: String              // "CAD" | "USD"
hoursPerWeek: Int16?          // only if rawType == "hourly"; nil uses profile default
normalisedAnnual: Decimal     // computed on save, stored — never re-derived
startDate: Date               // day not required — stored as first of month
endDate: Date?                // nil = current role
isCurrent: Bool
notes: String?                // 500 char max
createdAt: Date
updatedAt: Date
```

### 4.3 `MilestoneRecord`

```swift
// Core Data Entity
id: UUID
profile: UserProfile
milestoneKey: String          // e.g. "first_log", "six_figures"
unlockedAt: Date
wasShown: Bool                // false until animation displayed to user
```

### 4.4 `YearInReview`

```swift
// Core Data Entity
id: UUID
profile: UserProfile
year: Int16
generatedAt: Date
totalIncomeYear: Decimal      // sum of normalisedAnnual × months active for that year
biggestRaise: Decimal?        // largest single salary increase in the year
careerStage: String           // e.g. "Year 3 of Software Engineering"
goalProgressPercent: Decimal? // % toward goal at year end, if goal set
aiReflection: String?         // generated on-device or via lightweight API call (Trajectory+)
wasShared: Bool
```

---

## 5. Salary Normalisation Logic

All normalisation computed by `SalaryNormalisationService`. This logic is the most critical piece of business logic in the app — it must be unit tested exhaustively.

### 5.1 Normalisation Rules

```
Annual    → normalisedAnnual = rawAmount
Monthly   → normalisedAnnual = rawAmount × 12
Hourly    → normalisedAnnual = rawAmount × hoursPerWeek × 52
              (hoursPerWeek: use entry-level if set, else profile default, else 40)
```

### 5.2 Storage Rule

Always store both `rawAmount` + `rawType` + `normalisedAnnual`. Never re-derive `normalisedAnnual` from raw values at query time. Computed once on save, stored permanently.

### 5.3 Overlapping Jobs

A user can have two simultaneous active entries (e.g., full-time job + part-time side role). Both are valid. When computing total income for a period:

- Sum all `normalisedAnnual` values active in that period (pro-rated by overlap months)
- Display total and individual breakdowns separately — never silently merge them

### 5.4 Currency Handling

- All computation done in the user's `defaultCurrency` (set in profile)
- Entries in a different currency converted using a static bundled FX rate table
- FX rate table bundled in app binary — not a live API call
- Converted values marked with a `~` prefix and a footnote: "Converted using rate as of [date]"
- Historical FX rates not modelled at V1 — single static rate applied uniformly

### 5.5 Part-Time Normalisation

Part-time entries are normalised to annual using stated hours. They are:

- Visually flagged in the chart and timeline with a part-time indicator
- Included in total income calculations
- Excluded from "career salary growth" percentage if user wishes (user toggle — V2)
- Never silently treated as full-time equivalent

### 5.6 Edge Cases

| Scenario | Handling |
|---|---|
| $0 salary (unpaid internship) | Valid — accepted with no error or warning |
| End date before start date | Validation error — block submission with inline message |
| Two simultaneous "current" entries | Valid — both shown as active in timeline |
| Salary of $10M+ | Valid — no cap. No error. Some users are executives. |
| Gap in employment history | Valid — timeline shows gap visually, no imputation |
| Entry dated 1970 or earlier | Valid — accept any date from 1970-01-01 |
| Pay cut (lower salary than previous) | Valid — shown with a visual marker in chart, no warning |
| Missing company name | Valid — optional field, shown as "—" in UI |
| CloudKit sync conflict | Last-write-wins; show "last synced [time]" indicator |
| App reinstalled, no CloudKit | Data gone — surface this risk at "skip account" prompt with clear language |

---

## 6. Functional Requirements

### 6.1 Onboarding & First Run

| ID | Requirement | Notes |
|---|---|---|
| ONB-01 | App opens directly to "Log your first role" screen — no splash, no feature tour | |
| ONB-02 | Required fields for first entry: job title, start date, salary amount, salary type | Company, notes optional |
| ONB-03 | First entry triggers first milestone animation ("Your Arc begins") immediately | |
| ONB-04 | After first entry, a soft tooltip sequence (max 4 tips) explains timeline, goal setting, and chart — all dismissable | |
| ONB-05 | Account creation prompt appears after first entry is saved — not before | |
| ONB-06 | User can skip account creation — full offline functionality available immediately | Data loss risk disclosed clearly |
| ONB-07 | Face ID lock option offered during or immediately after onboarding | Framed as a feature, not a permission |
| ONB-08 | First chart render and first milestone animation visible within 60 seconds of first open | |

### 6.2 Salary & Job Entry Logging

| ID | Requirement | Tier | Notes |
|---|---|---|---|
| ENTRY-01 | User can log a new job/role with: job title, company (optional), employment type, start date, end date or "current" toggle, salary amount, salary type (hourly/monthly/annual), currency, hours/week (if hourly), notes (optional) | All | |
| ENTRY-02 | User can log a salary change within an existing role (same title, new amount, new date) | All | Displayed as a node annotation on the timeline |
| ENTRY-03 | User can log side income: label, amount, date, income type (one-time / recurring), frequency if recurring | All | |
| ENTRY-04 | normalisedAnnual computed and stored on save — never recomputed at query time | All | SalaryNormalisationService handles this |
| ENTRY-05 | All entries editable after creation (all fields) | All | |
| ENTRY-06 | All entries deletable | All | Core Data delete — CloudKit sync removes record |
| ENTRY-07 | Unlimited entries for all tiers — no cap | All | |
| ENTRY-08 | Overlapping active entries supported (two simultaneous jobs) | All | Both shown in timeline and aggregated in totals |

### 6.3 Career Timeline View

| ID | Requirement | Notes |
|---|---|---|
| TIMELINE-01 | Horizontal scrollable timeline — left (earliest) to right (present) | Each job is a node |
| TIMELINE-02 | Each node shows: role title, company (if entered), duration (e.g., "2 yrs 4 mo"), salary tier indicator (relative height — not absolute value) | |
| TIMELINE-03 | Tapping a node expands the full entry detail (all fields, edit/delete options) | |
| TIMELINE-04 | Gap periods (unemployment) shown as visual gaps in the path between nodes — not errors | |
| TIMELINE-05 | Salary change within a role shown as an annotation on the node (e.g., "+12% raise, Mar 2023") | |
| TIMELINE-06 | Side income entries shown as smaller secondary nodes below the main path | |
| TIMELINE-07 | "Current" role shown with a subtle animated pulse or glow to indicate active | |
| TIMELINE-08 | Timeline styled as a "journey" or "road" — not a flat list or generic chart | Core aesthetic — see CLAUDE.md §4.2 |

### 6.4 Growth Chart

| ID | Requirement | Tier | Notes |
|---|---|---|---|
| CHART-01 | Basic growth chart: normalised annual salary over time (line chart) | All | Swift Charts, step interpolation |
| CHART-02 | Chart Y axis starts at $0 unless labelled otherwise | All | |
| CHART-03 | Chart highlights milestone moments with gold markers on the line | All | |
| CHART-04 | Chart shows goal target line if goal is set | All | Dashed horizontal line at goal amount |
| CHART-05 | Salary type (full-time vs part-time) visually differentiated — part-time entries dashed or annotated | All | |
| CHART-06 | Chart line draws in smoothly on first load (left to right) | All | Respects Reduce Motion |
| CHART-07 | CAGR displayed below chart with "What is this?" disclosure tooltip | Trajectory | |
| CHART-08 | AI salary forecast line extends beyond current date (dotted, confidence range shaded) | Trajectory | |
| CHART-09 | Chart breakdown by income type (main salary vs side income) | Trajectory | |
| CHART-10 | Multi-career profile overlay (compare two career paths on same chart) | Wealth | |

### 6.5 Goal Tracking

| ID | Requirement | Tier | Notes |
|---|---|---|---|
| GOAL-01 | User sets a target salary (annual equivalent) and target date | Free | |
| GOAL-02 | Goal progress shown as: current gap (amount), % achieved, projected arrival date at current growth rate | Free | |
| GOAL-03 | Goal displayed as a subtle horizontal target line on the main chart | Free | |
| GOAL-04 | Goal can be updated or deleted at any time | Free | |
| GOAL-05 | One active goal per profile at V1 | Free | |
| GOAL-06 | Goal proximity notification: "You're 14% away from your $100k goal" (if notifications enabled) | Free | Local notification |

### 6.6 Milestones & Gamification

| ID | Requirement | Tier | Notes |
|---|---|---|---|
| MILE-01 | Milestone evaluated after every entry save — on background thread | All | MilestoneEvaluationService |
| MILE-02 | Milestone unlocks trigger an elegant in-app animation — card reveal with ring completion | All | |
| MILE-03 | Milestones stored in Core Data — never re-awarded after unlock | All | UNIQUE constraint on (profile, milestoneKey) |
| MILE-04 | Milestones gallery view: all unlocked milestones shown as achievement cards | All | |
| MILE-05 | Locked (future) milestones shown as dimmed hints — not hidden | All | Encourages continued use |
| MILE-06 | Extended milestone library (additional thresholds) | Trajectory | |

**V1 Milestone Definitions:**

| Key | Trigger |
|---|---|
| `arc_begins` | First entry saved |
| `first_raise` | Second entry with higher normalisedAnnual than first |
| `first_role_change` | Second distinct job title entered |
| `double_start` | normalisedAnnual ≥ 2× first entry's normalisedAnnual |
| `threshold_50k` | normalisedAnnual ≥ $50,000 |
| `threshold_75k` | normalisedAnnual ≥ $75,000 |
| `threshold_100k` | normalisedAnnual ≥ $100,000 |
| `five_years_tracked` | Span from earliest to latest entry ≥ 5 calendar years |
| `one_year_streak` | App used (any entry) in 2+ distinct calendar years |
| `side_hustle` | First side income entry logged |

### 6.7 Annual Year in Review

| ID | Requirement | Tier | Notes |
|---|---|---|---|
| REVIEW-01 | Year in Review generated automatically in December or on app anniversary | Free | Whichever comes first in any given year |
| REVIEW-02 | Content: biggest salary jump that year, total income earned, career stage label, goal progress, one AI reflection line | Free | |
| REVIEW-03 | Displayed as a scrollable card sequence — beautiful, branded, premium | Free | |
| REVIEW-04 | Shareable as an image — shows % growth and milestones only, never absolute salary numbers | Free | |
| REVIEW-05 | Share image formatted for Instagram Story (9:16) and standard square (1:1) | Free | |
| REVIEW-06 | AI reflection line generated on-device (or lightweight API) — constructive and warm | Trajectory (richer version) | Free version uses template strings |
| REVIEW-07 | Previous years' reviews accessible in history | All | |

### 6.8 Premium Features — Trajectory Tier

| ID | Requirement | Notes |
|---|---|---|
| TRAJ-01 | AI salary forecast: at current CAGR, projected salary at 1yr / 3yr / 5yr — shown as dotted line extension on chart | |
| TRAJ-02 | CAGR calculation displayed: formula disclosed via "How is this calculated?" tooltip | |
| TRAJ-03 | Market pulse: user's growth compared to anonymised aggregate for their field + region + career stage | Constructively framed — "You're in the top 40% for your field" — never raw negative |
| TRAJ-04 | Market pulse data sourced from bundled static dataset — updated with each app release | |
| TRAJ-05 | Additional chart types: annotated multi-axis (salary + income type breakdown) | |
| TRAJ-06 | Extended milestone library with additional threshold badges | |

### 6.9 Premium Features — Wealth Tier

| ID | Requirement | Notes |
|---|---|---|
| WEALTH-01 | Negotiation simulator: enter a proposed raise %, see compounding salary impact at 1yr / 3yr / 5yr | |
| WEALTH-02 | PDF export: full salary history formatted as a professional document — role, dates, amount, currency, employment type | Suitable for mortgage, visa, salary negotiation |
| WEALTH-03 | Lifetime earnings calculation: total income across all entries, pro-rated for duration | Computed on-device — never sent to server |
| WEALTH-04 | Multiple career profiles: user can create and switch between profiles (e.g., track a partner's career separately) | Data fully isolated between profiles |

### 6.10 Settings & Account

| ID | Requirement | Notes |
|---|---|---|
| SET-01 | Theme: dark (default) / light | |
| SET-02 | Default currency: CAD / USD | |
| SET-03 | Default hours per week (for hourly normalisation): user-set, default 40 | |
| SET-04 | Face ID / Touch ID app lock: on/off | |
| SET-05 | Notification preferences: quarterly check-in, milestone alerts — independently togglable | |
| SET-06 | CloudKit backup: enable/disable, last synced timestamp shown | |
| SET-07 | Subscription management: current tier, upgrade/downgrade, restore purchases | |
| SET-08 | Account deletion: deletes all Core Data, CloudKit records, and Apple account link | |
| SET-09 | Data export (raw CSV of all entries) | V2 |

### 6.11 Retention & Engagement

| ID | Requirement | Notes |
|---|---|---|
| RET-01 | Quarterly check-in: in-app prompt (not push unless opted in) 90 days after last entry — "Anything new to log?" | |
| RET-02 | Goal proximity alert: local notification when user is within 20% of goal salary | Opt-in |
| RET-03 | Annual Year in Review notification in December | Opt-in |
| RET-04 | Milestone unlock notification | Opt-in |
| RET-05 | Quarterly check-in frequency configurable: monthly / quarterly / annually | User preference |

---

## 7. Subscription & Paywall

### 7.1 Tier Summary

| Feature | Free | Trajectory ($3.99/mo) | Wealth ($6.99/mo) |
|---|---|---|---|
| Unlimited salary entries | ✅ | ✅ | ✅ |
| Career timeline | ✅ | ✅ | ✅ |
| Basic growth chart | ✅ | ✅ | ✅ |
| Goal tracking | ✅ | ✅ | ✅ |
| Year in Review | ✅ | ✅ (richer) | ✅ (richer) |
| Core milestones | ✅ | ✅ | ✅ |
| 3 themes | ✅ | ✅ | ✅ |
| AI salary forecast | — | ✅ | ✅ |
| CAGR analysis | — | ✅ | ✅ |
| Market pulse | — | ✅ | ✅ |
| Advanced charts | — | ✅ | ✅ |
| Premium themes | — | ✅ | ✅ |
| Negotiation simulator | — | — | ✅ |
| PDF export | — | — | ✅ |
| Lifetime earnings | — | — | ✅ |
| Multiple profiles | — | — | ✅ |

### 7.2 Implementation

- StoreKit 2 for all IAP and subscription management
- Subscription status validated locally (StoreKit) and optionally via RevenueCat for server-side receipt validation
- Tier cached in `UserProfile.subscriptionTier` — refreshed on app open and after any purchase/restore
- Paywall presented as a bottom sheet — never a blocking full-screen gate
- Upgrade prompt copy: understated, never urgent. "Unlock full analysis" not "Don't miss out!"

---

## 8. Computed Value Reference

All values computed by `SalaryAnalysisService` on background thread. Results cached in `AnalysisCache` (in-memory, invalidated on any new entry save).

| Value | Formula | Tier |
|---|---|---|
| Total % growth | (latestNormalisedAnnual - firstNormalisedAnnual) / firstNormalisedAnnual × 100 | Free |
| CAGR | (latestNormalisedAnnual / firstNormalisedAnnual)^(1/years) - 1 | Trajectory |
| Forecast (1yr) | latestNormalisedAnnual × (1 + CAGR) | Trajectory |
| Forecast (3yr) | latestNormalisedAnnual × (1 + CAGR)^3 | Trajectory |
| Forecast (5yr) | latestNormalisedAnnual × (1 + CAGR)^5 | Trajectory |
| Goal delta | goalAmount - latestNormalisedAnnual | Free |
| Goal arrival estimate | years until goalAmount at current CAGR | Free |
| Lifetime earnings | Sum of (normalisedAnnual × overlap_months/12) for all entries | Wealth |
| Negotiation impact (Nyr) | latestNormalisedAnnual × (1 + proposedRaise%) × (1 + CAGR)^N | Wealth |
| Total income (calendar year) | Sum of pro-rated normalisedAnnual for all entries active in that year | Free (Year in Review) |

All formulas disclosed in-app via "How is this calculated?" disclosure — no black-box outputs.

---

## 9. PDF Export Specification (Wealth Tier)

Generated using PDFKit (native iOS). Structure:

```
Page 1: Cover
  – Arc logo
  – "Career Salary History"
  – User's name and currency
  – Date range covered (first entry → today)
  – Generated date

Page 2+: Salary History Table
  – Columns: Role, Company, Employment Type, Start Date, End Date, Annual Equivalent, Currency
  – Sorted chronologically ascending
  – Part-time entries marked with "(part-time, N hrs/wk)"
  – Side income entries in a separate section

Final Page: Summary
  – Total career span
  – Total % salary growth
  – CAGR
  – Lifetime earnings total
  – Prepared by Arc — [generated date]
```

**Privacy:** PDF generated entirely on-device. No data transmitted. User shares the file via iOS Share Sheet — fully in user's control.

---

*Last updated: project inception. Update when requirements change — this document must reflect the current agreed state of Arc at all times.*
