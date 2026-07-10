# DESIGN.md — Personal Progression System ("Career RPG")

**Status:** Architecture spec, v1.0 — ready for implementation
**Hosting model:** Static site (HTML/CSS/JS) on GitHub Pages. localStorage for instant saves. Manual export → committed JSON file. Zero runtime dependencies on AI models or hosted services.

**Assumptions made (stated per the brief, not asked):**

- You are the only user; there is no auth, no multi-user concern, and honest self-reporting is assumed. The system trusts you.
- "Instant save" means synchronous localStorage writes on every log action; the exported repo file is the durable backup, not the live store.
- Vanilla JS (no framework) is acceptable and preferred for a zero-build GitHub Pages deploy. Nothing in this spec requires React or a bundler.
- Weekly sustainable output across all four domains is roughly 300–450 XP under the values below; the leveling curve is calibrated against that.

---

## 1. Core architectural decision: event-sourced, config-driven

Before the XP tables, one decision shapes everything else: **the source of truth is an append-only log of events, not a stored XP total.** Every log action ("completed Medium box foothold," "training session," "problem set 4.2") is an immutable event with a UUID and timestamp. Level, XP totals, momentum, and per-domain breakdowns are *always recomputed* from the event log plus the config.

Why this matters:

- **Sync conflicts become trivial.** Merging two divergent copies (laptop vs. desktop) is a set-union of events by UUID. There is no "whose XP total is right" problem because no XP total is ever authoritative.
- **Config edits are safe.** If you rebalance an XP value later, derived state recomputes cleanly. (Tradeoff discussed in §9.)
- **The badge layer stays decoupled.** Badges are a separate small collection mutated only by explicit user action — they never touch the event log or the XP pipeline.

Everything user-editable lives in one config object (§7). All logic reads from config; nothing is hardcoded.

---

## 2. XP conversion tables and balancing methodology

### 2.1 Methodology (read this before arguing with the numbers)

A Hard HTB box, a training session, and a problem set aren't naturally comparable, so XP is derived from a common formula rather than vibes:

```
XP = H × 20 × D × N        (rounded to nearest 5)

H = expected focused hours for the activity (honest median, not best case)
D = difficulty multiplier:
      0.75 routine · 1.0 standard · 1.25 demanding · 1.5 hard · 1.75+ peak
N = novelty/artifact multiplier:
      1.0 default · 1.25 if it produces a durable artifact (writeup,
      shipped feature, committed notes) or applies a skill for the first time
```

The anchor: **20 XP ≈ one focused hour of standard-difficulty work.** Every value below is this formula plus rounding, with two deliberate deviations called out where they occur. When you add new activities later, price them with the same formula so the economy stays coherent.

### 2.2 Domain: Cybersecurity / Pentesting (`cyber`)

HTB boxes pay out **per phase**, matching your recon → enumeration → foothold → privesc → loot workflow. Completing a box is simply having logged all phases — no separate completion event, no double-counting, and a box you abandon after foothold still paid you for the work you actually did.

**Box value by tier** (H × D from formula; N=1.25 assumes notes/writeup, which you already do):

| Tier | Est. hours | D | Box value |
|---|---|---|---|
| Easy | 2.5 | 1.0 | **75 XP** |
| Medium | 5 | 1.25 | **150 XP** |
| Hard | 8 | 1.5 | **300 XP** |
| Insane | 12+ | 1.75 | **500 XP** (deliberately capped below formula — Insane hour counts balloon unpredictably; see §9) |

**Phase split of box value:** enumeration 20% · foothold 30% · privilege escalation 30% · loot + writeup 20%.

| Activity | XP | Formula basis |
|---|---|---|
| HTB phase completed | tier value × phase % | above |
| New technique / CVE applied for the first time | **40** | 1.5h × 1.25 × 1.25(first-time) |
| Session notes committed to security-notes repo | **15** | 0.5h × 1.0 × 1.25(artifact) — *only when logged outside a box phase, since phase XP already includes notes* |
| Bug bounty report submitted (genuine, in-scope effort) | **75** | 3h × 1.25 |
| Bug bounty report accepted / triaged valid | **150** | outcome event; also the trigger for the "first accepted submission" badge, which you confirm manually |

### 2.3 Domain: Unity 2D Game Development (`gamedev`)

