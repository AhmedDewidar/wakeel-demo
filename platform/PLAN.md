# Build plan and database schema

Status: for founder review. No application code is written before this plan is accepted (brief, section 14, rule 1).

## 1. Repository layout

The repository root holds the password-protected feasibility site (GitHub Pages). The prototype goes in `platform/` so that the site continues to work.

```text
platform/
  src/domain/         pure functions + unit tests (pricing, commission, renewal check,
                      pro-rata, monthly payments, census validation, needs check)
  src/integrations/   one interface per external service + mock (9 interfaces, section 10)
  src/db/             Drizzle schema, migrations, query helpers with role and tenant filters
  src/app/            (public), /employer, /advisor, /insurer, /admin; server actions
  src/messages/       ar.json (default), en.json
  scripts/            seed, seed reset, claims document generator
  e2e/                Playwright demo script (section 13)
  README.md, ASSUMPTIONS.md, BACKLOG.md, .env.example
.github/workflows/ci.yml   lint, type check, Vitest, Playwright (PostgreSQL service)
```

## 2. Stack

The stack is the default of section 10. Three details are not in the brief:

| Item | Decision | Reason |
| --- | --- | --- |
| Sign-in | Auth.js credentials provider (email only, demo users only) with JWT sessions | The brief permits no real email service. No session tables are necessary |
| Database driver | `pg` with Drizzle. Local PostgreSQL 16 in development and CI; Neon in deployment | One driver for the two environments |
| Fonts | Self-hosted font files (Cairo, Noto Sans Arabic, Inter) | The same fonts for screens and for PDF output; no network call at build time |

## 3. Phases

The phases and the acceptance tests are those of section 12. Each phase ends with: all tests green, README update, a short note (works, mocked, open), then a stop for acceptance.

| Phase | Main work | Order inside the phase |
| --- | --- | --- |
| 0 | Foundation | Scaffold, schema + migration, seed, sign-in + roles + switcher, AR/EN shell + tokens + banner, insurer isolation test, CI |
| 1 | Registration, needs check, data pack | Domain rules first, then census, then claims reader (fixture, then Claude API), then document generator |
| 2 | RFQ, quotes, comparison | Pricing engine + tests, RFQ workflow, insurer portal, comparison, PDF export |
| 3 | Purchase, administration | Payment + financing + invoice + certificate mocks, members, payroll import, fleet |
| 4 | Renewal, claims follow-up, advisor console | Renewal check first (demo step 9), then console, then claims follow-up |
| 5 | Finish, deploy | Full Playwright script, empty and error states, 360 px check, deployment, handover |

## 4. Points where the brief is not clear (proposed resolution)

| # | Point | Proposed resolution |
| --- | --- | --- |
| 1 | Motor commission: default rate 8%, but the cap for compulsory-only cover is 2% | Two motor rates in the configuration: third party only 2% (cap 2%); comprehensive 8% (cap 15%) |
| 2 | Demo Clinics Group has "no health premium yet", but demo step 4 uploads its claims report | The report comes from its current insurer outside the platform. The needs check shows health as "not on the platform" |
| 3 | Demo step 2 starts the RFQ before the data pack exists; step 5 sends it | Step 2 makes a draft RFQ with the three lines. The advisor completes and sends it in step 5. `data_pack_version` is set at send time |
| 4 | Medical malpractice is shown as a line in the needs check, but it is a liability type | The needs rules give a line plus an optional type (liability: public, professional, malpractice) |
| 5 | "Deterministic seed" and "renewal in 55 days" | Random values use a fixed seed. Dates are relative to the time of the seed or reset |
| 6 | The invited HR user must be usable in demo step 3 | In demo mode an invitation creates the user immediately and writes the email to the outbox |
| 7 | The data model lists 20 entities, but the brief also needs a configuration table and an outbox | Two more tables: `config_entry`, `outbox_message` |
| 8 | Demo step 9 range (SAR 1.58 to 1.84 million) | Correct only if members now = average members (420). The seed keeps this equal: 1,242,000 x 1.10 / 0.80 = 1,707,750; range 1,579,669 to 1,835,831 |

Open questions of section 14: all defaults apply (name "منصة تأمين المنشآت (نموذج أولي)" through `APP_NAME`, Railway + Neon, Claude API with fixture fallback, lighter form for the other lines, section 7 values, demo needs rules, no real data).

## 5. Database schema

Conventions: `uuid` primary keys; money as `bigint` halalas; times as `timestamptz` in UTC (shown in Asia/Riyadh); rates as `numeric`; enumerations as PostgreSQL enums. FK = foreign key.

### Organisations and users

