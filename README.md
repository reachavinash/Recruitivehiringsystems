# Recruitive Hiring Systems — Recruitment Agency Website

Static multi-page website for **Recruitive Hiring Systems**, a startup recruitment agency based at
**Sector 58, Noida, Uttar Pradesh, India**.

Brand: deep navy + royal blue + silver. Tagline: *Structured Hiring. Scalable Growth.*

---

## 1. Goals

Build a credible, fully working agency site that serves three audiences from one codebase:

1. **Candidates** — browse live job vacancies and submit a CV.
2. **Employers / clients** — understand the service, then submit a hiring requirement.
3. **Recruiters (internal)** — post vacancies and work the incoming pipeline.

---

## 2. Completed features

### Public site
| Page | Path | Purpose |
|---|---|---|
| Home | `index.html` | Hero, live vacancy counter, quick search, latest 6 vacancies, services, process, dual CTA |
| Job vacancies | `jobs.html` | Full vacancy list with keyword / category / job-type filters and sorting |
| Job detail | `job.html?id=<record-id>` | Full role description, requirements, sidebar summary, apply CTA |
| Submit CV | `apply.html` (`?job=<record-id>` optional) | Candidate application form with CV upload |
| For employers | `employers.html` (`#enquiry`) | Services, engagement models, client hiring enquiry form |
| About | `about.html` | Positioning, principles, process |
| Contact | `contact.html` | WhatsApp / email / office cards, map embed, general contact form |
| Recruiter console | `admin.html` | Post vacancies, manage them, review applications and enquiries |

### Job vacancy posting
- Vacancies are stored in the `jobs` table and rendered live — no rebuild needed.
- `admin.html` → **Post a vacancy** publishes immediately (status `open`, `posted_at` stamped).
- **Manage vacancies** supports view / close / reopen / delete.
- Vacancy dropdown on the application form is populated from the same table.

### Candidate CV submission → `recruitivehiringsystems@gmail.com`
Three independent delivery paths, so a single failure never loses an application:
1. **CV upload** — the file is POSTed to the hosted runtime files route `/files/cvs`; only the
   returned `reference` and metadata are stored in the row (never the binary).
2. **Row persisted** — the application is written to the `applications` table via `tables/applications`.
3. **Email relay** — a formatted copy is POSTed to the FormSubmit AJAX endpoint, which forwards it to
   `recruitivehiringsystems@gmail.com` with the candidate set as reply-to.

The UI reports exactly what succeeded and falls back to "email us directly" with the address shown
when a path fails.

### Client hiring enquiries
`employers.html#enquiry` captures company, contact, work email, phone, industry, roles, positions,
employment type, budget and timeline → `client_enquiries` table + email relay. Statuses:
`new → contacted → proposal_sent → won → closed`.

### WhatsApp contact — `+91 70807 86893`
- Floating **Chat on WhatsApp** button on every page (`js/site.js` → `renderWhatsAppFloat`).
- Context-aware prefilled messages via any `data-wa="…"` attribute, wired by `hydrateWhatsAppLinks()`.
- Job detail, application and enquiry pages each generate role-specific messages.

### Address
`Sector 58, Noida, Uttar Pradesh, India` — footer on every page, contact page, map embed, and the
`LocalBusiness` JSON-LD block.

---

## 3. Project structure

```
index.html          Home
jobs.html           Vacancy listing
job.html            Vacancy detail (?id=)
apply.html          Candidate CV submission (?job=)
employers.html      Employer services + hiring enquiry (#enquiry)
about.html          About
contact.html        Contact + general message
admin.html          Recruiter console (restrict this route — see §6)
css/style.css       Full design system (single stylesheet)
js/config.js        SITE config, WhatsApp links, Table API wrapper, upload + email helpers
js/site.js          Header / footer / floating WhatsApp injection, icon + data-wa hydration
js/jobs-common.js   Shared job card renderer, sorting, select population
js/home.js          Homepage logic
js/jobs.js          Listing filters + sorting
js/job-detail.js    Single vacancy rendering
js/apply.js         Candidate application form
js/employers.js     Client hiring enquiry form
js/contact.js       General contact form
js/admin.js         Recruiter console
images/logo.jpg     Brand logo
```

Script load order on every page: `config.js` → `site.js` → (`jobs-common.js` where job helpers are
used) → page script.

---

## 4. Data models (`.tables/schema.json`)

**`jobs`** — `title`, `company`, `location`, `job_type`, `category`, `experience`, `salary`,
`openings`, `description`, `requirements`, `status` (`open`|`closed`), `posted_at`