| Activity | XP | Formula basis |
|---|---|---|
| Feature shipped (works, committed) | **75** | 3h × 1.0 × 1.25 |
| Meaningful bug fixed (the SpringJoint2D kind, not a typo) | **25** | 1h × 1.25 |
| Focused dev/debug session (no shippable outcome required) | **20** | 1h × 1.0 |
| Playtest session with written notes | **15** | 0.75h × 1.0 |
| Devlog entry published | **40** | 1.5h × 1.0 × 1.25 |

### 2.4 Domain: Coursework (`course`)

Named generically so it survives the transition from probability & statistics to whatever comes next — the config carries a `currentSubject` label for display.

| Activity | XP | Formula basis |
|---|---|---|
| Study session | **20** | 1h × 1.0 |
| Problem set completed | **50** | 2h × 1.0 × 1.25 |
| Concept mastered (can explain and apply it cold) | **30** | 1h × 1.25 × 1.25 |
| Exam completed | **100** | 4h prep+sit × 1.25 |
| Exam met/beat your target score | **+50** | outcome bonus |

### 2.5 Domain: Calisthenics (`physical`)

This domain has one deliberate rule: **volume earns zero XP.** Sessions earn XP; reps, sets, and tonnage are logged as *data attached to the session event* for your own tracking, but they never convert to points. This is the structural answer to the overtraining concern — there is nothing to grind. It also makes the logging regimen-agnostic: a session event carries a freeform `details` payload (exercises, sets×reps, holds, notes), so when your program changes, the schema doesn't.

| Activity | XP | Formula basis |
|---|---|---|
| Training session (any program, any structure) | **30** | 0.75h × 20 × 2.0 physical-intensity — the second deliberate formula deviation: intensity multiplier reflects that an hour of hard training is not an hour of routine desk work |
| Skill practice session (handstand work, planche progressions) | **30** | same; it's a session type flag, not a separate economy |
| Mobility / recovery session | **15** | 0.5h, lighter — recovery is rewarded, not just output |

Training XP is additionally capped at **5 XP-earning sessions per rolling 7 days** (config value). Sessions beyond that still log fully as data — they just pay 0 XP. The system never pays you to skip recovery.

---

## 3. Overall leveling formula

**Curve type: linearly increasing per-level cost (quadratic cumulative).** Steeper than linear so levels stay meaningful, gentler than exponential so year-three levels remain reachable in weeks, not quarters.

```
cost(L → L+1) = 100 + 50 × (L − 1)
totalXP(L)    = Σ  — see milestone table
```

Calibrated against a realistic 300–450 XP week:

| Level | Cumulative XP | ETA at ~375 XP/week |
|---|---|---|
| 2 | 100 | ~2 days |
| 5 | 700 | ~2 weeks |
| 10 | 2,700 | ~7 weeks |
| 20 | 10,450 | ~6–7 months |
| 30 | 23,200 | ~14 months |
| 40 | 40,950 | ~2.2 years |
| 50 | 63,700 | ~3.5 years |

Level 50 lands roughly on your 4-year cert horizon (CEH/OSCP era) — the number layer and the badge layer end up rhyming without being coupled. Curve constants (`base: 100`, `increment: 50`) live in config.

---

## 4. Consistency mechanics (no guilt, no grind)

Two mechanics, both windowed rather than streak-based. Neither can ever be "lost" — a missed day just slides out of the window.

**Momentum (rolling 7-day multiplier).** Count *distinct calendar days with any log* in the trailing 7 days, computed at the moment you log:

| Active days in trailing 7 | Multiplier on new XP |
|---|---|
| 0–2 | ×1.00 |
| 3–4 | ×1.05 |
| 5–7 | ×1.10 (cap) |

The cap at ×1.10 is intentional: logging all 7 days pays no more than 5 — there is no incentive to manufacture a log on a rest day. The UI shows "4 active days this week," never "streak: 47 🔥". Nothing resets.

**Breadth bonus (+10 XP).** The first log in each domain in a calendar week earns a flat +10. Four domains → at most +40/week. It nudges you back toward a neglected domain without punishing focus weeks.

Multiplier and bonus are stored *on the event at log time* (as `momentumMult` and `bonusXP` fields), so recomputation from the log is deterministic and doesn't depend on replaying window math.

---

## 5. Diminishing returns / soft daily caps

Two layers, both config-driven:

**Per-activity daily count tiers.** Each activity type defines how many times per day it pays full, half, then nothing:

