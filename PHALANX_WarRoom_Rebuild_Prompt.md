# PHALANX — War Room Rebuild — Build Brief for Claude

You're rebuilding a personal progression tracker called PHALANX. This document is self-contained — you don't have prior conversation history or project memory, so everything you need is below. Read the whole thing before writing any code.

## What this project is

PHALANX is a single self-contained `index.html` file — no build tools, no frameworks, no external JS dependencies (Google Fonts is the only external resource, already in use). It's a personal career RPG that turns real-world effort across four life domains into character progression: Cybersecurity, Game Dev, Coursework, Calisthenics. Data lives in browser localStorage with manual export/import to a `phalanx-data.json` file committed to the repo. The system is designed to outlast any single AI model or platform, which is exactly why it has zero dependencies.

You're not starting from a blank page. Below is the **exact current live file** — treat it as ground truth for data, formulas, and visual identity. The task is to rebuild it into a "war room" — the same character progression system, restyled and re-architected around a warfare metaphor, with genuinely new game systems layered on top.

## What must be preserved exactly — non-negotiable

These are locked. Do not redesign, rename, retune, or "improve" them — copy them forward as-is:

- The XP formula, leveling curve (`250 * (L-1)^1.6`), momentum bonus tiers (3-4 active days in trailing 7 = +10%, 5+ = +20%), and soft daily cap / 50% overflow logic — all exactly as implemented in the baseline file below.
- Every domain, every activity, every XP value, every daily cap — exactly as in the `CONFIG` object below.
- Every badge (certifications, career milestones, physical unlocks), their groupings, and the planned → in-progress → achieved manual-confirm cycle. Badges are still **never auto-awarded** — that rule extends to the new boss layer too (see below).
- The color palette, font stack, and general visual restraint (flat panels, thin borders, no gradients except the one subtle radial background wash already present). Same burnt orange / spartan red / iron dark / bone palette, same Chakra Petch / JetBrains Mono / Inter typography, same Λ (lambda) symbol as the core icon.
- localStorage-first persistence with export/import to `phalanx-data.json`, confirm-dialog on import showing both timestamps (never silent overwrite).
- The existing state schema's core shape (`version`, `lastModified`, `log[]`, `badges{}`) — extend it, don't replace it. Old save files must still load without error (add new fields with sensible defaults if missing).

## What's being added — the war room layer

Two new progression tracks sit **alongside** (never replacing) the existing Level/XP and Prestige Badges, which stay exactly as-is and remain the "sacred," never-punished core.

### 1. Army Size (permanent) and Army Readiness (volatile) — two separate numbers

- **Army Size**: permanent, only ever grows. Funded by spending domain currency on Reinforcements. This is the long-game "months of real work" bragging-rights number, same spirit as Prestige Badges.
- **Army Readiness**: volatile, recomputed fresh every time the page loads (not a live ticking interval — see Refresh Behavior below). It is a **pure reflection of the existing momentum calculation** — reuse the same trailing-7-day active-days logic already in the code, remapped to a 0–100% readiness display. Do not build a separate independent decay clock. This keeps readiness consistent with the "no streak penalties, rest days are neutral" philosophy the whole system is built on — it dips only because momentum genuinely dipped, never as an extra punishment layered on top.

### 2. Four unit types, one per domain, each independently upgradeable

