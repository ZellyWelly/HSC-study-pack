# HSC Study Pack — Build Handoff Spec

**Purpose of this doc:** a complete, self-contained brief a developer (or coding agent) can use to build the "HSC Study Pack" into a standalone app. It captures the product, data model, interactions, design system, and full seed content. A working single-file HTML prototype already exists (`study-pack.html`) — treat it as the reference implementation and source of truth for exact copy.

---

## 1. Summary

A personal, interactive HSC (NSW, Australia) revision companion for a single student. The core loop: the student sees **every syllabus dot point** for their subjects, **traffic-light colours** each one by confidence, and the app **shows per-subject progress** so they know where to study. Around that sit a **study-recipe playbook** and a **weekly schedule**.

Built originally as a single-file HTML artifact; the goal now is a proper standalone app (offline-capable, mobile-friendly, persistent, extensible to more subjects/students).

---

## 2. Target user & context

- **Who:** a Year 12 NSW HSC student. Design for one primary user but architect so multiple students/subjects are possible.
- **Subjects in v1:** Health & Movement Science (HMS), Biology, English Advanced, Visual Arts.
- **Key insight driving the design:** confidence per syllabus dot point is the unit of truth. The student self-assesses (traffic lights), and the plan reacts to it.
- **Tone:** warm, encouraging, plain-English. Avoid academic stiffness. Never make the student feel behind.

---

## 3. Core concepts