| Pattern | Full | 50% | Then | Applied to |
|---|---|---|---|---|
| Repeatable-cheap | 2 | 2 | 0% | study session, dev session, session notes, playtest |
| Repeatable-moderate | 2 | 1 | 0% | bug fix, concept mastered, mobility |
| Naturally rare | ∞ | — | — | box phases, features shipped, problem sets, exams, bounty events, training sessions (governed by the weekly 5-session rule instead) |

**Per-domain daily ceiling (backstop):** first 250 XP per domain per day pays full; everything past that pays 10%. This exists purely so no future config edit or edge case makes a single day farmable. On an honest big day (finishing a Hard box), you'll notice the ceiling — that's acceptable; the box's remaining value effectively rolls into the satisfaction of logging it, and §9 names this tension.

---

## 6. Prestige badge system

**Mechanics:** badges are records, not logic. There is no code path that awards a badge. Status changes happen only through an explicit UI action ("Mark achieved") which stamps the date. XP and levels never touch this collection, and — one deliberate design choice — **badges grant no XP.** The moment a badge pays XP, the layers couple and the level starts to feel like the point. The badge case is the trophy shelf; the level is the heartbeat. (Tension acknowledged in §9.)

**Badge record schema:**

```json
{
  "id": "cert-oscp",
  "name": "OSCP",
  "domain": "cyber",
  "category": "certification",      // certification | career | physical | custom
  "status": "planned",              // planned | in_progress | achieved
  "targetWindow": "4y",             // optional: "now" | "1y" | "4y" | null
  "dateAchieved": null,             // ISO date, set only on confirm
  "notes": "",
  "sortOrder": 60,
  "updatedAt": "2026-07-10T00:00:00Z"
}
```

**Seeded starter list** (all editable; placeholders are real records with editable names, so "adding a badge later" = appending one JSON object — no logic changes):

| id | Name | Category | Status | Window |
|---|---|---|---|---|
| cert-google-cyber | Google Cybersecurity Certificate | certification | in_progress | now |
| cert-sql | SQL Certification | certification | in_progress | now |
| cert-anthropic-aff | Anthropic AI Fluency Foundations | certification | in_progress | now |
| cert-secplus | CompTIA Security+ | certification | planned | 1y |
| cert-cissp | CISSP | certification | planned | 1y |
| cert-slot-entry-1/2 | (placeholder — entry-level cert) | certification | planned | 1y |
| cert-ceh | CEH | certification | planned | 4y |
| cert-oscp | OSCP | certification | planned | 4y |
| cert-slot-adv-1/2 | (placeholder — advanced cert) | certification | planned | 4y |
| career-first-bounty-accepted | First accepted bug bounty | career | planned | — |
| career-first-payout | First bug bounty payout | career | planned | — |
| career-recurring-bounty | Bounty as reliable income | career | planned | — |
| career-first-pentest-role | First pentesting role | career | planned | — |
| career-first-con | First hacking convention | career | planned | — |
| phys-handstand | Freestanding handstand | physical | planned | — |
| phys-planche | Planche | physical | planned | — |
| phys-pistol | Pistol squat | physical | planned | — |
| phys-muscleup | Muscle-up | physical | planned | — |
| phys-slot-1/2 | (placeholder — physical skill) | physical | planned | — |

---

## 7. Data architecture

### 7.1 Config object (the single editable section)

Lives in `config.js` as one exported object — the only file you touch for rebalancing:

```js
export const CONFIG = {
  schemaVersion: 1,
  leveling: { base: 100, increment: 50 },
  momentum: { tiers: [[3, 1.05], [5, 1.10]] },
  breadthBonusXP: 10,
  domainDailyCeiling: { full: 250, overflowRate: 0.10 },
  domains: [
    { id: "cyber",    label: "Cybersecurity", color: "var(--orange)" },
    { id: "gamedev",  label: "Game Dev",      color: "var(--red)" },
    { id: "course",   label: "Coursework",    color: "var(--grey-1)",
      currentSubject: "Probability & Statistics" },
    { id: "physical", label: "Calisthenics",  color: "var(--grey-2)",
      weeklyXpSessionCap: 5 }
  ],
  activities: [
    { id: "htb-phase", domainId: "cyber", label: "HTB box phase",
      xp: "computed",                    // uses tiers + phaseSplit below
      tiers: { easy: 75, medium: 150, hard: 300, insane: 500 },
      phaseSplit: { enumeration: 0.20, foothold: 0.30,
                    privesc: 0.30, loot: 0.20 } },
    { id: "new-technique", domainId: "cyber", label: "New technique/CVE",
      xp: 40, dailyTiers: { full: 2, half: 1 } },
    { id: "session-notes", domainId: "cyber", label: "Session notes",
      xp: 15, dailyTiers: { full: 2, half: 2 } },
    // ... every activity from §2, same shape ...
    { id: "training-session", domainId: "physical", label: "Training session",
      xp: 30, dailyTiers: null, detailsSchema: "freeform" }
  ],
  seedBadges: [ /* §6 table as records */ ]
};
```

