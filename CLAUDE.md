# TechVenture Co. — Project Architecture

> Single-file HTML idle startup game. All CSS, HTML, and JS live in `index.html` (~4200 lines, ~76K tokens).
> Read this file FIRST in every session — it saves ~70K tokens by avoiding a full file read.

---

## 📁 File Structure Map (`index.html`)

| Lines | Section | ~Tokens |
|-------|---------|---------|
| 1–6 | `<head>`, meta, title | 50 |
| 7–420 | `<style>` — CSS variables, components, overlays | ~9,500 |
| 421–570 | HTML overlays (options, tech-pivot, takeover, espionage, IPO screens) | ~1,200 |
| 571–860 | HTML game layout (top-bar, stats-row, tabs, tab panels) | ~1,500 |
| **JS Script starts ~line 870** | | |
| 870–1150 | Constants: `STAGES`, `DEPTS`, `INDUSTRIES`, `UPGRADES`, `PROJECTS`, `BOSS_EVENTS`, `ACQUISITIONS`, `LEGACY_PERKS`, `FOUNDER_SPECS`, `DAILY_TYPES`, `STORY_MILESTONES`, `SKILL_TREES_S`, `SIG_PASSIVES`, `GEAR_ITEMS`, `INIT_COMPETITORS`, `COMPETITOR_MOVES` | ~9,600 |
| 1153–1217 | **`G` state object** (master game state) | ~1,400 |
| 1218–1255 | `DIFFICULTY_MODS`, `INDUSTRY_VERTICALS`, `STOCKS` | ~900 |
| 1256–1480 | `TECH_PIVOTS`, `SPY_MISSIONS`, `triggerHostileTakeover`, `tickEspionage`, `renderEspionage` | ~4,400 |
| 1481–1670 | `IPO_PITCHES`, `IPO_PRICE_TIERS`, `IPO_MID_EVENTS`, all IPO functions | ~4,200 |
| 1671–1780 | Helper utils: `fmt`, `fmtTime`, `addLog`, `addHeadline`, `getGameDay`, `getRepMult` | ~800 |
| 1781–1807 | **`saveGame()`** | ~600 |
| 1808–1870 | **`loadGame()`** | ~700 |
| 1871–2143 | Employee helpers: `createEmployee`, `giveExp`, `levelUp`, `fireEmployee`, `getEmpMPS` | ~3,000 |
| 2144–2543 | **`recalcMPS()`**, `getClickPower`, `getTotalPayroll`, `getProjRewardMult`, `getCompanyEffects` | ~4,800 |
| 2544–3009 | `renderEmpList()`, `renderGearTab`, `renderStockTab`, `renderResearchTab` | ~10,600 |
| 3010–3055 | `calcPrestigeLP`, `startPrestige`, `confirmPrestige` | ~700 |
| 3056–3280 | `render()` — main UI render, all tab renders, `renderCompany` | ~4,000 |
| 3281–3444 | `saveGame`/`loadGame`/`startGame` wiring, `startGameLoop` | ~700 |
| 3445–3977 | Game systems: `checkStage`, `doWork`, `doGacha`, `startProject`, `doResearch`, `tickProjects`, `tickStress`, `tickCompetitors`, `tickReputation`, `applyPassives`, `applySigPassives` | ~14,000 |
| 3978–4010 | **`runGameTick()`** — master tick (called every real second) | ~200 |
| 4011–4216 | `tickStocks`, `startGameLoop`, event handlers, `window.onload` | ~500 |

---

## 🧠 Global State: `G` Object (line 1153)

### Core Economy
```js
G.money          // current cash
G.totalEarned    // cumulative (used for stage check)
G.baseMps        // base money-per-second before multipliers
G.clickPower     // base click value
G.techDebt       // 0–100, slows projects
```