1. **Dot-point tracking.** Each subject is broken into its real syllabus dot points, grouped by Module → Inquiry-question. Every dot point has a clickable confidence marker.
2. **5-colour traffic-light system** (the student's own scheme):
   | key | label | bucket |
   |---|---|---|
   | `dg` | Dark green — Mastered | strong |
   | `lg` | Light green — Good | strong |
   | `yl` | Yellow — OK | ok |
   | `or` | Orange — Shaky | weak |
   | `rd` | Red — Weak | weak |
   | (unset) | — | untracked |
3. **Priority chip** (separate from the student's colour): the app's *suggested* priority per dot point — `RED`/`AMBER`/`GREEN`. Advisory; does not change.
4. **Progress bars.** Per subject, compute % weak / ok / strong across its dot points; flag the subject with the most work.
5. **Study recipes.** A reusable "how to study" playbook (a 4-step study loop + 9 named recipes). The schedule references recipes by letter.
6. **Weekly schedule.** Day-by-day blocks (~1–1.5 hr each) with times, meals, exercise; each block names a topic + a recipe.

---

## 4. Feature list

### MVP
- Subject sections, each with modules → inquiry groups → dot-point rows.
- Click a dot point → colour picker (5 colours + clear) → saves.
- Per-subject progress bars (weak/ok/strong %) + "needs most work" flag; overall tally.
- Priority chips + exam-info panel per subject.
- Study-recipes reference screen.
- Weekly schedule screen with time-blocks.
- Persistence (survives reload/close). Export/import of progress.
- Mobile responsive; printable.

### Stretch
- Spaced-repetition flashcards seeded from red/orange dot points.
- Multi-student / multi-subject; add new subjects from syllabus templates.
- Progress history over time (see reds turning green across days).
- Notifications / daily nudge.

---

## 5. Information architecture (screens)

1. **Dashboard / "Make it yours"** — colour key, overall tally, per-subject progress bars, controls (print, reset, export/import).
2. **Subject screen** (one per subject) — exam-info panel + dot-point checklist grouped by module/inquiry, each row with priority chip + confidence marker + its own progress bar.
3. **Study recipes** — the study loop + 9 recipes.
4. **Weekly schedule** — day cards with time-guide strip + blocks.

---

## 6. Data model

```ts
type Confidence = 'dg' | 'lg' | 'yl' | 'or' | 'rd' | null; // null = untracked
type Priority = 'red' | 'amber' | 'green';

interface DotPoint {
  id: string;            // stable, e.g. "hms-roles", "bio-freq"
  label: string;         // bold lead, plain English
  detail?: string;       // one-line "what to nail"
  priority: Priority;    // app's suggested priority (advisory)
  // student state (persisted separately, keyed by id): confidence: Confidence
}

interface InquiryGroup { title: string; dotPoints: DotPoint[]; }
interface Module { title: string; groups: InquiryGroup[]; }

interface Subject {
  key: 'hms' | 'bio' | 'eng' | 'va';
  name: string;          // "Health & Movement Science"
  tag: string;           // short chip, "HMS"
  examInfo: string;      // exam-structure snippet (see §9)
  modules: Module[];     // English & Visual Arts are "skill checkpoints", same shape
}

interface Recipe { letter: string; title: string; steps: string[]; }

interface Block {
  time?: string;         // "9:00"
  kind: 'study'|'move'|'meal'|'lecture'|'rest'|'travel'|'review';
  subject?: Subject['key'];
  priorityChip?: Priority;
  title: string;
  recipe?: string;       // e.g. "A", "C", "G + F"
  steps?: string[];
}
interface Day { name: string; note: string; timeGuide: string; blocks: Block[]; }

// Persisted student state:
interface Progress { [dotPointId: string]: Confidence; } // localStorage key "hscTraffic_v1"
```

### Derived computations
- **bucket(c):** `dg|lg → strong`, `yl → ok`, `or|rd → weak`, `null → untracked`.
- **Per-subject bar:** count buckets over that subject's dot points; render three segments (weak `#e4572e`, ok `#f4d03f`, strong `#2e9e5b`), remainder grey = untracked. Show `weak% / ok% / strong%`.
- **"Needs most work":** for each subject with ≥1 tracked dot point, score = `(weak*2 + ok) / total`; flag the highest.
- **Overall tally:** counts of each colour + untracked.

---

## 7. Interaction specs

- **Colour picker:** clicking a dot-point marker opens a popover with 5 colour swatches + clear (✕). Selecting sets `Progress[id]` and repaints; clearing deletes the key. Close on outside-click / scroll / resize.
- **Persistence:** MVP can use `localStorage` (key `hscTraffic_v1`, JSON map). For a real app prefer IndexedDB or a small backend so it survives across devices; keep an **Export/Import JSON** button regardless (the HTML prototype's colours live only in one browser — a known limitation to fix).
- **Reset:** confirm, then clear all.
- **Print:** hide interactive chrome; keep colours (`print-color-adjust: exact`).
- **Re-colour prompt:** the schedule's final block each study day tells the student to revisit and re-colour — surface as a gentle in-app nudge.

---

## 8. Design system

**Palette / tokens** (from prototype):
```
--ink:#1a1a2e; --muted:#5b6472; --line:#e3e7ee; --bg:#ffffff; --soft:#f6f8fb;
--accent:#5b4b8a (purple); --blue:#3a6ea5;
Priority chips: red #e4572e on #fdeae4 · amber #9a7000 on #fcf3d6 · green #2e9e5b on #e4f4ea
5 confidence colours: dg #1e7d3c · lg #6cc24a · yl #f4d03f · or #e88b1a · rd #e4572e
Subject tags: HMS #e4572e · Bio #2e9e5b · English #3a6ea5 · Visual Arts #a1489c
```
- Cards: white, 1px `--line` border, ~12–14px radius. Module title bars: bold caps, purple on `#e6e2f2`. Inquiry sub-headers: soft purple, normal weight.
- System font stack. Mobile-first; use simple `circle + text` rows, not dense tables.
- Accessible contrast; light/dark friendly progress bars.

---

## 9. Seed content

> Exact copy for every row lives in the reference file `study-pack.html`. Below is the structure + priorities to seed the app; pull final strings from the HTML.

### Exam-info snippets
- **HMS:** "One written paper, 100 marks · 3 hours (+10 min reading), three sections — multiple choice, short answer, extended response. All compulsory; roughly equal weighting across the two focus areas."
- **Biology:** "One written paper, 100 marks · 3 hours (+5 min reading). Section I = 20 marks multiple choice; Section II = 80 marks short-answer & extended-response. Equal weighting across Modules 5–8."
- **English Advanced:** "Paper 1 = Common Module (Texts & Human Experiences): unseen short-answers + essay (50 marks, 1.5 hrs). Paper 2 = Module A + B + C, one response each (25 marks each, 2 hrs)."
- **Visual Arts:** "Body of Work 50% + Written exam 50%. Written ~1.5 hrs — Section I short answers on unseen plates (25), Section II essay (25). Tested via the Frames, the Conceptual Framework, and Practice."

### HMS — dot points (priority)
**Module: Health in an Australian & global context**
- *How healthy are Australians?* — Health status of Australians (amber) · Groups experiencing inequities (green) · Australia vs OECD (amber) · CVD, cancer & one other condition (green) · Impact of an ageing population (amber)
- *How does the healthcare system work?* — Effectiveness of the system (amber) · Who's responsible: gov & non-gov (red) · Gov & non-gov collaboration (red) · Health expenditure (red) · Complementary healthcare (amber) · Critical health consumer (red) · Emerging challenges (red)
- *Technology & data* — Technology & health (amber) · New technologies & treatments (amber) · Digital health (amber) · Big data in health (red)
- *Actions* — The SDGs 3/4/10/11 (red) · Applying SDGs to a community (red)

**Module: Training for Improved Performance**
- *Personalising assessment* — Pre-exercise screening (green) · Fitness/performance testing (green) · Assessment → training programs (green)
- *How training influences performance* — Training types & methods (green) · Principles of training (amber) · Physiological adaptations (amber)
- *Individual vs group sports* — Designing a session (amber) · Yearly program (amber) · Psychological strategies (green) · Applying strategies & tactics (green)
- *Sleep, nutrition & supplementation* — Dietary & fluid requirements (amber) · Sleep, nutrition & hydration (amber) · Supplements (green)
- *Sustained performance* — Biomechanics (green) · Recovery strategies (green) · Technology for performance (green) · Injury management & TOTAPS (green) · Rehab & return-to-play (amber) · Drugs in sport (green)

### Biology — dot points (priority)
**Module 5 — Heredity**
- *Reproduction* — Mechanisms of reproduction (green) · Fertilisation & implantation (green) · Hormonal control of pregnancy & birth (amber) · Manipulating reproduction in agriculture (green)
- *Cell Replication* — Cell replication (green) · Replication & continuity (green)
- *DNA & Polypeptide Synthesis* — Forms of DNA (green) · Polypeptide synthesis (green) · Phenotypic expression (amber) · Proteins (green)
- *Genetic Variation* — Variation from meiosis (amber) · Inheritance patterns/modes (amber) · Pedigrees & Punnett squares (amber) · Frequencies in a population (red)
- *Inheritance Patterns in a Population* — Technologies / DNA sequencing & profiling (red) · Large-scale genetics data: conservation, disease, human evolution (red)

**Module 6 — Genetic Change**
- *Mutation* — Mutagens (green) · Types of mutation (green) · Somatic vs germ-line (green) · Coding vs non-coding DNA (green) · Causes of genetic variation (green) · Mutation, gene flow & drift (green)
- *Biotechnology* — Biotechnology uses (green)
- *Genetic Technologies* — Current genetic technologies (green) · Reproductive technologies (green) · Cloning (green) · Recombinant DNA (green) · Benefits of genetic technologies (green) · Biotech & biodiversity (green) · Social/economic/cultural contexts (green)

**Module 7 — Infectious Disease**
- *Causes* — Infectious diseases & transmission (amber) · Koch & Pasteur (green) · Disease in agriculture (amber) · Pathogen adaptations (green)
- *Responses & Immunity* (untested on original scans; default amber) — Plant response to a pathogen · Host responses · Innate & adaptive immunity · Immune response to exposure
- **Note:** the HSC Biology exam also covers **Module 8 (Non-infectious Disease & Disorders)** — not in v1 because the student hadn't studied it. Architect so a module can be added later.

### English Advanced — skill checkpoints (student colours them; priority optional)
- **Common Module (Texts & Human Experiences):** name form/composer/human experiences · 6–8 quotes+techniques+meaning · discuss anomalies/paradoxes · thesis that answers an unseen question · analyse an unseen text fast · plan+write an essay in ~40 min.
- **Module A (Textual Conversations):** know both texts/contexts · explain resonance/reimagining · 3–4 comparison points with paired evidence · write integrated (not text-by-text).
- **Module B (Critical Study):** know text as a whole · argue textual integrity/value · understand critical readings · defend own judgement.
- **Module C (Craft of Writing):** 2–3 adaptable pieces · studied mentor-text features · write to unseen stimulus under time · reflection statement.

### Visual Arts — skill checkpoints
Subjective frame · Cultural frame · Structural frame · Postmodern frame · Conceptual Framework (Artist↔Artwork↔World↔Audience) · 5+ case studies (2 frames each) · Section I (unseen plates) · Section II (timed essay).

### Study recipes
**The Study Loop** (any content topic): 1) **Learn** — read notes to the exact dot point (~10 min). 2) **Condense** — close book, rebuild in own words (~10 min). 3) **Test** — answer a question from memory (~10 min). 4) **Fix** — check in another colour; gaps → flashcards (~5 min).