- Cybersecurity → **Intel** troops · Coursework → **Supply** troops · Calisthenics → **Muscle** troops · Game Dev → **Engineering** troops (Materiel).
- Each domain earns its own currency alongside XP (Intel, Supply, Muscle, Materiel) at some reasonable conversion rate from that domain's XP — pick a sensible rate and put it in CONFIG, tunable later. This currency is **spendable** and separate from the untouchable XP total.
- Each domain's currency funds exactly two things for its own troop type, independently of the other three domains:
  - **Reinforcements** — grows that domain's permanent Army Size contribution (troop count).
  - **Upgrades** — advances that domain's troop quality tier: Recruit → Veteran → Elite. Higher tiers hit harder in the push-line calculation (see below) and should modestly improve how gracefully that domain's readiness responds to a quiet week (mechanically: reuse the readiness reflection above, just let higher-tier troops soften its visual impact slightly — don't build a second decay system to do this).
- **Research** is the one cross-cutting tree. Nodes cost whichever currency(ies) thematically fit the unlock (a recon-flavored node costs Intel, a fortification node costs Materiel, etc). Suggested nodes: faster/cheaper fog-of-war reveals, a passive readiness-decay softener for genuinely busy weeks, siege-flavored options relevant to boss fights, and **one alternate formation** (see below). Some Research nodes should be gated behind real Prestige badges being marked achieved — real-world achievement unlocking new in-game mechanics is an intentional design goal, not an edge case to avoid.

### 3. Weekly skirmishes

- A Monday-start calendar week (separate from, and not to be confused with, the existing 7-day *trailing* momentum window, which is unaffected).
- **Hybrid enemy selection**: the system auto-suggests a weekly enemy/campaign flavor based on recent logged activity (e.g., weighting toward whichever domain has been comparatively neglected, since that's the one worth a recon push), the player confirms or can pick something else. Simple heuristic is fine — this doesn't need to be clever, just plausible.
- Every weekly enemy has a **hidden exploit vector** — a specific domain they're built to punish. It's revealed via **fog of war**: spending Intel (recon) burns away the fog and reveals which domain they're targeting. Until revealed, the player doesn't know. If the targeted domain's readiness is under some reasonable threshold when the push line resolves, the enemy gets a force bonus against the player specifically.
- **Live tug-of-war push line**: a horizontal bar/line whose contact point shifts based on the player's total campaign-weighted force (derived from Army Size, Upgrades tier, and current Readiness across domains, weighted toward whatever the confirmed campaign is emphasizing that week) versus the enemy's force (a base value that scales gently with the player's overall progress so skirmishes stay a fair, close match even at high levels — suggest deriving enemy base force as roughly 80–110% of the player's current total force depending on a difficulty flavor per enemy — plus the exploit bonus if applicable). This recomputes fresh every time the dashboard is opened, not on a running timer.
- **Stakes are asymmetric on purpose**: winning banks bonus Command Points (split across currencies, or weighted toward the week's focus — your call). Losing is **neutral — no downside, no readiness hit, nothing rolls back.** This is a deliberate design choice matching the existing "no punishment" philosophy; do not add a loss penalty here even if it seems like it would add stakes. The stakes belong entirely to the boss tier below.

### 4. Boss tier — real stakes, tied to real milestones

- Reuse the existing Prestige Badge system directly as the boss trigger — **do not build a separate boss state object.** When a certification badge (or suitable career-milestone badge) is marked `in-progress`, that boss encounter becomes active. When the badge is marked `achieved` (through the existing manual-confirm dialog — unchanged), the boss is resolved as defeated. No auto-awarding, same as today.
- Bosses carry **genuine risk**, unlike weekly skirmishes: if the player's Army Size/Readiness/Upgrades going into the fight are under-prepared relative to the boss's threshold, the fight can be lost. A loss does **not** touch core XP, Level, or Army Size — those stay untouched no matter what. A loss should mean a "regroup": Readiness takes a real (but bounded, not devastating) dip, and there's some reasonable cooldown or resource cost before re-attempting. How decisively a boss is won (once won) should scale with how prepared the player actually was — a narrow win and a crushing win should look and feel different, even though both count as a win.
- This tier should read as rare and significant — these are the OSCP/CISSP/CEH/Security+-scale fights, not weekly content.

### 5. Formation, visually

- A **live tug-of-war push line** rendered as small flat shapes (simple geometric troop icons — think tiny triangles/rectangles suggesting spears/shields, not detailed sprites) in two facing ranks, with the contact point positioned based on the push-line math above.
- **One alternate formation**, unlockable via Research: default is a defense-leaning **Wall** (matches "hold the line"), unlockable alternative is a more aggressive **Wedge**. Keep this simple — a formation is a named modifier on the push-line math (e.g. Wall dampens swings in both directions, Wedge amplifies them), not a whole tactical subsystem.
- Formation, front-line depth, and readiness should all be visually legible from the same panel at a glance — this is the centerpiece visual of the War Room tab.

## Current baseline — the exact live file (rebuild from this)

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>PHALANX — Progression</title>
<!--
  PHALANX PROGRESSION SYSTEM
  ==========================
  Design decisions (edit CONFIG below to change any of this — logic never needs touching):

  XP anchor: 1 XP ~ 1 minute of focused baseline effort. Discrete achievements are
  priced as (typical time) x (difficulty/novelty multiplier).

  Leveling curve: cumulative XP to reach level L = 250 * (L-1)^1.6
  Long-game shape: ~level 10 in a few months of steady work, level 50 on an
  OSCP-scale multi-year timeline. No level cap.

  Momentum (consistency without guilt): count of active days in the trailing 7.
  3-4 active days = +10% XP on new logs, 5+ = +20%. Rest days are neutral —
  nothing ever resets or punishes.

  Soft daily caps: per-domain daily XP past the cap earns at 50%. Caps sized so a
  legitimately big day (a Hard box, a long study block) fits; grinding trivial
  entries doesn't scale.

  Badges: manual-only. Never auto-awarded. status: planned | in-progress | achieved.

  Data: saved to localStorage on every change (key below). Export downloads
  phalanx-data.json for committing to the repo; Import restores from that file
  (confirm dialog shows both timestamps — newest wins is YOUR call, not automatic).
-->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Chakra+Petch:wght@500;700&family=JetBrains+Mono:wght@400;700&family=Inter:wght@400;600&display=swap" rel="stylesheet">
<style>
:root{
  --iron:#1C1E22;        /* page background */
  --steel:#26292F;       /* panels */
  --rivet:#33373F;       /* borders, dividers */
  --bronze:#C4682B;      /* burnt orange — primary accent */
  --blood:#8E2A22;       /* spartan red — secondary accent */
  --bone:#E9E5DB;        /* headings, strong text */
  --ash:#A9ADB5;         /* body text */
  --moss:#6F8F5E;        /* achieved-state green, muted */
  --ring-track:#3A3E46;
}
*{box-sizing:border-box;margin:0;padding:0}
html{background:var(--iron)}
body{
  font-family:'Inter',system-ui,sans-serif;color:var(--ash);
  background:
    radial-gradient(ellipse 80% 50% at 50% -10%, rgba(196,104,43,.07), transparent),
    var(--iron);
  min-height:100vh;padding:24px 16px 80px;
}
.wrap{max-width:1060px;margin:0 auto}
h1,h2,h3{font-family:'Chakra Petch',sans-serif;color:var(--bone);letter-spacing:.04em}
.mono{font-family:'JetBrains Mono',monospace}

/* ---------- header: the shield ring ---------- */
header{display:flex;align-items:center;gap:28px;flex-wrap:wrap;margin-bottom:36px}
.shield{position:relative;width:170px;height:170px;flex:none}
.shield svg{width:100%;height:100%;transform:rotate(-90deg)}
.shield .lambda{
  position:absolute;inset:0;display:flex;align-items:center;justify-content:center;
  font-family:'Chakra Petch',sans-serif;font-weight:700;font-size:56px;color:var(--bronze);
  text-shadow:0 0 24px rgba(196,104,43,.35);
}
.shield .lvl-tag{
  position:absolute;left:50%;bottom:6px;transform:translateX(-50%);
  background:var(--blood);color:var(--bone);font-family:'JetBrains Mono',monospace;
  font-size:12px;font-weight:700;padding:2px 10px;border-radius:2px;letter-spacing:.08em;
}
.head-info h1{font-size:30px;text-transform:uppercase}
.head-info .sub{font-size:13px;letter-spacing:.14em;text-transform:uppercase;color:var(--bronze);margin-top:2px}
.head-stats{display:flex;gap:26px;margin-top:14px;flex-wrap:wrap}
.head-stats .stat b{display:block;font-family:'JetBrains Mono',monospace;font-size:20px;color:var(--bone)}
.head-stats .stat span{font-size:11px;text-transform:uppercase;letter-spacing:.1em}
.momentum-pips{display:flex;gap:5px;margin-top:5px}
.momentum-pips i{width:14px;height:6px;background:var(--rivet);display:block}
.momentum-pips i.on{background:var(--bronze)}

/* ---------- layout ---------- */
.grid{display:grid;grid-template-columns:minmax(300px,380px) 1fr;gap:20px}
@media(max-width:820px){.grid{grid-template-columns:1fr}}
.panel{background:var(--steel);border:1px solid var(--rivet);border-radius:6px;padding:20px}
.panel h2{font-size:14px;text-transform:uppercase;letter-spacing:.14em;margin-bottom:16px;
  padding-bottom:10px;border-bottom:1px solid var(--rivet)}
.panel h2 .accent{color:var(--bronze)}

/* ---------- log form ---------- */
label{display:block;font-size:11px;text-transform:uppercase;letter-spacing:.1em;margin:14px 0 6px}
select,input[type=text],input[type=date],textarea{
  width:100%;background:var(--iron);border:1px solid var(--rivet);color:var(--bone);
  padding:9px 10px;border-radius:4px;font-family:'JetBrains Mono',monospace;font-size:13px;
}
select:focus,input:focus,textarea:focus{outline:2px solid var(--bronze);outline-offset:-1px}
.xp-preview{margin-top:14px;font-family:'JetBrains Mono',monospace;font-size:13px;color:var(--bronze)}
.xp-preview small{color:var(--ash);display:block;margin-top:3px;font-size:11px}
button{
  font-family:'Chakra Petch',sans-serif;font-weight:700;letter-spacing:.08em;text-transform:uppercase;
  border:none;border-radius:4px;cursor:pointer;font-size:13px;padding:11px 18px;
}
button:focus-visible{outline:2px solid var(--bone);outline-offset:2px}
.btn-primary{background:var(--bronze);color:#1a130c;width:100%;margin-top:16px}
.btn-primary:hover{background:#d5793a}
.btn-ghost{background:transparent;color:var(--ash);border:1px solid var(--rivet);font-size:11px;padding:8px 12px}
.btn-ghost:hover{border-color:var(--bronze);color:var(--bronze)}

/* ---------- domain bars ---------- */
.domain{margin-bottom:16px}
.domain .row{display:flex;justify-content:space-between;font-size:12px;margin-bottom:5px}
.domain .row b{color:var(--bone);font-family:'Chakra Petch',sans-serif;letter-spacing:.06em}
.domain .row span{font-family:'JetBrains Mono',monospace}
.bar{height:8px;background:var(--iron);border-radius:2px;overflow:hidden}
.bar i{display:block;height:100%;background:linear-gradient(90deg,var(--blood),var(--bronze));transition:width .5s ease}

/* ---------- badge armory ---------- */
.armory{margin-top:20px}
.badge-group{margin-bottom:18px}
.badge-group h3{font-size:12px;letter-spacing:.12em;text-transform:uppercase;color:var(--ash);margin-bottom:10px}
.badges{display:grid;grid-template-columns:repeat(auto-fill,minmax(190px,1fr));gap:10px}
.badge{
  border:1px solid var(--rivet);border-radius:4px;padding:10px 12px;cursor:pointer;
  background:var(--iron);transition:border-color .15s;
}
.badge:hover{border-color:var(--bronze)}
.badge .name{font-size:13px;color:var(--bone);font-weight:600}
.badge .meta{font-family:'JetBrains Mono',monospace;font-size:10px;margin-top:5px;letter-spacing:.06em;text-transform:uppercase}
.badge.planned .meta{color:var(--ash)}
.badge.in-progress{border-color:var(--bronze)}
.badge.in-progress .meta{color:var(--bronze)}
.badge.achieved{border-color:var(--moss);background:rgba(111,143,94,.08)}
.badge.achieved .meta{color:var(--moss)}
.badge.achieved .name::before{content:"Λ ";color:var(--moss)}

/* ---------- log history ---------- */
.log-list{list-style:none;max-height:300px;overflow-y:auto}
.log-list li{
  display:flex;justify-content:space-between;gap:10px;padding:8px 2px;font-size:12px;
  border-bottom:1px solid var(--rivet);
}
.log-list li .what{color:var(--bone)}
.log-list li .when{font-family:'JetBrains Mono',monospace;color:var(--ash);white-space:nowrap}
.log-list li .xp{font-family:'JetBrains Mono',monospace;color:var(--bronze);white-space:nowrap}
.log-list li button{padding:0 6px;font-size:11px;background:none;color:var(--blood)}

/* ---------- data bar ---------- */
.data-bar{display:flex;gap:10px;align-items:center;margin-top:20px;flex-wrap:wrap}
.data-bar .note{font-size:11px;font-family:'JetBrains Mono',monospace}
.toast{
  position:fixed;left:50%;bottom:26px;transform:translateX(-50%);
  background:var(--bronze);color:#1a130c;font-family:'Chakra Petch',sans-serif;font-weight:700;
  padding:12px 22px;border-radius:4px;letter-spacing:.06em;display:none;z-index:50;
}
.levelup{animation:pulse 1.2s ease 2}
@keyframes pulse{0%,100%{filter:none}50%{filter:drop-shadow(0 0 18px rgba(196,104,43,.9))}}
@media (prefers-reduced-motion: reduce){ .levelup{animation:none} .bar i{transition:none} }
</style>
</head>
<body>
<div class="wrap">

<header>
  <div class="shield" id="shield">
    <svg viewBox="0 0 120 120" aria-hidden="true">
      <circle cx="60" cy="60" r="52" fill="none" stroke="var(--ring-track)" stroke-width="7"/>
      <circle id="ring" cx="60" cy="60" r="52" fill="none" stroke="var(--bronze)" stroke-width="7"
        stroke-linecap="butt" stroke-dasharray="326.7" stroke-dashoffset="326.7"/>
      <g id="ticks" stroke="var(--iron)" stroke-width="2"></g>
    </svg>
    <div class="lambda">Λ</div>
    <div class="lvl-tag" id="lvlTag">LVL 1</div>
  </div>
  <div class="head-info">
    <h1>Phalanx</h1>
    <div class="sub">Hold the line</div>
    <div class="head-stats">
      <div class="stat"><b id="totalXp" class="mono">0</b><span>Total XP</span></div>
      <div class="stat"><b id="toNext" class="mono">—</b><span>XP to next level</span></div>
      <div class="stat">
        <b id="momentumPct" class="mono">+0%</b><span>Momentum (7-day)</span>
        <div class="momentum-pips" id="pips" title="Active days in the last 7"></div>
      </div>
    </div>
  </div>
</header>

<div class="grid">
  <div class="panel">
    <h2>Log <span class="accent">//</span> Entry</h2>
    <label for="fDomain">Domain</label>
    <select id="fDomain"></select>
    <label for="fActivity">Activity</label>
    <select id="fActivity"></select>
    <label for="fNote">Note (optional)</label>
    <input type="text" id="fNote" placeholder="e.g. HTB: Keeper — rooted" maxlength="120">
    <label for="fDate">Date</label>
    <input type="date" id="fDate">
    <div class="xp-preview" id="xpPreview"></div>
    <button class="btn-primary" id="logBtn">Log it</button>
  </div>

  <div>
    <div class="panel">
      <h2>Domains <span class="accent">//</span> Today &amp; All-time</h2>
      <div id="domains"></div>
    </div>
    <div class="panel" style="margin-top:20px">
      <h2>Recent <span class="accent">//</span> Log</h2>
      <ul class="log-list" id="logList"></ul>
    </div>
  </div>
</div>

<div class="panel armory">
  <h2>The Armory <span class="accent">//</span> Prestige Badges <span style="float:right;font-size:10px;color:var(--ash);letter-spacing:.06em">click a badge to cycle: planned → in progress → achieved</span></h2>
  <div id="armory"></div>
</div>

<div class="data-bar">
  <button class="btn-ghost" id="exportBtn">Export phalanx-data.json</button>
  <button class="btn-ghost" id="importBtn">Import from file</button>
  <input type="file" id="importFile" accept=".json" style="display:none">
  <span class="note" id="saveNote">saved locally</span>
</div>

</div>
<div class="toast" id="toast"></div>

<script>
/* =====================================================================
   CONFIG — everything tunable lives here. Edit freely; logic below
   reads from this and never hardcodes domains, values, or badges.
   ===================================================================== */
const CONFIG = {
  xpAnchor: "1 XP ~ 1 focused minute at baseline effort",
  levelCurve: { base: 250, exponent: 1.6 },   // cumulative XP to reach L = base*(L-1)^exp
  momentum: [                                  // active days in trailing 7 -> bonus
    { minDays: 5, bonus: 0.20 },
    { minDays: 3, bonus: 0.10 },
  ],
  softCapOverflowRate: 0.5,                    // XP past a domain's daily cap earns at this rate
  domains: [
    { id:"cyber", name:"Cybersecurity", dailyCap:400, activities:[
      { id:"htb-easy",    name:"HTB box — Easy (rooted)",   xp:60 },
      { id:"htb-med",     name:"HTB box — Medium (rooted)", xp:120 },
      { id:"htb-hard",    name:"HTB box — Hard (rooted)",   xp:240 },
      { id:"htb-insane",  name:"HTB box — Insane (rooted)", xp:400 },
      { id:"box-stage",   name:"Box stage cleared (recon/enum/foothold/privesc/loot)", xp:15 },
      { id:"new-tech",    name:"New technique or CVE applied (first time)", xp:40 },
      { id:"notes",       name:"Session notes written up",  xp:20 },
      { id:"bounty-sub",  name:"Bug bounty submission filed", xp:100 },
    ]},
    { id:"unity", name:"Game Dev", dailyCap:240, activities:[
      { id:"feature",  name:"Feature shipped",            xp:80 },
      { id:"bugfix",   name:"Meaningful bug fixed",       xp:40 },
      { id:"session",  name:"Focused dev session",        xp:30 },
      { id:"playtest", name:"Playtest run",               xp:20 },
      { id:"devlog",   name:"Devlog entry published",     xp:30 },
    ]},
    { id:"course", name:"Coursework", dailyCap:240, activities:[
      { id:"study",    name:"Study session",              xp:30 },
      { id:"pset",     name:"Problem set completed",      xp:50 },
      { id:"concept",  name:"New concept mastered",       xp:40 },
      { id:"exam",     name:"Exam taken",                 xp:100 },
    ]},
    { id:"body", name:"Calisthenics", dailyCap:120, activities:[
      { id:"train",    name:"Training session",           xp:40 },
      { id:"skill",    name:"Skill progression work",     xp:20 },
      { id:"volume",   name:"Extra volume logged",        xp:10 },
    ]},
  ],
  badges: [
    // --- certifications ---
    { id:"cert-google", group:"Certifications", name:"Google Cybersecurity (Coursera)" },
    { id:"cert-sql",    group:"Certifications", name:"SQL Certification" },
    { id:"cert-anthropic", group:"Certifications", name:"Anthropic AI Fluency Foundations" },
    { id:"cert-secplus", group:"Certifications", name:"CompTIA Security+" },
    { id:"cert-cissp",  group:"Certifications", name:"CISSP" },
    { id:"cert-ceh",    group:"Certifications", name:"CEH" },
    { id:"cert-oscp",   group:"Certifications", name:"OSCP" },
    // --- career milestones ---
    { id:"car-sub",    group:"Career", name:"First accepted bounty submission" },
    { id:"car-payout", group:"Career", name:"First paid bounty payout" },
    { id:"car-income", group:"Career", name:"Bounty income — recurring" },
    { id:"car-role",   group:"Career", name:"First pentesting role" },
    { id:"car-con",    group:"Career", name:"First hacking convention" },
    // --- physical unlocks ---
    { id:"phy-hs",     group:"Physical", name:"Freestanding handstand" },
    { id:"phy-planche",group:"Physical", name:"Planche" },
    { id:"phy-pistol", group:"Physical", name:"Pistol squat" },
    { id:"phy-mu",     group:"Physical", name:"Muscle-up" },
  ],
};

/* =====================================================================
   STATE + PERSISTENCE
   ===================================================================== */
const SAVE_KEY = "phalanx-save-v1";
const BADGE_STATUSES = ["planned","in-progress","achieved"];
let state = load();

function freshState(){
  return { version:1, lastModified:new Date().toISOString(), log:[], badges:{} };
}
function load(){
  try{
    const raw = localStorage.getItem(SAVE_KEY);
    if(raw){ return JSON.parse(raw); }
  }catch(e){ console.error("load failed", e); }
  return freshState();
}
function save(){
  state.lastModified = new Date().toISOString();
  try{
    localStorage.setItem(SAVE_KEY, JSON.stringify(state));
    document.getElementById("saveNote").textContent =
      "saved locally · " + new Date().toLocaleTimeString();
  }catch(e){
    document.getElementById("saveNote").textContent = "SAVE FAILED — export now";
  }
}

/* =====================================================================
   XP ENGINE
   ===================================================================== */
function activityById(domainId, actId){
  const d = CONFIG.domains.find(d=>d.id===domainId);
  return d ? d.activities.find(a=>a.id===actId) : null;
}
function domainXpOnDate(domainId, dateStr){
  return state.log.filter(e=>e.domain===domainId && e.date===dateStr)
    .reduce((s,e)=>s+e.awarded,0);
}
function activeDaysLast7(refDate){
  const days = new Set(state.log.map(e=>e.date));
  let count=0;
  for(let i=1;i<=7;i++){
    const d = new Date(refDate); d.setDate(d.getDate()-i);
    if(days.has(isoDate(d))) count++;
  }
  return count;
}
function momentumBonus(refDate){
  const n = activeDaysLast7(refDate);
  for(const tier of CONFIG.momentum){ if(n>=tier.minDays) return tier.bonus; }
  return 0;
}
function computeAward(domainId, actId, dateStr){
  const act = activityById(domainId, actId);
  if(!act) return null;
  const domain = CONFIG.domains.find(d=>d.id===domainId);
  const bonus = momentumBonus(new Date(dateStr+"T12:00:00"));
  let raw = Math.round(act.xp * (1+bonus));
  const already = domainXpOnDate(domainId, dateStr);
  let awarded = raw, capped = false;
  if(already >= domain.dailyCap){
    awarded = Math.round(raw * CONFIG.softCapOverflowRate); capped = true;
  } else if(already + raw > domain.dailyCap){
    const under = domain.dailyCap - already;
    awarded = under + Math.round((raw-under)*CONFIG.softCapOverflowRate); capped = true;
  }
  return { base:act.xp, bonus, awarded, capped };
}
function totalXp(){ return state.log.reduce((s,e)=>s+e.awarded,0); }
function xpForLevel(L){ return Math.round(CONFIG.levelCurve.base * Math.pow(L-1, CONFIG.levelCurve.exponent)); }
function levelFromXp(xp){
  let L=1; while(xpForLevel(L+1)<=xp) L++;
  return L;
}

/* =====================================================================
   UI
   ===================================================================== */
const $ = id => document.getElementById(id);
function isoDate(d){ return d.toISOString().slice(0,10); }

function initTicks(){
  const g = $("ticks");
  for(let i=0;i<24;i++){
    const a = i*15*Math.PI/180;
    const x1=60+48*Math.cos(a), y1=60+48*Math.sin(a);
    const x2=60+56*Math.cos(a), y2=60+56*Math.sin(a);
    const l = document.createElementNS("http://www.w3.org/2000/svg","line");
    l.setAttribute("x1",x1);l.setAttribute("y1",y1);l.setAttribute("x2",x2);l.setAttribute("y2",y2);
    g.appendChild(l);
  }
}
function renderHeader(){
  const xp = totalXp(), lvl = levelFromXp(xp);
  const cur = xpForLevel(lvl), next = xpForLevel(lvl+1);
  const frac = (xp-cur)/(next-cur);
  $("lvlTag").textContent = "LVL "+lvl;
  $("totalXp").textContent = xp.toLocaleString();
  $("toNext").textContent = (next-xp).toLocaleString();
  const C = 326.7;
  $("ring").style.strokeDashoffset = C*(1-Math.max(0,Math.min(1,frac)));
  const n = activeDaysLast7(new Date());
  $("momentumPct").textContent = "+"+Math.round(momentumBonus(new Date())*100)+"%";
  const pips = $("pips"); pips.innerHTML="";
  for(let i=0;i<7;i++){ const p=document.createElement("i"); if(i<n)p.classList.add("on"); pips.appendChild(p); }
}
function renderDomains(){
  const today = isoDate(new Date());
  const host = $("domains"); host.innerHTML="";
  for(const d of CONFIG.domains){
    const todayXp = domainXpOnDate(d.id, today);
    const allXp = state.log.filter(e=>e.domain===d.id).reduce((s,e)=>s+e.awarded,0);
    const pct = Math.min(100, todayXp/d.dailyCap*100);
    const el = document.createElement("div"); el.className="domain";
    el.innerHTML = `<div class="row"><b>${d.name}</b>
      <span>${todayXp} / ${d.dailyCap} today · ${allXp.toLocaleString()} all-time</span></div>
      <div class="bar"><i style="width:${pct}%"></i></div>`;
    host.appendChild(el);
  }
}
function renderLog(){
  const ul = $("logList"); ul.innerHTML="";
  const items = [...state.log].sort((a,b)=> b.ts.localeCompare(a.ts)).slice(0,50);
  if(!items.length){ ul.innerHTML = '<li><span class="what">Nothing logged yet. First entry starts the record.</span></li>'; return; }
  for(const e of items){
    const act = activityById(e.domain, e.activity);
    const li = document.createElement("li");
    li.innerHTML = `<span class="what">${act?act.name:e.activity}${e.note?" — "+escapeHtml(e.note):""}</span>
      <span class="when">${e.date}</span><span class="xp">+${e.awarded}</span>`;
    const del = document.createElement("button"); del.textContent="✕"; del.title="Delete entry";
    del.onclick = ()=>{ if(confirm("Delete this entry?")){ state.log = state.log.filter(x=>x.ts!==e.ts); save(); renderAll(); } };
    li.appendChild(del); ul.appendChild(li);
  }
}
function renderArmory(){
  const host = $("armory"); host.innerHTML="";
  const groups = [...new Set(CONFIG.badges.map(b=>b.group))];
  for(const g of groups){
    const wrap = document.createElement("div"); wrap.className="badge-group";
    wrap.innerHTML = `<h3>${g}</h3>`;
    const grid = document.createElement("div"); grid.className="badges";
    for(const b of CONFIG.badges.filter(b=>b.group===g)){
      const st = state.badges[b.id] || { status:"planned", date:null };
      const el = document.createElement("div");
      el.className = "badge "+st.status;
      el.setAttribute("role","button"); el.setAttribute("tabindex","0");
      el.innerHTML = `<div class="name">${b.name}</div>
        <div class="meta">${st.status.replace("-"," ")}${st.date? " · "+st.date : ""}</div>`;
      const cycle = ()=>{
        const idx = BADGE_STATUSES.indexOf(st.status);
        const next = BADGE_STATUSES[(idx+1)%BADGE_STATUSES.length];
        if(next==="achieved"){
          if(!confirm(`Mark "${b.name}" as ACTUALLY achieved? Only confirm if it's real.`)) return;
          state.badges[b.id] = { status:"achieved", date:isoDate(new Date()) };
          toast("Λ  "+b.name+" — earned");
        } else {
          state.badges[b.id] = { status:next, date:null };
        }
        save(); renderArmory();
      };
      el.onclick = cycle;
      el.onkeydown = ev=>{ if(ev.key==="Enter"||ev.key===" "){ ev.preventDefault(); cycle(); } };
      grid.appendChild(el);
    }
    wrap.appendChild(grid); host.appendChild(wrap);
  }
}
function renderForm(){
  const dSel = $("fDomain");
  if(!dSel.options.length){
    for(const d of CONFIG.domains){
      const o = document.createElement("option"); o.value=d.id; o.textContent=d.name; dSel.appendChild(o);
    }
    dSel.onchange = renderForm;
    $("fActivity").onchange = updatePreview;
    $("fDate").value = isoDate(new Date());
    $("fDate").onchange = updatePreview;
  }
  const d = CONFIG.domains.find(x=>x.id===dSel.value) || CONFIG.domains[0];
  const aSel = $("fActivity"); aSel.innerHTML="";
  for(const a of d.activities){
    const o = document.createElement("option"); o.value=a.id;
    o.textContent = `${a.name}  (+${a.xp})`; aSel.appendChild(o);
  }
  updatePreview();
}
function updatePreview(){
  const res = computeAward($("fDomain").value, $("fActivity").value, $("fDate").value || isoDate(new Date()));
  if(!res) return;
  let txt = `→ +${res.awarded} XP`;
  let sub = [];
  if(res.bonus>0) sub.push(`momentum +${Math.round(res.bonus*100)}%`);
  if(res.capped) sub.push(`past today's soft cap — overflow earns at ${CONFIG.softCapOverflowRate*100}%`);
  $("xpPreview").innerHTML = txt + (sub.length? `<small>${sub.join(" · ")}</small>`:"");
}
function logEntry(){
  const dateStr = $("fDate").value || isoDate(new Date());
  const res = computeAward($("fDomain").value, $("fActivity").value, dateStr);
  if(!res) return;
  const before = levelFromXp(totalXp());
  state.log.push({
    ts: new Date().toISOString(), date: dateStr,
    domain: $("fDomain").value, activity: $("fActivity").value,
    note: $("fNote").value.trim(), awarded: res.awarded,
  });
  $("fNote").value="";
  save(); renderAll();
  const after = levelFromXp(totalXp());
  if(after>before){ $("shield").classList.remove("levelup"); void $("shield").offsetWidth;
    $("shield").classList.add("levelup"); toast("LEVEL UP — "+after); }
  else toast("+"+res.awarded+" XP");
}

/* ---------- export / import ---------- */
function exportData(){
  const blob = new Blob([JSON.stringify(state,null,2)],{type:"application/json"});
  const a = document.createElement("a");
  a.href = URL.createObjectURL(blob); a.download = "phalanx-data.json"; a.click();
  URL.revokeObjectURL(a.href);
  toast("Exported — commit it to the repo");
}
function importData(file){
  const r = new FileReader();
  r.onload = ()=>{
    try{
      const incoming = JSON.parse(r.result);
      if(!incoming.log || !incoming.badges) throw new Error("not a phalanx file");
      const ok = confirm(
        `Replace current data?\n\nLocal:    ${state.lastModified||"(empty)"} — ${state.log.length} entries\nIncoming: ${incoming.lastModified} — ${incoming.log.length} entries\n\nOK = use incoming file. Cancel = keep local.`);
      if(ok){ state = incoming; save(); renderAll(); toast("Imported"); }
    }catch(e){ alert("Import failed: "+e.message); }
  };
  r.readAsText(file);
}

/* ---------- misc ---------- */
function toast(msg){
  const t=$("toast"); t.textContent=msg; t.style.display="block";
  clearTimeout(t._h); t._h=setTimeout(()=>t.style.display="none", 2600);
}
function escapeHtml(s){ return s.replace(/[&<>"']/g, c=>({"&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;","'":"&#39;"}[c])); }
function renderAll(){ renderHeader(); renderDomains(); renderLog(); renderArmory(); }

$("logBtn").onclick = logEntry;
$("exportBtn").onclick = exportData;
$("importBtn").onclick = ()=>$("importFile").click();
$("importFile").onchange = e=>{ if(e.target.files[0]) importData(e.target.files[0]); e.target.value=""; };

initTicks(); renderForm(); renderAll();
</script>
</body>
</html>

```

## Navigation & architecture

Four tabs, replacing the current single-scroll layout:

1. **War Room** — the default/landing tab. Push-line formation visual, current weekly campaign (enemy name, fog-of-war state, recon action), readiness readout, and the Reinforce / Upgrade / Research spending actions for all four currencies.
2. **Log** — the existing entry form (domain/activity/note/date, XP preview, log button) plus the existing domain daily-cap bars and recent-log list. Same fields, same behavior, restyled to fit the war room visual language.
3. **Armory** — existing Prestige Badges (Certifications / Career / Physical groups, same click-to-cycle status behavior) plus the boss tier presented as flavor text/status layered on top of the relevant in-progress badges. Don't build a separate boss list UI — augment the existing badge cards.
4. **Research** — the Research tree: nodes, costs, unlock states, and the formation picker (Wall/Wedge) once Wedge is unlocked.

The header (shield ring, level, total XP, XP-to-next, momentum pips) stays persistent and visible across all four tabs — it's the one thing that's always on screen, matching its "sacred core stat" status.

## Refresh behavior

Recompute everything (readiness, push line, domain bars, log, header) on page load and on tab navigation — exactly matching how the current file already works (`renderAll()` on load, re-render after any state change). **Do not** build a live/ticking `setInterval` loop. Nothing should visibly move on its own while the user is just sitting on the page; it updates when they act or reload.

## Visual & animation guidance

Keep this restrained — the current file already sets the tone (it only animates the level-up pulse). Extend that same restraint:

- CSS transitions on value changes: bars filling, the push-line contact point easing to its new position, a brief highlight when Command Points are spent. All should respect `prefers-reduced-motion`, same as the existing `.levelup` animation already does.
- No sprite-based clash animation, no elaborate combat sequences. The push line moving smoothly to its new resting position when the tab is opened or after logging an entry *is* the "battle" — it doesn't need a separate cinematic on top.
- Flat shapes only for the troop icons (simple SVG or CSS, matching the existing inline-SVG shield ring in style) — no image assets, no icon libraries beyond what's already inline in the current file.

## State schema — suggested additions

Extend the existing `{ version, lastModified, log[], badges{} }` shape. Exact key names are your call, but the new state needs to cover:

- Per-domain currency balances (earned minus spent) for Intel/Supply/Muscle/Materiel.
- Per-domain Army Size (permanent count) and Upgrade tier (Recruit/Veteran/Elite + progress toward next).
- Current weekly campaign: which enemy, whether it's been confirmed vs. still just suggested, fog-of-war reveal state, campaign start date.
- Boss/regroup state per relevant badge: attempt count, last attempt date, cooldown status — only needed for badges that are currently `in-progress` (i.e., an active boss).
- Unlocked Research node IDs, and current formation selection (defaults to Wall).

Bump `version` to `2` and write a small migration in `load()` so existing (even empty) save files upgrade cleanly with sensible defaults rather than erroring.

## Deployment / commit workflow

You cannot push directly — the human runs a curl command locally. Repo and Edge Function details:

- GitHub repo: `https://github.com/tharold4096/phalanx`
- GitHub Pages: `https://tharold4096.github.io/phalanx`
- Supabase project URL: `https://aqscpevhzexgjalocurl.supabase.co`
- Deployed Edge Function: `github-push` (`verify_jwt: true`)

Edge Function contract:
```
POST /functions/v1/github-push
Authorization: Bearer <anon-jwt>
Content-Type: application/json
{ "path": string, "content": string, "message"?: string, "branch"?: string }
```

For a file this size, use `--data @filename` rather than inlining the content directly in the curl command. The anon JWT is the legacy key (a real JWT), not the `sb_publishable_` key. End your response with a ready-to-run curl command targeting `path: "index.html"`.

## Build process guidance

This is a large single file by the time it's done — realistically several thousand lines. Don't try to write it in one undifferentiated pass. Work in clear internal stages and keep the file coherent at each step:

1. Extend the state schema and CONFIG block first (currencies, army size/readiness, research nodes, campaign state) — get the data model right before touching rendering.
2. Port forward the engine functions that must stay identical (XP, level, momentum, soft caps) unchanged, then add the new engine functions (currency accrual, push-line force calculation, readiness-from-momentum mapping, boss threshold checks).
3. Build the four-tab shell and persistent header.
4. Build each tab's rendering, starting with Log and Armory (lowest risk, closest to existing code) before War Room and Research (net-new).
5. Layer in the CSS transitions/animations last, once the underlying values are correctly computing.

If anything in this brief is ambiguous or you have to make a judgment call on a number/threshold/naming choice, state your assumption briefly and keep moving — don't stop and wait unless something is genuinely blocking. This should end with a complete, working `index.html` and the curl command, ready for review.

## What NOT to do

- Don't change the XP formula, momentum tiers, soft cap logic, badge list, or domain/activity list.
- Don't add any downside to weekly skirmish losses.
- Don't build a separate boss-state system disconnected from the existing badges.
- Don't add a live/ticking auto-refresh loop.
- Don't reach for animation libraries, icon fonts, or image assets — stay dependency-free like the current file.
- Don't auto-award any badge or boss victory — everything achievement-related stays manual-confirm.