**`applications`** — `job_id`, `job_title`, `full_name`, `email`, `phone`, `city`,
`experience_years`, `current_ctc`, `expected_ctc`, `notice_period`, `cover_letter`,
`cv_reference`, `cv_filename`, `status` (`new`|`screened`|`shortlisted`|`interview`|`offered`|`rejected`), `applied_at`

**`client_enquiries`** — `enquiry_type` (`hiring`|`general`), `company_name`, `contact_name`,
`work_email`, `phone`, `industry`, `hiring_for`, `positions`, `employment_type`, `budget`,
`timeline`, `message`, `status` (`new`|`contacted`|`proposal_sent`|`won`|`closed`), `submitted_at`

Browser access uses **relative** URLs and keeps the `tables/` prefix (`tables/jobs`), while file
uploads use the absolute same-origin `/files/cvs`.

---

## 5. Preview vs. live data — important

These are **two separate stores and are never synced automatically**:

- **Preview store** — rows added with `TableDataAdd` (8 sample vacancies are seeded here).
- **Live D1 database** — created **empty** by Hosted Deploy; only reachable by real visitors.

So after deploying, the live site shows **no vacancies** until the preview rows are imported. To copy
them across, run **`HostedDbSyncFromPreview`** (then `hosted_action_wait`). It is idempotent, keeps
live-only rows, and never deletes anything. Alternatively post vacancies directly in the live site's
`admin.html` console.

---

## 6. Access control — action required

`admin.html` reads candidate PII (CVs, emails, phone numbers) and client contact details, so the
following routes should be **restricted at the hosting layer**, not with page JavaScript:

| Route | Should be |
|---|---|
| `/admin.html` | authenticated |
| `tables/applications` (GET) | authenticated |
| `tables/client_enquiries` (GET) | authenticated |
| `tables/jobs` (GET) | public (needed by the public vacancy list) |
| `tables/jobs` (POST/PATCH/DELETE) | authenticated |
| `tables/applications` (POST) | public (candidates apply anonymously) |
| `tables/client_enquiries` (POST) | public (clients enquire anonymously) |
| `/files/cvs` (POST) | public (candidate CV upload) |
| `/files/cvs/{id}` (GET) | authenticated (CV downloads) |

Saved rules only take effect after an approved Hosted Deploy. Note that the platform's organization
floor always applies and takes precedence. Until these rules are published, treat `admin.html` as
unlisted-but-public and do not share the URL.

---

## 7. External dependencies

| Service | Used for | Notes |
|---|---|---|
| FormSubmit AJAX (`formsubmit.co/ajax/…`) | Email relay to the agency inbox | **First submission triggers a one-time activation email** to `recruitivehiringsystems@gmail.com` — click the link in it once, or the relay silently fails. The table row is still saved regardless. |
| Google Fonts | Inter + Sora | Falls back to system fonts offline |
| Google Maps embed | Contact page map | No API key needed |
| jsDelivr / platform | none required | No frontend framework used |

No credentials, API keys or secrets are stored in the site. Nothing pretends to enforce
authorization in client code.

---

## 8. Verification performed

- `PlaywrightConsoleCapture` on `index.html`, `jobs.html`, `apply.html`, `admin.html` — no page errors.
  (Two real bugs were found and fixed this way: `admin.html` and `apply.html` were missing the
  `js/jobs-common.js` script tag, so `fillSelect` / `sortJobsByDate` / `isOpen` were undefined.)
- `PlaywrightScreenshot` on `index.html` (desktop, full page) and `jobs.html` — header, nav, hero,
  search card, vacancy cards, filters and floating WhatsApp button all render correctly, with no raw
  code or `${…}` template placeholders leaking as visible text.

---

## 9. Remaining work / recommended next steps

1. **Deploy and activate the email relay** — publish, then submit one test application and click the
   FormSubmit activation link sent to the agency inbox.
2. **Publish the access rules in §6** so `admin.html` and the PII tables are not publicly readable.
3. **Import the sample vacancies** to the live database (`HostedDbSyncFromPreview`), or post real ones
   from the live console.
4. **Confirm visitor CV uploads are enabled** on the deployed address (requires Hosted membership +
   platform activation). Until then the form still saves the application and tells the candidate to
   email the CV — it never silently discards it.
5. Replace the 8 seeded sample vacancies with real mandates.
6. Optional: candidate-facing application status lookup, recruiter login instead of route rules,
   bulk CV export, and an analytics-free "jobs by category" chart.