### 7.2 Event record (append-only log)

```json
{
  "id": "c7b9f1e2-...",             // crypto.randomUUID()
  "ts": "2026-07-10T21:14:00-05:00",
  "activityId": "htb-phase",
  "domainId": "cyber",
  "params": { "tier": "hard", "phase": "foothold", "box": "Vintage" },
  "details": "chained CVE-2025-XXXXX after cred spray",
  "baseXP": 90,
  "momentumMult": 1.05,
  "bonusXP": 0,
  "capAdjustedXP": 94                // final XP after tiers/ceiling, frozen
}
```

Final XP is frozen on the event so history is stable even if config changes later; a "recompute all" maintenance action exists for deliberate rebalances (§9).

### 7.3 localStorage schema

| Key | Contents |
|---|---|
| `crpg:v1:events` | JSON array of event records (append-only) |
| `crpg:v1:badges` | JSON array of badge records |
| `crpg:v1:meta` | `{ lastSyncedAt, lastExportHash, deviceId }` |

Config is *not* stored in localStorage — it ships with the site from the repo, so a config edit deploys everywhere on next page load.

### 7.4 Export file (`data/progression-data.json`, committed to the repo)

```json
{
  "schemaVersion": 1,
  "exportedAt": "2026-07-10T21:30:00-05:00",
  "deviceId": "desktop-win11",
  "events": [ /* full array */ ],
  "badges": [ /* full array */ ]
}
```

"Export & Sync" serializes both collections, downloads the file, and records `lastSyncedAt` + a hash in meta. You commit it like any other repo file.

### 7.5 Consistency and conflict resolution

On page load the app fetches `./data/progression-data.json` (same-origin — works on GitHub Pages with no CORS issues) and reconciles:

1. **Events:** union by `id`. Append-only + UUIDs means merging is order-independent and idempotent. A box logged on your laptop and a training session logged on your desktop simply both exist afterward.
2. **Badges:** last-write-wins per record by `updatedAt`. Badge edits are rare and single-user; LWW at per-record granularity is sufficient.
3. **Divergence detection:** if localStorage contains events absent from the fetched file *and* vice versa, the UI shows a one-line diff summary ("Local has 4 events not in repo; repo has 2 not in local — merge?") and merges on confirm. It never silently discards either side.
4. **Fresh device:** empty localStorage + repo file present → import wholesale. The committed file is the recovery path for cleared browser storage, which is localStorage's known fragility (§9).

The invariant that makes all of this safe: **derived state is never merged, only recomputed.**

---

## 8. Dashboard UI concept

**Recommendation on branding: align with the Spartan identity.** This is the flagship personal artifact tying together every other repo the shield already fronts — a neutral theme would orphan it. Better yet, the brand geometry maps naturally onto the mechanics: the **phalanx ring becomes the level-progress ring**. XP toward next level fills the ring segment by segment, like shields locking into formation. That's not decoration; it's the brand doing mechanical work.

**Visual language:** dark charcoal ground (`#1a1a1d`-family), burnt orange (`#c4501b`-family) as the XP/energy color, deep red for the cyber domain, two greys for supporting domains, and a monospace terminal face for numerals — echoing the "4096" plaque on your profile shield. Achieved badges render in lit bronze/orange; in-progress badges as orange outlines; planned badges dim grey. Confirming a badge is the one moment that earns a real animation (ring flare) — scarcity keeps it meaningful.

**Layout (single page, three zones):**

```
┌──────────────────────────────────────────────────────┐
│  HEADER — character sheet                            │
│  [Phalanx level ring: LV 14, 62% to 15]              │
│  Total XP · Momentum: "5 active days this week"      │
├────────────────────────────┬─────────────────────────┤
│  DOMAIN PANELS (2×2)       │  QUICK LOG              │
│  Each: name, this-week XP, │  domain → activity →    │
│  last activity, small      │  params → details →     │
│  4-week sparkline          │  [Log it]  (2 taps for  │
│                            │  common actions)        │
├────────────────────────────┴─────────────────────────┤
│  BADGE CASE — grouped: Certifications / Career /     │
│  Physical.  Grid of shield-shaped slots.             │
│  [Export & Sync] button + last-synced timestamp      │
└──────────────────────────────────────────────────────┘
```