### Progression
```js
G.stage          // 0=Seed, 1=SeriesA, 2=SeriesB, 3=SeriesC, 4=IPO
G.stages[]       // ['Seed','Series A','Series B','Series C','IPO']
G.stageCap[]     // [1000, 15000, 250000, 3000000, 30000000]
G.reputation     // affects gacha, prestige LP
G.legacyPoints   // LP — prestige currency
G.prestigeCount  // number of prestiges done
G.unlockedLegacy // array of legacy perk IDs
G.legacyEffects  // compiled effect object from unlockedLegacy
```

### Time
```js
G.gameTime       // game minutes elapsed (starts at 8*60 = Day 1 08:00)
G.speedMult      // 0=paused, 1=normal, 2=fast
G.tick           // integer tick counter (increments each real second × speedMult)
// GAME_MIN_PER_TICK = 2  →  1 real sec = 2 game min = 1 game day = 12 real min
```

### Employees
```js
G.employees[]    // array of employee objects (see Employee Shape below)
G.selectedEmpId  // currently selected employee ID
G.nextEmpId      // auto-increment ID
G.sortMode       // 'dept'|'mps'|'rarity'|'stress'
```

### Projects & Research
```js
G.activeProjects[]    // projects in progress
G.availableProjects[] // projects available to start
G.projectsDone        // counter
G.researchPoints      // RP
G.unlockedResearch    // Set of unlocked node IDs
G.researchEffects     // compiled effects from research tree
```

### Market & Competitors
```js
G.marketPhase    // 'boom'|'normal'|'recession'
G.competitors[]  // [{name,mps,reputation,color,bg,personality,lastMoveDay,ipoTrigger}]
G.stocks         // {prices:{}, holdings:{}, costBasis:{}}
```

### Systems (Batches 1–3)
```js
G.techPivot             // null | 'blockchain'|'ai_first'|'green'|'platform'
G.deptCollapse          // {[deptId]: bool}
G.espCooldown           // ticks remaining on cooldown
G.counterIntelUntil     // gameTime when counter-intel expires
G.spyMpsBuff            // {mult, until} | null
G.spyMpsDebuff          // {mult, until} | null
G.spyResearchDisc       // {disc, until} | null
G.absorbedMPS           // MPS gained from hostile takeover
G.takeoverCooldownDay   // game day when takeover cooldown expires
G.ipoPhase              // null|'roadshow'|'pricing'|'live'|'payout'
G.ipoPitchesDone[]      // pitch IDs completed
G.ipoValMult            // 1.0 base, modified by pitches
G.ipoPriceTier          // 'conservative'|'balanced'|'aggressive'
G.ipoStockPrice         // current stock price during IPO live phase
G.ipoTargetPrice        // target price set at launch
G.ipoLiveTicks          // countdown ticks for live trading (starts 20)
G.ipoBonusLP            // LP calculated after payout
```

---

## 🔧 Employee Shape

```js
{
  id, name, rarity,        // 'C'|'B'|'A'|'S'
  dept,                    // 'eng'|'mkt'|'design'|'hr'|'finance'
  mps,                     // base MPS contribution
  salary,                  // daily salary
  level, exp, expToNext,
  stars,                   // 0–5
  skills,                  // [{nodeIndex, branch?}] unlocked skill nodes
  gear,                    // {tool, workspace, network} → gearId | null
  stress,                  // 0–100
  burnout,                 // bool
  onProject,               // bool
  loyalty,                 // bool — immune to poaching
  specialTitle,            // string | null
  storyMilestonesDone,     // Set of milestone thresholds seen
}
```

---

## ⚙️ Key Constants

```js
GAME_MIN_PER_TICK = 2      // time conversion
RARITY_RANK = {S:0,A:1,B:2,C:3}  // lower = rarer/better

// Stage caps (G.totalEarned thresholds)
Seed → Series A   : ฿1,000
Series A → B      : ฿15,000
Series B → C      : ฿250,000
Series C → IPO    : ฿3,000,000
IPO complete      : ฿30,000,000

// MPS multiplier chain in recalcMPS() (line 2144):
base × difficulty × industry × techPivot × market × research × legacy × spy buff/debuff + absorbedMPS
```

