# HSC Study Pack — Build Plan & Product Roadmap

*Companion to `HSC Study Pack — Build Handoff Spec.md`. That doc is the source of truth for the product's content, data model, and interactions. This doc is the **go-forward plan**: the decisions we've locked, the architecture, the content pipeline, and a phased roadmap — tuned to a web-first app (with a downloadable desktop app), seeded for 8–10 subjects, and built to prove itself on one student before opening up.*

*Last updated: 13 July 2026.*

---

## 1. Decisions locked this session

These came out of the planning conversation and set the frame for everything below.

| Decision | Choice | Why it matters |
|---|---|---|
| **The hero** | The **confidence map** — the dot-point traffic-light tracker. The schedule and recipes serve it. | This is the unique insight. Protect it; don't let feature sprawl bury it. |
| **Out of scope** | **No AI tutor.** Deliberately excluded. | It was the most expensive and complex piece; cutting it keeps the build focused on the tracker. |
| **Ambition (for now)** | **Better version for you first**, architected so others can use it later. | "Prove on one, architect for many." Ship deeply for one real user before scaling. |
| **Content scope target** | **Top 8–10 popular subjects**, seeded deeply. | Reach beyond your four without taking on all ~120 HSC courses at once. |
| **Delivery model** | **brain.fm-style**: one web app that runs in the browser, **plus a downloadable app** on top. Desktop is the first download target. | Web = zero-friction access + easy sharing later. Download = offline, feels like "a real app," lives in the dock. Same codebase both ways. |
| **This session's output** | **Plan first, build later.** | This document. Building starts in a later session once you're happy with the direction. |

---

## 2. Product thesis & the problem

**The problem:** HSC students study hard but mostly study *blind*. The syllabus dot points are the actual contract with the examiner — every question maps back to one — yet almost no student tracks their learning against them. They measure effort in hours and pages, not in "which specific things can I *produce* under exam conditions." The result: they over-study what already feels comfortable, avoid what's weak, and walk into the exam with an inaccurate map of their own readiness.

**The wedge:** *Confidence per dot point is the unit of truth.* The student self-assesses each real syllabus point with a traffic-light colour; the plan reacts to that map. Most study tools track completion or time — almost none make a student honestly rate calibrated confidence against the real syllabus. That's the thing that makes someone say "oh, this is different."

**What students struggle with (design targets):** poor calibration (mistaking "I read it" for "I can do it"); overwhelm and not knowing where to start; avoidance of weak areas; recognition vs retrieval; no reliable self-marking to HSC standard; command-term confusion (describe vs explain vs evaluate); decoding essay questions; motivation dips; and the cost/access gap around good tutoring.

---

## 3. Target users & scope

- **Primary (v1):** you — a NSW Year 12 HSC student. Every design call optimises for one real, engaged user.
- **Secondary (built for, not launched to):** other NSW HSC students taking the seeded subjects.
- **Explicitly out of scope for now:** other states/curricula, teacher/parent dashboards, social features. Note them as future doors, don't build for them.

---

## 4. Guiding principles (non-negotiables)

1. **The map is sacred.** The tracker must be fast, honest, and always one tap from view. Everything else is secondary.
2. **Content-as-data.** Subjects are JSON, rendered generically. A new subject is new data, never new code.
3. **Your data is yours and it never disappears.** Fix the prototype's core flaw (colours trapped in one browser). Local-first storage + always-available JSON export/import.
4. **Web-first, download-optional, one codebase.** The browser version and the downloadable app are the same app.
5. **Offline works.** The tracker, schedule, and recipes must function with no network.
6. **Warm, plain-English, never "you're behind."** Carry the spec's tone into every string and empty state.
7. **Accurate or nothing.** Syllabus content that's wrong destroys trust faster than missing content. Every subject verified against current NESA documents.

---

## 5. Architecture recommendation

### 5.1 The shape: one web app, downloadable as a desktop app

The brain.fm model maps cleanly onto a standard modern stack:

- **Core = a web app** (runs at a URL, works in any browser).
- **Made installable as a PWA** — "Add to home screen / Install app" straight from the browser, offline-capable, no store needed. This is the lightest possible "download the app."
- **Packaged as a downloadable desktop app** using **Tauri v2**, which wraps the *same* web build into a real installer (`.exe` / `.dmg`) that lives in the dock and runs offline. Desktop is the first download target, per your call.
- **Mobile app later** (if wanted) via the same web build through Capacitor or PWA install — a port, not a rewrite.

One codebase, three surfaces (browser → installable PWA → desktop installer), added in that order.

### 5.2 Why Tauri v2 for the desktop download (not Electron)

Current (2025–26) trade-offs, verified this session:

- **Bundle size:** Tauri desktop apps are dramatically smaller — commonly ~3–10 MB vs Electron's ~85–150 MB, because Tauri uses the OS's built-in webview instead of shipping a whole Chromium. For a "download the app" button, a tiny installer is a real UX win.
- **Memory/performance:** Tauri idles lighter (Rust backend, system webview).
- **Offline + local storage:** Tauri bundles a real local database (SQLite) cleanly — exactly what we need to replace fragile `localStorage`.
- **The Electron case:** more mature, larger ecosystem, all-JavaScript (no Rust). Choose it only if whoever builds strongly prefers a pure-Node toolchain. For this app, Tauri's size/offline profile wins.

**Recommendation:** Web app + PWA first, Tauri v2 wrapper for the desktop download. Electron is the documented fallback.

### 5.3 Suggested stack

- **Frontend:** React + TypeScript (or SvelteKit if preferred). Component-driven, since dot-point rows, progress bars, and pickers repeat everywhere.
- **Styling:** CSS variables using the §8 design tokens from the handoff spec (Tailwind optional).
- **Local persistence:** IndexedDB (via Dexie) for the browser/PWA; SQLite via Tauri for desktop. Abstract storage behind a single `ProgressStore` interface so the UI never knows which backend it's talking to.
- **Sync (later, optional):** a light backend (e.g. Supabase) *only* when cross-device sync becomes worth it. Not needed for "better version for me first." JSON export/import covers device-moves until then.
- **Content:** seed subjects shipped as versioned JSON, loaded at runtime.

### 5.4 The storage fix (the prototype's #1 limitation)

Replace the single `localStorage` key `hscTraffic_v1` with:

1. A storage abstraction (`ProgressStore`) with `get/set/clear/exportJSON/importJSON`.
2. IndexedDB implementation for web/PWA; SQLite implementation for desktop.
3. **Export/Import JSON always present**, regardless of backend — this is the user-controlled backup and the bridge between devices until real sync exists.
4. A one-time migration that reads any existing `hscTraffic_v1` blob and imports it, so nothing you've already coloured is lost.

---

## 6. Data model (evolved from the spec)

Keep the handoff spec's model (§6) as the base. Extensions for multi-subject scale, content versioning, and progress-over-time:

```ts
// --- Content (shipped as JSON, versioned) ---
interface SubjectContent {
  key: string;              // 'bio', 'eng-adv', 'math-std2', ...
  name: string;             // 'Biology'
  tag: string;              // 'Bio'
  colorToken: string;       // subject tag colour from §8
  examInfo: string;
  shape: 'dotpoint' | 'skill' | 'topic-skill'; // see §7.3 — subjects aren't all the same
  syllabusVersion: string;  // e.g. 'NESA-2017-amended-2024' — provenance for trust
  lastVerified: string;     // ISO date the content was checked against NESA
  modules: Module[];
}

interface ContentBundle {
  schemaVersion: number;
  subjects: SubjectContent[];
}

// --- Student state (persisted, per profile) ---
type Confidence = 'dg' | 'lg' | 'yl' | 'or' | 'rd' | null;

interface ProgressEntry {
  confidence: Confidence;
  updatedAt: string;        // ISO timestamp — enables progress-over-time history
}
interface Progress { [dotPointId: string]: ProgressEntry; }

// history: an append-only log so we can show reds turning green over days
interface ProgressEvent {
  dotPointId: string;
  from: Confidence;
  to: Confidence;
  at: string;               // ISO timestamp
}

// --- Profile (single now, list-ready for later) ---
interface Profile {
  id: string;
  name: string;
  subjectKeys: string[];    // which seeded subjects this student is taking
  progress: Progress;
  history: ProgressEvent[];
}
```