**Nine recipes:**
- **A — Learn a content topic:** run the loop; finish with 3–4 exam-ready dot points + one real example; say it aloud.
- **B — Master a problem type** (pedigrees, Punnett, allele frequencies): copy one worked example → redo from scratch → 5–8 fresh questions → redo misses.
- **C — Prep an English module essay:** 3 past questions; underline key word; one-sentence thesis; 3 techniques+quotes; write one body paragraph (P-Q-A-link).
- **D — Timed exam practice:** 5 min plan · 35 min write · 10 min self-mark vs criteria; note one biggest fix.
- **E — Craft of Writing:** take one device from a mentor text → write to a past stimulus → 3–4 sentence reflection.
- **F — Build a Visual Arts case study:** one page per artist (artwork · materials · intention · world · audience · best 2 frames); say from memory; 5+ artists.
- **G — Visual Arts Section I:** read plate 30s → read question + mark value → answer in frame/framework language; 3–4 per sitting.
- **H — Visual Arts Section II:** deconstruct question (which agency/frame) → 2 case studies → plan 3 paras → write timed.
- **I — Active recall & flashcards:** Q→A cards for facts you blank on; test, shuffle, re-test misses; 10–15 min.

### English "question attack" — the BOSS method (essay-question technique)
- **B — Box the directive:** *To what extent / How far* = argue a degree; *How / In what ways* = explain methods & effects; *Explore / Discuss* = multiple angles; *Represent* = composer's deliberate choices; *Analyse* = techniques→meaning; *Evaluate/Assess* = judgement.
- **O — Own the key idea:** underline the concept; everything links back to it.
- **S — State your line:** one-sentence thesis that takes a position AND reuses the question's key words.
- **S — Support ×3:** three topic sentences, each proving the line, each ending by echoing the key idea.