---

## 🔄 Game Loop (`runGameTick`, line 3978)

Called every **real second** (×speedMult times if sped up):
```
applyPassives → applySigPassives → tickStress → tickCompetitors
→ tickReputation → tickProjects → tickTechDebt → doAutoClicks
→ tickStocks → tickEspionage → tickIPOMarket → checkStage → checkPayday → render
```

---

## ✅ Adding a New Feature — Checklist

Every new game system needs ALL of these. Miss one = bug.

### 1. Data / Constants
```js
const MY_SYSTEM = [...];  // define near line 1256 area (after existing batch data)
```

### 2. State in `G` (line 1153)
```js
// Add to G object
myField: null,
myCounter: 0,
```

### 3. Effect function pattern
```js
function getMySystemEffect() { return getMySystem()?.effect || {}; }
```

### 4. Wire effects into calculation functions
| Function | What to add |
|----------|------------|
| `recalcMPS()` line 2144 | `if(getMySystemEffect().mpsMult) total *= getMySystemEffect().mpsMult;` |
| `getClickPower()` | `*(getMySystemEffect().clickMult \|\| 1)` |
| `getTotalPayroll()` | `*(getMySystemEffect().salaryMult \|\| 1)` |
| `getProjRewardMult()` | `*(getMySystemEffect().projRewardMult \|\| 1)` |
| `tickReputation()` | `+(getMySystemEffect().repBonus \|\| 0)` |
| `doResearch()` cost | `*(getMySystemEffect().researchCostMult \|\| 1)` |
| `pickGachaWeighted()` sRateMult | `*(getMySystemEffect().sRateMult \|\| 1)` |

### 5. Tick function (add to game loop line 3990)
```js
function tickMySystem() {
  if (G.myPhase !== 'active') return;
  // logic here
}
// In runGameTick(): ...tickIPOMarket(); tickMySystem();
```

### 6. Render function
```js
function renderMySystem() {
  const el = document.getElementById('my-system-display');
  if (!el) return;
  el.innerHTML = `...`;
}
// Call from render() or relevant tab render
```

### 7. Save (line 1781)
```js
myField: G.myField || null,
myCounter: G.myCounter || 0,
```

### 8. Load (line 1808)
```js
G.myField = d.myField || null;
G.myCounter = d.myCounter || 0;
```

### 9. Prestige Reset (line 3056 in `confirmPrestige`)
```js
myField: null,
myCounter: 0,
```

### 10. HTML overlay (if full-screen UI needed)
```html
<!-- Add before </body>, pattern: position:fixed overlay with id="my-screen" -->
<div id="my-screen" class="screen-overlay" style="display:none">
  ...
</div>
```
Use `_showEventOverlay()` / `_closeEventOverlay()` for event popups (prevents stacking).

---

## 🎨 CSS Conventions

```css
/* Color variables — always use these, never hardcode */
--color-text-primary / secondary / success / warning / danger / info
--color-background-primary / secondary / tertiary
--color-border-primary / secondary / tertiary

/* Component classes */
.panel          /* card container */
.sm-btn         /* small inline button */
.prestige-btn   /* large CTA button */
.screen-overlay /* position:fixed full-screen overlay, z-index:1000 */
.sec-title      /* section label (10px uppercase) */
.badge-S/A/B/C  /* rarity badges */
```

---

## 📊 MPS Calculation Chain (`recalcMPS`, line 2144)

```
1. Sum base MPS from employees (getEmpMPS each)
2. × difficulty mod (diffMod().mps)
3. × industry effect (getIndEffect().mpsMult)
4. × tech pivot (getTechPivotEffect().mpsMult)
5. × market phase (boom=1.4, normal=1.0, recession=0.6)
6. × research effects (G.researchEffects.mpsMult)
7. × legacy effects (G.legacyEffects.mpsMult)
8. × company upgrades (getCompanyEffects())
9. × spy MPS buff (G.spyMpsBuff)
10. × spy MPS debuff (G.spyMpsDebuff)
11. + G.absorbedMPS (hostile takeover absorbed MPS)
```