| Table | Columns |
| --- | --- |
| `organisation` | `id`, `type` (employer, insurer, broker, financier), `name_ar`, `name_en`, `created_at` |
| `employer_profile` | `organisation_id` PK FK, `cr_number` unique, `activity_code`, `activity_group`, `city`, `established_on`, `employee_count`, `vehicle_count`, `advisor_user_id` FK user |
| `insurer_profile` | `organisation_id` PK FK, `lines[]`, `automated_pricing`, `price_factor`, `network_levels[]`, `benefit_tables` jsonb (one table per class) |
| `user` | `id`, `organisation_id` FK, `role` (employer_owner, employer_hr, employer_finance, advisor, underwriter, admin), `name`, `email` unique, `locale`, `created_at` |

### Data pack

| Table | Columns |
| --- | --- |
| `member` | `id`, `employer_id` FK, `national_id_test`, `name`, `birth_date`, `gender`, `relation` (employee, spouse, child), `sponsor_member_id` FK member, `class` (VIP, A, B, C), `nationality`, `status` (active, pending, removed); unique (`employer_id`, `national_id_test`) |
| `census_upload` | `id`, `employer_id` FK, `file_ref`, `column_map` jsonb, `rows` jsonb (staged rows with errors and warnings), `row_count`, `error_count`, `warning_count`, `status` (mapped, has_errors, valid, confirmed), `uploaded_by`, `created_at` |
| `claims_document` | `id`, `employer_id` FK, `file_ref`, `file_name`, `type` (pdf_text, pdf_scan, image, excel, csv), `status` (uploaded, extracting, needs_review, confirmed, failed), `extraction_id`, `uploaded_by`, `created_at` |
| `claims_extraction` | `id`, `document_id` FK, `provider` (claude, fixture, parser), `json_result` jsonb (schema of section 8), `confidence`, `validation` jsonb, `duration_ms`, `verified_by` FK user, `verified_at` |
| `data_pack` | `id`, `employer_id` FK, `version`, `profile_snapshot` jsonb, `census_snapshot` jsonb, `claims_summary` jsonb, `created_by`, `created_at`; unique (`employer_id`, `version`) |
| `vehicle` | `id`, `employer_id` FK, `plate_test`, `type` (private_car, pickup, van, truck, bus), `make`, `year`, `value`, `use`, `status` (active, pending, removed) |

### RFQ and quotes

| Table | Columns |
| --- | --- |
| `rfq` | `id`, `employer_id` FK, `data_pack_version` (null in draft), `start_date`, `status` (draft, sent, decided, cancelled), `is_renewal`, `created_by`, `sent_at`, `created_at` |
| `rfq_line` | `id`, `rfq_id` FK, `line` (health, motor, property, liability, cargo), `options_json` jsonb |
| `rfq_insurer` | `id`, `rfq_line_id` FK, `insurer_id` FK, `status` (pending, quoted, declined, expired), `due_at`, `decline_reason`; unique (`rfq_line_id`, `insurer_id`) |
| `quote` | `id`, `rfq_insurer_id` FK unique, `premium_total`, `vat`, `premium_by_class` jsonb, `benefits_json` jsonb, `network_level`, `valid_until`, `conditions`, `commission_rate`, `commission_amount`, `source` (instant, manual), `updated_at` |
| `recommendation` | `id`, `rfq_line_id` FK unique, `quote_id` FK, `advisor_id` FK, `reason`, `created_at` |

### Policies and service

| Table | Columns |
| --- | --- |
| `policy` | `id`, `employer_id` FK, `line`, `insurer_id` FK, `quote_id` FK (null for seed policies), `policy_number`, `start_date`, `end_date`, `premium`, `vat`, `payment_plan` jsonb (method, financier, months, monthly amount), `status` (active, expired, cancelled) |
| `policy_document` | `id`, `policy_id` FK, `type` (invoice, certificate, schedule), `file_ref`, `created_at` |
| `change_request` | `id`, `policy_id` FK, `type` (add, delete), `member_id` FK or `vehicle_id` FK (one of the two), `effective_date`, `prorata_estimate`, `status` (draft, sent, accepted, rejected), `source` (form, file, connector), `created_by`, `created_at` |
| `claim` | `id`, `policy_id` FK, `member_id` FK, `kind` (claim, pre_approval), `date`, `category`, `amount`, `status`, `follow_up_status` (none, requested, in_progress, closed), `notes` jsonb (author, time, text) |

### System

| Table | Columns |
| --- | --- |
| `audit_event` | `id`, `user_id`, `organisation_id`, `action`, `entity`, `entity_id`, `at`, `meta` jsonb |
| `config_entry` | `key` PK, `value` jsonb, `updated_by`, `updated_at` (commission rates and caps, pricing factors, base rates, renewal assumptions, needs rules) |
| `outbox_message` | `id`, `channel` (email, whatsapp), `recipient`, `subject`, `body`, `organisation_id`, `created_at` |

### Access rules in the data layer

- Employer users: each query filters by `employer_id` = the user's organisation.
- Advisor: each query filters by `employer_profile.advisor_user_id` = the user.
- Underwriter: each query starts from `rfq_insurer.insurer_id` = the user's organisation. Quotes, RFQs and data packs are reached only through these rows. An automated test proves the isolation (section 6).
- Each server action checks the role and writes an `audit_event`.