### Weekly schedule (template — Sydney, winter-holiday week; times flexible)
- **Block model:** ~1–1.5 hr each, 4 blocks/day, ~5–6 hr days. Standard day: `8:00 gym · 9:00 B1 · 10:30 B2 · 11:45 lunch · 12:30 B3 · 2:00 B4 · ~4:00 walk · ~6:00 dinner`. First block of the day runs long — expected.
- **Sun (x2):** travel days, no study. **Mon:** pilates then blocks. **Wed:** light (lecture in city + possible shoot) → one easy evening block.
- Each study block = subject + priority + recipe letter + 2–3 concrete steps. (Full block content in `study-pack.html` §7.)
- **Personalisation from live use:** HMS "health-in-Australia logistics" = weakest+most-avoided → schedule first while fresh, as memorisation ("who does what" map + flashcards). Biology "data analysis" = weak *skill* → drill past-paper graph/table questions, don't read. Visual Arts easy for this student → keep light despite the bar flagging it.


---

## 10. Tech recommendations

- **Frontend:** React + TypeScript (or SvelteKit). **PWA** for offline + mobile install.
- **State/persistence:** IndexedDB (Dexie) or light backend (Supabase) to sync across devices; always keep JSON export/import.
- **Content as data:** ship seed content (§9) as JSON; render generically. New subjects = new JSON, no code.
- **Styling:** Tailwind or CSS variables using §8 tokens.
- **Non-negotiables:** offline tracker/schedule; fast; mobile-first; accessible colours; printable.

---

## 11. Suggested build phases

1. **P1 — Tracker MVP:** subjects/modules/dot points from JSON, colour picker, persistence, per-subject bars, tally, export/import.
2. **P2 — Content screens:** exam-info panels, recipes, weekly schedule.
3. **P3 — Polish:** PWA/offline, print, mobile, progress-over-time history.

---

## 12. Source of truth

`study-pack.html` — the current working single-file prototype (HTML + CSS + vanilla JS). Contains exact copy for every dot point, the full weekly schedule with per-block steps, the recipes, exam snippets, and the working tracker logic (colour picker, per-subject bars, tally, persistence via `localStorage` key `hscTraffic_v1`). Use it for exact strings and as the interaction reference. Main limitation to fix: colours persist only in one browser's localStorage — replace with proper storage + export/import.

---

*Built from a live study-coaching session for a NSW Year 12 student. Syllabus content reflects: HMS 11–12 (2023), Biology Stage 6 (Modules 5–7 tracked; 8 to add), English Advanced, Visual Arts. Verify current exam specifications against NESA before shipping.*