---

## 💾 Save/Load Key

- **Save key**: `localStorage.getItem('techventure_save')`
- **Format**: JSON string
- **Backward compat**: always use `d.field || defaultValue` in loadGame
- **Special case**: `G.ipoPhase === 'live'` is reset to `null` on load (can't resume mid-IPO)
- **Sets**: `G.unlockedResearch` is serialized as Array, deserialized back to `new Set()`

---

## 🚀 Prestige / IPO Flow

```
G.totalEarned >= stageCap[4] (฿30M)
  → startIPO()                    // shows IPO screen, sets G.ipoPhase='roadshow'
  → attemptPitch(id)              // roadshow pitches, updates G.ipoValMult
  → showIPOPricing()              // G.ipoPhase='pricing'
  → selectIPOPriceTier(id)        // G.ipoPhase='live', starts 20-tick market simulation
  → tickIPOMarket()               // runs each tick, updates G.ipoStockPrice
  → (after 20 ticks) G.ipoPhase='payout', calculates G.ipoBonusLP
  → confirmIPOPrestige()          // calls confirmPrestige() with bonus LP
  → confirmPrestige()             // resets G to new run, keeps legacyPoints
```

---

## 🕵️ Espionage & Takeover Flow

```
Spy missions (SPY_MISSIONS): cost + cooldown, success/fail chance
  → tech_steal: G.spyMpsBuff = {mult:1.20, until: G.gameTime + 90*GAME_MIN_PER_TICK}
  → intel_leak: G.spyResearchDisc = {disc:0.25, until:...}
  → headhunt: hire free B-rank employee

Competitor spy (every 80 ticks, 18% chance, blocked by counterIntel):
  → debuff MPS, add stress, or steal money

Hostile Takeover (every 120 ticks, stage≥2):
  Trigger condition: competitor.reputation > G.reputation×2 AND >300 AND personality='aggressive'
  Player choices: Fight Back (cost ฿5K, rep -50) | Negotiate (pay ฿8K) | Surrender (lose 30% MPS)
```

---

## 🏗️ Future Refactor Plan (Phases 2–5)

```
Phase 2: src/data/       ← constants, signatures, state definition
Phase 3: src/systems/    ← economy, employees, projects, competitors, ipo, espionage, research, prestige
Phase 4: src/ui/         ← render-main, render-emp, render-company, overlays, headlines
Phase 5: vite.config.js  ← bundle back to dist/index.html + skills/ folder
```
Token savings after Phase 2–4: ~92% per editing session

---

## ⚡ Quick Reference

```js
// Time helpers
getGameDay()                        // current game day number
G.gameTime                          // total game minutes elapsed
N * GAME_MIN_PER_TICK               // convert N real-seconds to game-minutes

// Money formatting
fmt(number)                         // ฿1.23K / ฿4.56M
fmtTime(gameMinutes)                // "Day 5 · 14:30"

// Logging
addLog(text, highlight?)            // add to event log (bottom panel)
addHeadline(text)                   // add to news ticker

// Event overlays (use these, not direct DOM show/hide)
_showEventOverlay(htmlContent)      // shows modal, queues if busy
_closeEventOverlay()                // closes and dequeues next

// Difficulty
diffMod()                           // returns {cost, mps, salary, debt, lpBonus, compAggr}
cm(price)                           // price × diffMod().cost

// Industry
getIndEffect()                      // {mpsMult, clickMult, ...} for current industry

// Tech Pivot
getTechPivotEffect()                // {} if none, else pivot.effect

// Recalculate MPS (call after any employee/effect change)
recalcMPS()

// Re-render entire UI
render()

// Re-render only employee list (lighter)
renderEmpList()
```