The quick-log form is the make-or-break surface: if logging a training session takes more than ~5 seconds, the system dies of friction within a month. Most-used activities should be one tap from the top level.

---

## 9. Explicit tradeoffs

**Curve steepness vs. long-term motivation.** The quadratic curve means level 3 takes days and level 43 takes a month. That's correct for prestige but risks the "dead zone" feeling around month 8. Mitigation: the momentum indicator and weekly XP sparkline give sub-level feedback loops. If levels still feel too slow around L25, lower `increment` in config and recompute — the event-sourced design makes rebalancing a one-line change plus a recompute, but note the flip side: **recomputing rewrites your historical level timeline.** Frozen per-event XP is the default precisely so casual config tweaks don't silently rewrite the past; rebalancing is a deliberate, run-the-maintenance-action decision.

**One unified level hides domain neglect.** You can hit level 30 having not opened Unity in a year. That's partially intentional (life has seasons), but the 2×2 domain panels and breadth bonus exist as the counterweight. If neglect bothers you later, a per-domain "rank" display can be derived from the same events without touching the economy.

**Badges paying zero XP keeps the layers clean but risks anticlimax.** Passing OSCP and watching your level not move is philosophically pure and emotionally weird. The ring-flare animation is the compensation. If it still feels hollow, the least-damaging compromise is a *fixed ceremonial amount* (e.g., flat 100 XP regardless of badge) — flat, so badge XP never becomes something to optimize.

**Self-reported difficulty is gameable by definition.** "Meaningful bug fix" vs. typo, "concept mastered" vs. skimmed — only you enforce these. The count tiers and ceilings limit the damage of self-deception but can't eliminate it. The system's honest framing: it's a mirror, not a judge.

**The domain daily ceiling clips legitimate big days.** Finish a Hard box's last two phases plus notes in one sitting and you'll brush the 250 cap. Accepting occasional clipping is better than leaving a farmable hole; if it stings in practice, raise `domainDailyCeiling.full` — it's one config value.

**Insane boxes are underpriced by the formula on purpose.** A 20-hour Insane box "deserves" ~875 XP by strict formula, but hour-counts at that tier are dominated by being stuck, and paying for stuck-time teaches the wrong lesson. 500 is a floor-of-respect, not a wage.

**localStorage is fragile; the export is manual.** A cleared browser between syncs loses unsynced events. The mitigations: the last-synced timestamp is permanently visible next to the export button, and the app can surface a gentle "12 events unsynced" note after a threshold. A stretch item (auto-download on interval) exists if manual discipline proves insufficient.

---

## 10. Phased build roadmap

### MVP — first working version

1. `config.js` with the full §7.1 object (all four domains, all activities, seeded badges).
2. Event logging: quick-log form → validated event → localStorage append. Includes momentum multiplier, breadth bonus, and count-tier/ceiling math at log time.
3. Derived-state engine: pure function `(events, config) → {level, xpIntoLevel, perDomainWeekXP, momentum}`.
4. Header with level + plain progress bar (ring art comes later), momentum line, per-domain week totals as plain numbers.
5. Badge case: grouped list, status toggle with date stamp, LWW `updatedAt`.
6. Export & Sync: download `progression-data.json`, stamp meta.
7. Load-time reconcile: fetch repo file, union events, LWW badges, diff-summary confirm.

That's a complete, durable system — everything after this is polish.

### Stretch — later passes

- Phalanx-ring level visualization and full Spartan theming pass (use the frontend-design conventions from your other repos).
- Badge-confirm animation.
- 4-week sparklines per domain; a history/timeline view with event editing (edit = tombstone old event + append corrected one, preserving append-only semantics).
- Unsynced-events nag threshold; optional periodic auto-download.
- "Recompute all" maintenance action for deliberate rebalances.
- Per-domain derived ranks.
- Import-file picker (manual merge from a downloaded JSON without going through the repo).
- PWA manifest for phone-homescreen logging (biggest friction-killer for the training domain).
- Optional: bounty-income tracker feeding the "recurring income" badge decision with actual numbers.

---

*End of spec. Hand this file to an implementation session as-is; §7.1 is the contract the code must honor.*