Key changes vs the spec: **timestamps on every confidence change** (unlocks progress history for free), a **`history` event log**, **content provenance** (`syllabusVersion` + `lastVerified`) so trust is auditable, a **`shape`** field so different subject types render correctly, and a **`Profile`** wrapper so multi-student is a data change, not a rebuild.

Derived computations (buckets, per-subject bars, "needs most work," overall tally) are unchanged from the spec §6.

---

## 7. Content pipeline — the real work

This is where the effort actually lives. The engine is a few weeks; **accurate content for 8–10 subjects is the mountain.**

### 7.1 Recommended seed set

Your four subjects, plus the highest-enrolment courses to reach ~10. Enrolments are NESA's 2025 HSC snapshot (~1 Sept):

| # | Subject | 2025 enrolments | Status |
|---|---|---|---|
| 1 | English Standard | 33,978 | add (biggest cohort in NSW) |
| 2 | Mathematics Standard 2 | 32,243 | add |
| 3 | English Advanced | 26,377 | **you have it** |
| 4 | Biology | 20,920 | **you have it** |
| 5 | Business Studies | 20,783 | add |
| 6 | PDHPE | 18,069 | add (see HMS note) |
| 7 | Mathematics Advanced | 17,023 | add |
| 8 | Chemistry | 10,494 | add (pairs with Bio/Physics) |
| 9 | Visual Arts | 9,089 | **you have it** |
| 10 | Physics | 8,919 | add |

Plus **Health & Movement Science (HMS)** — you already have it, and it's the *new* Stage 6 course (first HSC exam 2026) replacing much of the PDHPE space, so few competitors will have good HMS material. That's an early-mover edge worth keeping even though PDHPE still shows the bigger legacy enrolment. **Verify the PDHPE ↔ HMS transition status against NESA** before deciding whether to seed one, the other, or both.

This set covers the great majority of HSC students: the two Englishes, two Mathematics, three Sciences, Business Studies, PDHPE/HMS, and Visual Arts.

### 7.2 Sourcing & validation process (per subject)

1. Pull the current syllabus from NESA and extract every dot point, grouped Module → Inquiry question (mirroring your existing structure).
2. Write the plain-English **"what to nail"** line per dot point — the warm, exam-focused gloss that is the pack's signature. This is the craft step and can't be fully automated.
3. Set an **advisory priority** (`red`/`amber`/`green`) as a *default suggestion* only — clearly the app's opinion, not the student's.
4. Stamp `syllabusVersion` + `lastVerified`.
5. **Human accuracy check** against the NESA document before a subject goes live. Wrong content is worse than absent content.

### 7.3 Subjects aren't all the same shape

Your spec already handles two shapes; scaling to Maths adds a third. The generic engine must render all three (hence the `shape` field):

- **`dotpoint`** — Sciences, HMS: Module → Inquiry question → dot points. The classic case.
- **`skill`** — English, Visual Arts: "skill checkpoints" the student colours (your spec already does this).
- **`topic-skill`** — Mathematics: confidence sits on *topics and problem-types* (e.g. "differentiate composite functions"), not syllabus prose. Design the tracker so a "dot point" can be a skill/problem-type, and lean on Recipe B ("master a problem type") for Maths.

Getting Maths right is the one genuinely new content-design problem in the seed set. Worth a small dedicated pass.

### 7.4 Effort signal

Roughly: Sciences/HMS are dot-point-dense (more rows, more "what to nail" lines); English/Visual Arts are lighter (skill checkpoints); Maths needs the new topic-skill shape designed once, then it's mechanical. Budget the bulk of content time on the three sciences + Maths.

---

## 8. Feature scope by surface

Carrying the spec's MVP/stretch split, re-cut for web-first + tracker-hero:

**Must-have (the hero + its frame):** subject sections (modules → inquiry groups → dot-point rows); colour picker (5 colours + clear) with instant repaint; per-subject progress bars + "needs most work" flag + overall tally; priority chips + exam-info panels; study-recipes screen; weekly-schedule screen (a personal *template* for others, seeded with yours); durable local storage; export/import; offline; responsive; printable.

**Fast-follow:** installable PWA; downloadable desktop app (Tauri); progress-over-time history (reds → greens across days); the gentle "re-colour" nudge from the schedule's last block.

**Later:** spaced-repetition flashcards seeded from red/orange dot points (plain Q→A cards, *not* AI); multi-profile support; a gentle daily-nudge notification. *(An AI tutor is explicitly out of scope for this build.)*

---

## 9. Phased roadmap

Each phase is a shippable, usable increment. You could stop after any one and still have something better than the prototype.

**P0 — Foundation (engine + storage fix).**
Scaffold the web app (React/TS). Load subjects from JSON. Rebuild the tracker (rows, colour picker, per-subject bars, tally) against a `ProgressStore` abstraction backed by IndexedDB. Ship export/import + migration from the old `localStorage`. Seed with your existing four subjects (exact copy from `study-pack.html`).
*Done when:* your four subjects work end-to-end in a browser, colours survive reload, and nothing from the old prototype is lost.

**P1 — Content screens + PWA.**
Exam-info panels, recipes screen, weekly-schedule screen. Make it an installable PWA (the first "download the app"). Print styles. Mobile-responsive polish.
*Done when:* the full personal pack runs offline, installs from the browser, and prints cleanly.

**P2 — Desktop download.**
Wrap the web build in Tauri v2; SQLite-backed `ProgressStore` for desktop; produce a signed installer. Add progress-over-time history (using the timestamped events from P0).
*Done when:* there's a downloadable desktop app that runs offline and shows reds turning green over time.

**P3 — Multi-subject content.**
Source, write, and NESA-verify the remaining subjects to reach the top 8–10 (§7.1), including the Maths `topic-skill` shape. Add a subject picker so a student enables only their subjects.
*Done when:* 8–10 verified subjects are selectable and the app is genuinely useful to students who aren't you.

**P4 — Retention & polish (optional).**
Spaced-repetition flashcards auto-seeded from red/orange dot points (plain Q→A, no AI), a gentle daily nudge, and multi-profile support if you decide to open it to other students.
*Done when:* weak dot points flow into flashcards the student can drill, and the app is ready to share.

---

## 10. Risks & open decisions

- **Content accuracy & maintenance** *(highest risk)*: syllabi get amended; wrong dot points break trust. Mitigate with `lastVerified` stamps and a periodic re-check. Decide who owns content upkeep.
- **PDHPE vs HMS**: confirm the transition status with NESA and decide which to seed (§7.1).
- **Maths content shape**: needs the dedicated `topic-skill` design pass (§7.3) — the one real unknown in the seed set.
- **Copyright**: NESA syllabus *outcomes/dot points* are official wording. Confirm acceptable use / attribution before distributing to other students (fine for personal use; worth checking before you share).
- **Scope discipline**: the AI tutor is intentionally excluded — it was the biggest source of cost and complexity. Keep the focus on the confidence tracker and getting its content right.
- **"For me first" vs "for others"**: the architecture supports both, but keep resisting feature creep until your own daily use proves the core.

---

## 11. Immediate next actions

1. **You:** confirm the seed subject list in §7.1 (and the PDHPE/HMS call), and confirm React/TS + Tauri is an acceptable stack.
2. **Next build session (P0):** scaffold the web app, port your four subjects' exact copy from `study-pack.html` into JSON, and build the tracker on a durable `ProgressStore` with export/import + old-data migration.
3. **In parallel, whenever:** begin the content-sourcing pass for the first new subject (suggest **Biology's sibling sciences** or **English Standard**, since their shapes are closest to what you've already built).

---

*Content and exam specifications must be verified against current NESA syllabus documents before shipping to anyone beyond yourself. Enrolment figures: NESA 2025 HSC enrolment snapshot. Desktop-packaging trade-offs reflect the 2025–26 Tauri v2 / Electron comparison.*
