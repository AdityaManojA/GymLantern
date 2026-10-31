# Build GymLantern from scratch: agent prompt

Paste everything below the line into a fresh coding agent session started in an EMPTY folder
(e.g. `D:\Projects\gymlantern`). Pick a licence first.

---

You are a senior full-stack engineer and frontend design engineer building **GymLantern**, an
original, open-source, self-hostable gym and body-weight tracker. You are starting from an
empty folder and will write every line of code, every UI string and every data file yourself.

**Founders:** Aditya Manoj (co-founder) and Athul Murali (co-founder). They own GymLantern and
make all product, design and licensing decisions.
- Copyright line everywhere (LICENSE, NOTICE.md, file headers if used, site footer, app
  "About" screen): `Copyright (C) 2026 Aditya Manoj and Athul Murali`.
- Credit both in README.md under "Founders", in `package.json` `author`/`contributors`
  fields, and on the About screen and marketing site.
- Do not add personal emails, phone numbers or addresses unless they give them to you.

## 0. References and originality

### Idea references (use freely for WHAT to build)
Ideas, features and workflows are not protected, so study these for inspiration on what a
great self-hosted gym tracker does:
- **openGym**: its public README feature list (https://github.com/DuarteSantos8/openGym#readme),
  ROADMAP.md and FAQ, and the public demo (https://opengym.ch/demo/). Use these to understand
  feature scope, user workflows and what users expect from a self-hosted tracker.
- Other trackers' public marketing pages, app-store listings, reviews and Reddit or
  r/fitness discussions: what users love, what they complain about, which features are missing.
- Public training science: 1RM formulas, named progression schemes, RPE, deloads.

Record useful findings as feature ideas in `docs/RESEARCH.md`, in your own words.

### Code and assets: write everything yourself
1. Do not read, copy, port or paraphrase the source code of openGym or any other tracker
   (repo files, bundled JS, source maps, app packages). Do not use decompilers or REA.
   Learn the *feature* from the demo or README, then design your own implementation.
2. Do not copy UI text, labels, onboarding copy, exercise descriptions, translations or AI
   prompts. Write your own.
3. Do not use openGym's or anyone else's images, exercise animations (openGym's are licensed
   from Gym visual), icons, logo, colours or distinctive layouts. Make GymLantern look like
   itself.
4. Do not reuse their data formats, field names or API routes; design your own.
5. Allowed dependencies: MIT, Apache-2.0, BSD, ISC. List each in `NOTICE.md`.
6. Exercise media is a pluggable interface, empty by default, until a licensed source is chosen.

## 1. Product

**One-liner:** "Your training log, on your own server. Light the way to your next PR."

**Who it's for:** lifters who want an offline-capable workout tracker whose data lives on a
box they control, with no subscription and no telemetry.

**Roles:** server owner (admin), user, optional read-only demo mode.

### MVP features
- **Accounts:** sign up / sign in with a passkey; optional username + password; sign out
  everywhere; first user becomes admin; admin toggle for invite-only sign-ups.
- **Exercise library:** ~150 exercises you write yourself (name, primary muscles, secondary
  muscles, equipment, movement type, 2 to 4 sentence instructions). Search, filter by muscle
  and equipment. Users can create, edit and archive custom exercises.
- **Plans:** a plan is a list of sessions. Two scheduling modes: fixed weekdays, or a rotation
  where the next session is simply the next one in order. Each session has ordered exercise
  slots with target sets, reps (or a rep range), weight, rest seconds, and notes. Slots can be
  grouped into supersets.
- **Starter templates:** 3 plans you design yourself (e.g. Full Body 3x/week, Upper/Lower
  4x/week, Push/Pull/Legs) that load as normal editable plans.
- **Live workout:** start today's session or an empty one. Log each set (weight, reps,
  optional RPE, set type: warm-up / working / drop / failure). Previous performance shown
  beside each exercise. Rest timer that starts automatically after a set, with +/-15 s.
  Reorder, add or swap exercises mid-workout. Finish screen with duration, volume, PRs hit.
  An unfinished workout survives a page reload or the app being closed.
- **Progression engine:** per exercise slot, choose a rule: none, linear (add X kg after a
  successful session, deload Y% after N failures), double progression (raise reps across a
  range, then raise weight and reset reps). Pure functions, fully unit-tested.
- **History & stats:** list of past workouts with detail view and edit/delete; per-exercise
  chart of estimated 1RM and top set over time; PR table (heaviest weight, best e1RM, most
  reps at a weight, highest session volume); weekly sets per muscle group.
- **Body tracking:** daily body-weight entry with a trend line (7-day moving average);
  optional measurements (waist, chest, arm, thigh, body-fat %).
- **Units & settings:** kg/lb (stored canonically in kg), week starts Monday/Sunday,
  light/dark/system theme, default rest time, plate/bar settings for a plate calculator.
- **Offline-first & sync:** works fully offline; syncs between devices when online.
- **Import/export:** full JSON backup and restore; CSV export of sets and body weight.
- **Self-hosting:** one `docker compose up -d`, one data volume, configured by env vars.

### v2 (after MVP is solid)
Web Push rest-timer and workout reminders; plate calculator UI; deload weeks; timed and
distance exercises (planks, carries, cardio); CSV import from a generic format; i18n
scaffolding (English only at first); admin page (users, invites, storage); public demo mode
with seeded fake data; muscle body-map visual (draw your own SVG).

### Later
Native wrappers (Capacitor), watch apps, health-platform sync, an optional AI coach.

Use `docs/RESEARCH.md` to propose additional features; add them to v2/Later, never to MVP
without asking me.

## 2. Architecture

Monorepo using npm workspaces:

```
/packages/core     TypeScript domain logic shared by web and server: types, zod schemas,
                   1RM, progression rules, PR detection, volume, unit conversion, sync merge.
                   Zero runtime deps besides zod. 100% of training maths lives here.
/apps/server       Node 22 + TypeScript, Fastify, SQLite (better-sqlite3) with versioned SQL
                   migrations, @simplewebauthn/server, argon2 for passwords.
/apps/web          React 19 + Vite + TypeScript, React Router, Zustand or TanStack Query,
                   IndexedDB via Dexie for the offline store, vite-plugin-pwa, a small chart lib
                   (uPlot or Recharts). Own small components, no heavy UI kit.
/apps/site         (later) marketing site for gymlantern.com, plain HTML/CSS or Astro.
/docs              SELF_HOSTING.md, ARCHITECTURE.md, RESEARCH.md, REFERENCES.md, CONTRIBUTING.md
```

### Data model (design it yourself; minimum entities)
User, Credential (passkey), Session, Invite, Exercise, Plan, PlanSession, PlanSlot,
Workout, WorkoutExercise, SetEntry, BodyWeightEntry, MeasurementEntry, Settings.
Every syncable row: `id` (UUIDv7, generated on the client), `userId`, `createdAt`,
`updatedAt` (hybrid logical clock or server-issued timestamp), `deletedAt` (soft delete).

### Sync
- Client keeps a local outbox of changed rows in IndexedDB.
- `POST /api/sync` with `{ since: cursor, changes: [...] }` returns
  `{ accepted, conflicts, changes: [...], cursor }`.
- Conflict rule: per-row last-writer-wins on `updatedAt`, with deletes winning ties. The merge
  function lives in `packages/core`; client and server use the SAME code.
- Property-based tests (fast-check) proving merge is commutative and idempotent.

### Security requirements
- Session: random 256-bit token, stored hashed in SQLite, httpOnly + Secure + SameSite=Lax
  cookie, configurable lifetime, rotation on sign-in, revoke-all endpoint.
- CSRF: reject state-changing requests whose `Sec-Fetch-Site` is not same-origin (fall back
  to Origin check).
- Rate limiting on all auth routes (per IP and per account), with lockout backoff.
- Input validation with zod on every route; body size limit; strict CORS (none by default).
- Passkeys need `RP_ID` and `ORIGIN` env vars; document the HTTPS requirement.
- Security headers (CSP, frame-ancestors none, nosniff, referrer-policy).
- No telemetry, no external network calls at runtime.

### Config (env vars)
`PORT`, `ORIGIN`, `RP_ID`, `RP_NAME`, `DATA_DIR`, `SESSION_DAYS`, `INVITE_ONLY`,
`PASSWORD_LOGIN`, `LOG_LEVEL`. Provide `.env.example` with comments.

## 3. Frontend design: anti-slop rules

Act as a senior frontend design engineer. The standard is **intentional and polished**, not
merely functional or flashy. Treat libraries as a toolkit, not a shopping list: never add an
animation, dependency, page, section, effect or feature just because it is available.

### antislop skills (install before any UI, copy or code work)
Use antislop (MIT, https://github.com/miqdadbadjuber/anti-slop) as a filter on top of these
rules. Install it in this project at step 2:
```bash
npx antislop-ai            # pick: this project, all skills, your agent
# or, in Claude Code:
# /plugin marketplace add https://github.com/miqdadbadjuber/anti-slop
# /plugin install antislop@anti-slop
```
- **Mode: DURING.** Session instruction from the founders: "Use antislop during for this
  session." Apply the rules while writing, and run its Delivery Gate before each milestone.
- Load the matching skill before each kind of work:
  `antislop` (core, always) · `antislop-ui` (interfaces, colour, motion, performance) ·
  `antislop-layoutmobile` (phone-to-desktop reflow, tap targets) · `antislop-human`
  (contrast checker, keyboard, focus, states) · `antislop-copywriting` (UI text, marketing
  copy, README) · `antislop-code` (comment hygiene; never changes code behaviour).
- Its three always-on steps apply here: settle design direction before building (R-37:
  `DESIGN.md` is mandatory; anything built without it is labelled "draft without direction"),
  ask before creating any asset (R-23), and run the Delivery Gate before delivering.
- Its Hard Gate rules are absolute in GymLantern, notably R-17 (no invented data or numbers),
  R-18 and R-36 (no fake testimonials or claims), R-25 (contrast), R-27 (UI states),
  R-32 (keyboard), R-34 (every theme you ship works: light and dark), R-35 (verify before
  you deliver), R-38 (real content or honest placeholder).
- If `DESIGN.md` asks for something antislop flags, name the element and the rule and ask
  the founders to keep or drop it; record the answer in one line in `DESIGN.md`.
- After MVP, run antislop in AFTER mode once as an audit
  (`anti-slop/audit-001-YYYY-MM-DD.md`) and fix only the findings the founders approve.
- Credit antislop in NOTICE.md (MIT, Copyright (c) 2026 Miqdad Badjuber).

### Design Read (write this into `docs/DESIGN.md` before building UI)
```yaml
Design Read:
  artifact: installable web app (PWA), mobile-first workout tracker
  audience: lifters logging sets mid-workout, often one-handed, sweaty, in bad gym lighting
  visual-language: [choose a specific family, e.g. "warm utilitarian: lantern-amber accents on deep charcoal"]
  mode: greenfield
  visual-variance: 4
  motion-intensity: 3
  information-density: 7
  asset-dependence: 2
  brand-fidelity: 8
```
These dials must drive real decisions. Before building, offer me **three clearly different
visual directions** (palette, type pairing, one sample screen each) and wait for my pick.
Then define a compact design system as CSS custom properties: palette (light + dark),
typography, spacing scale, radius, shadows, motion durations/easings.

### Build with restraint
- Semantic HTML, clear component boundaries, CSS Grid/Flexbox, design tokens as CSS custom
  properties, fluid sizing with `clamp()`.
- Design for phone first (the live workout screen is the hero), then tablet and desktop;
  don't shrink a desktop layout.
- Every interactive element has hover, focus, active, disabled, loading, empty and error states.
- Big tap targets (>= 44 px) on the workout logger; numbers are the dominant content.
- Copy is short, specific and human. No fake stats, testimonials, reviews or claims.
- Avoid: gradient-heavy "tech" looks, card grids where content doesn't call for them, emoji
  as icons, excessive blur/glass/parallax/particles, scroll-triggered animation, default
  display fonts.
- Motion: CSS for simple transitions; GSAP only for genuinely coordinated sequences (e.g. a
  PR celebration); no smooth-scroll hijacking. Respect `prefers-reduced-motion`. Clean up
  timers, listeners and animation frames on unmount. Animation is never the only way to
  reach content.
- Any WebGL/canvas effect (only if it truly earns its place, e.g. a subtle lantern glow on
  the marketing site) needs a static fallback, lazy loading and reduced-motion support.
- Logo: a simple, original lantern mark as SVG, legible at 16 px, with mono and colour
  variants plus PWA icon sizes.
- Use SiteInspire / Prompt Motion only for general inspiration; never copy another site's
  assets, copy or distinctive design.

### Accessibility, performance, reliability
Check: semantic landmarks and heading order, keyboard navigation with visible focus,
accessible names and labels, alt text (empty for decorative), contrast (WCAG AA), touch
targets, reduced motion, no horizontal overflow at 360 px, no broken imports/links/interactions.
Never claim an accessibility or performance result you didn't actually measure.

### Final self-review (before each UI milestone)
1. Design direction: coherent, not assembled from unrelated effects?
2. Hierarchy: is the most important content (current set, weight, reps, timer) dominant?
3. Craft: consistent spacing, type, colour, alignment and states?
4. Functionality: does every element work and serve a purpose?
5. Originality: distinctive, not a generic template, and not a copy of any other tracker?

## 4. Quality bar
- TypeScript strict mode everywhere; ESLint + Prettier configured; no `any` in core.
- Vitest for core and web, Vitest or node:test for server, Playwright for 3 to 5 key flows
  (sign up, plan a session, log a workout offline then sync, export).
- `packages/core` test coverage at least 90%.
- GitHub Actions: install, lint, typecheck, test, build and docker build on every PR.
- Never say "tested in the browser" unless Playwright or a real browser run actually happened.

## 5. Build order (commit after each step, keep everything green)
1. Research pass: write `docs/RESEARCH.md` (features, user pain points, ideas) from the idea
   references in section 0. No code reading.
2. Scaffold monorepo, licence (`[MIT or AGPL-3.0-or-later]`, Copyright (C) 2026 Aditya Manoj and Athul Murali),
   NOTICE.md, CI, lint/format/typecheck.
3. `packages/core`: types and zod schemas, unit conversion, 1RM (Epley, Brzycki, pick a
   default), PR detection, volume, progression rules, sync merge. Tests first.
4. Server: migrations, auth (passkey + optional password), sessions, rate limiting, CSRF,
   CRUD routes, `/api/sync`, `/api/health`, backup export/import. Integration tests.
5. Design Read + three visual directions; wait for my choice; then the design system.
6. Web shell: routing, theme, auth screens, settings, offline store + sync loop.
7. Exercise library + custom exercises (write the seed data yourself).
8. Plans + starter templates.
9. Live workout logger + rest timer + crash-safe in-progress state.
10. Finish flow applying progression rules; history; stats and charts; PRs.
11. Body weight + measurements.
12. PWA install, offline tests, Dockerfile (multi-stage, non-root), docker-compose.yml,
    SELF_HOSTING.md.
13. Playwright end-to-end tests, accessibility pass, performance pass.
14. Then the v2 list, one feature per PR.

## 6. How to work
- Before each step, write a short plan in `docs/progress.md`; after it, record what was done,
  how to run it and the test results.
- Small, readable modules. No file over ~400 lines.
- Ask me only for decisions that change product scope or visual direction; otherwise choose
  sensibly and note it.
- At the end of each step, give me the exact commands to run it locally.

## 7. Required root README.md (keep updated; final version at MVP)
Must reflect the actual implementation:
1. What GymLantern is and what was built.
2. Install, run, build and test commands (only verified ones, or labelled unverified).
3. Key design and implementation decisions.
4. Libraries actually used, with versions from package files, and why.
5. Asset sources and licence/attribution notes.
6. Testing and accessibility evidence, and what was not tested.
7. The release checklist below, filled in honestly.
8. Remaining owner actions and launch blockers.
9. A "Founders" section: Aditya Manoj (co-founder) and Athul Murali (co-founder).

Statuses: `- [x] DONE: ...` only when completed and verified (with evidence);
`- [ ] PENDING: ...` otherwise; `N/A: [reason]` for items that don't apply. Never put
credentials or private keys in the README.

### Release checklist
**Build and handoff**
- [ ] Setup, run, build and test commands documented accurately.
- [ ] Routes, components, features and interactions described.
- [ ] Every dependency listed with a reason.
- [ ] Assets and licensing/attribution documented.
- [ ] Links, forms, navigation and interactive states checked, or marked unverified.
- [ ] Responsive behaviour checked at 360, 768, 1280 px, or marked unverified.
- [ ] Keyboard access, focus, labels, alt text and reduced motion reviewed.
- [ ] Console, build and test results reported only if actually checked.
- [ ] Known defects, limitations and assumptions listed.

**SEO (marketing site and public pages; the logged-in app should be `noindex`)**
- [ ] Distinct titles and meta descriptions on indexable pages.
- [ ] Semantic structure and descriptive link text.
- [ ] Open Graph / social metadata.
- [ ] Canonical URLs use the real production domain (gymlantern.com), not guessed.
- [ ] `robots.txt` and sitemap correct; no stray `noindex` on public pages.
- [ ] Structured data only where accurate (e.g. SoftwareApplication).
- [ ] Broken links, redirects and 404 checked, or marked pending.
- [ ] Core Web Vitals recorded only if measured on a real deployment.
- [ ] No ranking or indexing outcome promised.

**Google Search Console**
- [ ] Production domain and property type documented.
- [ ] Domain ownership verified by the owner, or marked as an owner action.
- [ ] Sitemap submitted, or marked as an owner action.
- [ ] Key URLs inspected after launch, or marked pending.
- [ ] Indexing and security reports scheduled for review.
- [ ] No credentials in the codebase; unverified steps labelled pending.

**Privacy, cookies, consent, legal**
- [ ] Owner has identified jurisdictions and audiences, or marked pending.
- [ ] Data collection, forms, analytics, embeds and third parties reviewed (app has none by design).
- [ ] Privacy notice accurately describes data, purposes, retention and rights (self-hosted:
      the server owner is the controller; document this).
- [ ] Cookie/storage behaviour reviewed (session cookie is strictly necessary; no tracking).
- [ ] No consent banner unless nonessential tracking is actually added.
- [ ] Forms explain data use and collect nothing unnecessary.
- [ ] Policy documents match real behaviour and are marked DRAFT pending legal review.
- [ ] Terms, disclaimers (not medical advice), age safeguards assessed where relevant.
- [ ] Licence compliance: NOTICE.md complete; no third-party code or assets copied.
- [ ] Remaining owner decisions and legal review listed.

Definition of done for MVP: a fresh clone runs with `docker compose up -d`; a new user can
sign up with a passkey, load a starter plan, log a full workout offline on their phone, see it
sync to their laptop, see a PR and a progress chart, and export a JSON backup that restores
cleanly on a new server. All CI checks pass and the README checklist is honestly filled in.
