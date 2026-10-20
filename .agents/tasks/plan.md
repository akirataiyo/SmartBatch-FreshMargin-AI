# Implementation Plan: SmartBatch single-file redesign + guided calculator + customer chatbot + AWS explainer

Target files: `index.html` (rewrite in place, still ONE file, inline CSS+JS) and `README.md` (section 5 only).
Review gate (owned by the loop, not by this plan): `.agents/tasks/review.json`, `verdict` == `APPROVED`.

Decision on workflow shape: NOT restructured into FEATs. All four parts live in one HTML file and share one CSS design system and one JS state object, so splitting them would make FEATs edit the same regions and break each other. The existing implement + review loop runs this plan. The workspace is NOT a git repo (`git status` fails), so do not try to commit.

Environment facts (checked): Node v24.13.1 and npm 11.8 are installed. Edge exists at `C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe` (Chrome is absent). No test framework or package.json exists and none should be added.

Persona reminder: "พี่เล็ก", home-bakery/lunch-box owner, phone, LINE, under 3 minutes. All user-facing text is Thai. Numbers in the UI are illustrative, and that must stay visible.

---

## A. Current structure of index.html (806 lines, read in full)

- Lines 1-9: `<html lang="th">`, `<title>`, no fonts link. Lines 9-330: one `<style>`. It uses a Tailwind-blue palette (`--primary:#2563eb`, slate greys), a `linear-gradient(135deg,#0f172a,#1e3a8a)` header, `.card` (white, radius 12, soft shadow), `.grid-2/.grid-3` card grids, and `.badge-aws/.badge-ready` filler badges. Every element uses `font-family: system-ui...`.
- Lines 333-348 `<header>`: "🍲 SmartBatch & FreshMargin AI", two filler badges ("AWS Serverless Ready", "Real-time Recalc Active").
- Lines 351-358 disclaimer card (💡, ends "ใน 3 นาที!").
- ROW 1 (360-470) is a `.grid-2` of two cards:
  - Price radar. 4 `<input type=range>` with ids `priceLime` (1-10, step .5, default 3.5), `pricePork` (130-250, step 5, default 180), `priceEgg` (3-6, step .2, default 4.2), `priceButter` (180-380, step 10, default 260). Readouts: `valLime/valPork/valEgg/valButter`. Preset buttons `applyPreset('normal'|'lime_surge'|'meat_surge')`. Preset values: normal = 3.5/180/4.2/260; lime_surge = 9/185/4.5/270; meat_surge = 4/240/5.2/320.
  - Recipe cost card. `select#menuSelect` (pork_rice | croissant | tomyum), metric boxes, breakdown rows, `#marginAlertBox` (shown when gross margin < 40%), badge `#marginHealthBadge` (<40 danger, <50 warn, else safe). Several values are hard-coded text, for example "รวมค่าหั่นสูญเสีย 12%", the "GP 30%" label, and the 7.50/2.50 breakdown rows.
- ROW 2 (card, 419-535): planner. Inputs `inputPreOrder` (28), `inputStock` (6), `inputWalkIn` (35), `select#batchUnit` (12|24|50, default 24). Three scenario cards `scenCardA/B/C` with ids `scenX_prod/_batches/_profit`, `scenB_breakeven`, `scenC_risk`. `selectScenario()` only restyles the card.
- ROW 3 (538-619): waste card (`wasteQty` default 4, `wasteCost`, `wasteRec`, `logWaste()` which only calls `alert()`), and the AWS card holding an ASCII `<div class="aws-arch-box">` diagram plus 3 bullet lines ("Zero Idle Cost", "Cold-Start Resilient", "Privacy-First").
- Footer (621-625).
- Script (627-803), global functions, state read straight from the DOM on every call:
  - `recipes` = `{pork_rice:{sellingPrice:65, deliveryPrice:89, baseFixedCost:10, calcRawCost(lime,pork,egg,butter)=pork*0.16+egg*0.5+lime*0.2}, croissant:{55, 79, 7, butter*0.045+egg*0.3}, tomyum:{150, 199, 35, lime*3.0+egg*0.2}}`. The 0.16 for pork is 120 g at 75% yield.
  - `calculateAll()` computes costs and the 3 scenarios. These formulas are inconsistent with the README (A = pre-orders only, B = 0.6 walk-in, C = full walk-in, with ad-hoc profit and risk formulas). They are replaced by the clean model in section C.
  - `updateWaste()` computes `wasteQty * totalCost`. Advice thresholds: >6 "ลด Walk-in ลง 25% ทันที", >0 "ปรับลดผลิตลง 1 Batch", else "กำลังผลิตสมดุลยอดเยี่ยม".
  - Inline `oninput/onclick/onchange` handlers everywhere. These get replaced by `addEventListener`.
- Nothing is persisted, there is no validation (`parseInt(...)||0` only), there is no chatbot, and focus styles are removed (`outline:none`).

Existing capabilities that MUST survive: the 4 ingredient price adjusters, the 3 presets, the 3 menus, yield loss, delivery GP, 3 scenarios, batch quantization, per-dish net margin, the waste cost and advice, the margin alert at the 40% target, and real-time recalculation.

---

## B. Decisions (with rationale)

1. **One state object, one pipeline.** `state` (inputs, persisted) -> `Core.derive(state)` (pure, no DOM) -> `derived` -> `render*()` and `Chat`. Every input handler calls `update(patch)` = sanitize -> assign -> `derived = Core.derive(state)` -> `renderAll()` -> `save()` -> `Chat.onStateChange()`. This is what makes the bot's answers change when the calculator changes. No other module may read the DOM for numbers.
2. **Pure logic lives in two marked blocks so Node can test it without a browser.** Wrap the pure code in `/* CORE:BEGIN */ const Core = (() => {...})(); /* CORE:END */` and the chatbot NLU in `/* BOT:BEGIN */ const BotNLU = (() => {...})(); /* BOT:END */`. Neither block may touch `document`, `window` or `localStorage`. The verification script extracts the text between the markers and runs it with `node:vm`.
3. **Scenarios use the README formula with a walk-in coverage factor** `k`: A (Conservative) k=0, B (Balanced) k=0.6, C (Aggressive) k=1.0. For each scenario, `P = max(0, ceil((pre + round(walk*k) - stock)/batch) * batch)`. C with k=1 is exactly the README formula. B is the recommended default (`state.scenario='B'`). The old code used the same 0/0.6/1.0 coverage, so behavior is continuous.
4. **Money model** (replaces the old ad-hoc formulas). Constants: `REALIZATION = 0.7` (walk-ins who actually show up, labelled in the UI as an assumption).
   - `available = P + stock`
   - `demand = pre + round(walk * 0.7)`
   - `sold = min(available, demand)`
   - `revenue = sold * sellPrice`
   - `prodCost = P * unitCost` (existing stock is sunk cost)
   - `profit = revenue - prodCost`
   - `soldFromNew = max(0, sold - stock)`
   - `atRisk = (P - soldFromNew) * unitCost` (money at risk if unsold)
   - `lost = max(0, demand - available)` (sales missed)
   - `breakEvenUnits = P>0 ? ceil(prodCost / sellPrice) : 0`
   - Keep every number as a float and round only for display.
5. **Yield loss becomes real.** Each recipe line is `{ing, qty, trim}`. The raw cost of a line is `price[ing] * qty / (1 - trimLossPct/100)` when `trim:true`, otherwise `price*qty`. The only trimmed line is pork in pork_rice (qty 0.12 kg, default loss 25%), which equals the old 0.16 factor. The "ตั้งค่าเพิ่มเติม" control is disabled with a Thai note for menus with no trim line. `unitCost = rawCost + fixedCost` (fixedCost: pork_rice 10, croissant 7, tomyum 35, as before).
6. **Delivery net margin.** `deliveryNet = deliveryPrice * (1 - gp/100)`, `deliveryMarginPct = (deliveryNet - unitCost)/deliveryNet*100`, `storeMarginPct = (sellPrice - unitCost)/sellPrice*100`. Status vs `targetMarginPct` (default 40): `< target` = "ต่ำกว่าเป้า" (danger), `< target+10` = "เฝ้าระวัง" (warn), otherwise "ผ่านเป้า" (ok). Every status is shown as an icon plus a Thai word, never color alone. Note: with the unchanged recipe data, pork_rice at benchmark prices has unitCost 41.60 and a store margin of 36.0%, so it shows the warning on first load. Keep it. It demonstrates the feature and the data stays as it was.
7. **Per-menu plans.** `state.plans[menuId] = {pre, walk, stock, batch}` so the bot can quote stock for all 3 dishes. Defaults: pork_rice `{28,35,6,24}` (the old defaults), croissant `{20,30,5,24}` (the README sanity example), tomyum `{6,10,2,12}`. Step 2 edits the plan of `state.menuId`.
8. **Chatbot "sold so far" is simulated, deterministic and visible.** `soldSim = min(available, pre + round(walk * chat.progress))`, `chat.progress` default 0.4, changeable with a small "จำลองเวลาขาย" slider in the chat panel. `remaining = max(0, available - soldSim - chatOrders[menuId])`. Pre-orders made in chat only reduce the bot's own `remaining`. They do not feed back into the calculator (out of scope; mention nothing about it in the UI).
9. **Palette and type.** Warm paper background, one ink, one accent, plus two small semantic tones that always come with an icon and a word.
   - Tokens: `--paper:#F6F0E4`, `--paper-2:#EDE3CE`, `--ink:#221E1A`, `--ink-2:#5B5247` (secondary text, about 6.7:1 on paper), `--rule:#CFC3AB` (decoration only, never text), `--accent:#B5361E` (chili red, about 5.3:1 on paper), `--ok:#2F6B3F`, `--warn:#8A5A00`.
   - Focus ring: `outline:3px solid var(--ink); outline-offset:3px`, plus a paper inner ring on ink backgrounds.
   - Fonts: `<link>` to Google Fonts Sarabun (wght 400;600;700;800, `display=swap`) with preconnect. Stack: `'Sarabun','Noto Sans Thai','Leelawadee UI','Tahoma',system-ui,sans-serif`, so offline it falls back to system Thai fonts.
   - Base size 17px, line-height 1.6 (Thai marks need room), headings line-height >= 1.25. Money and counts use `font-variant-numeric: tabular-nums`.
   - Remove: gradients, soft shadows, glass, emoji, the two filler badges, uniform white card grids.
10. **Layout.** Left-aligned editorial masthead. Desktop (>=960px): a 12-column grid with a sticky left rail (about 4 columns) holding the 3-step index and the live headline, and the content column (about 8 columns) on the right. Mobile (<960px): single column; the rail becomes a sticky top strip showing "1 ราคา · 2 ยอดขาย · 3 ผลลัพธ์" plus the live "ผลิต N ชิ้น". Steps are stacked sections (no hidden panels), so everything is reachable with Tab and scroll. Dividers are 1px `--rule` lines, tables are ledger-style, and the result is a "kitchen ticket" with a 2px ink border and a dashed divider.
11. **Icons.** Hidden `<svg>` sprite with `<symbol>`s at the top of `<body>` (viewBox 0 0 24 24, `fill:none; stroke:currentColor; stroke-width:1.75; stroke-linecap:round; stroke-linejoin:round`). Use them as `<svg class="ic" aria-hidden="true" focusable="false"><use href="#i-name"/></svg>`. Same-document refs work from `file://`. Symbols: `i-plus i-minus i-up i-down i-reset i-warn i-check i-chevron i-chat i-close i-send i-store i-phone i-globe i-door i-bolt i-db i-eye i-line i-receipt i-basket`.
12. **No inline event handlers, no `innerHTML` with variable content anywhere.** Static template markup is written in the HTML file. The only dynamic DOM creation is via `createElement`/`createElementNS` + `textContent`/`setAttribute`. The chatbot must never use `innerHTML`, `insertAdjacentHTML` or `outerHTML` (the injection test below depends on this).

---

## C. The shared JS state (exact shape)

```js
const STORAGE_KEY = 'sb.state.v2';        // JSON of `state` (below)
const WASTE_LOG_KEY = 'sb.wasteLog.v1';   // JSON array, newest last, max 30
const BENCHMARK = { lime:3.5, pork:180, egg:4.2, butter:260 };   // "ราคาอ้างอิงตัวอย่าง"

const state = {
  v: 2,
  menuId: 'pork_rice',                    // 'pork_rice' | 'croissant' | 'tomyum'
  prices: { lime:3.5, pork:180, egg:4.2, butter:260 },    // today's prices (Step 1)
  plans: {                                // Step 2, one per menu
    pork_rice: { pre:28, walk:35, stock:6, batch:24 },
    croissant: { pre:20, walk:30, stock:5, batch:24 },
    tomyum:    { pre:6,  walk:10, stock:2, batch:12 }
  },
  menuCfg: {                              // advanced, per menu
    pork_rice: { sellPrice:65,  deliveryPrice:89,  trimLossPct:25 },
    croissant: { sellPrice:55,  deliveryPrice:79,  trimLossPct:0  },
    tomyum:    { sellPrice:150, deliveryPrice:199, trimLossPct:0  }
  },
  gpPct: 30,                              // delivery platform GP %, 0..40
  targetMarginPct: 40,                    // 10..80
  scenario: 'B',                          // 'A'|'B'|'C' chosen plan, headline + bot use this
  wasteQty: 4,                            // leftover at close, 0..999
  chat: { progress: 0.4 }                 // simulated sales clock 0..1 (persisted)
};
// Not persisted, memory only:
let lastKnown = {...BENCHMARK};           // = prices loaded from storage at page start (else BENCHMARK); shown as "ครั้งก่อน"
const chatRuntime = { open:false, orders:{pork_rice:0,croissant:0,tomyum:0}, flow:null, history:[] };
let derived = Core.derive(state);
```

Static recipe table (extends the old `recipes`; names/prices unchanged). Per-menu extras are used by the bot.

```js
const RECIPES = {
  pork_rice: { name:'ข้าวหมูกรอบคั่วพริกเกลือ', short:'ข้าวหมูกรอบ', unitWord:'จาน', trayWord:'รอบ', fixed:10,
    lines:[{ing:'pork',qty:0.12,trim:true},{ing:'egg',qty:0.5},{ing:'lime',qty:0.2}],
    aliases:['ข้าวหมู','หมูกรอบ','ข้าวหมูกรอบ','คั่วพริกเกลือ','ข้าวกล่อง'],
    allergens:'มีหมู ไข่ และเครื่องปรุงที่อาจมีถั่วเหลืองหรือปลา', shelf:'ควรกินภายในวันเดียวกัน เก็บเกิน 4 ชั่วโมงให้แช่เย็นและอุ่นก่อนกิน' },
  croissant: { name:'ครัวซองต์เนยสดพรีเมียม', short:'ครัวซองต์', unitWord:'ชิ้น', trayWord:'ถาด', fixed:7,
    lines:[{ing:'butter',qty:0.045},{ing:'egg',qty:0.3}],
    aliases:['ครัวซอง','ครัวซองต์','ครอซอง','croissant','ขนมปัง','เบเกอรี่'],
    allergens:'มีแป้งสาลี (กลูเตน) เนยนม และไข่', shelf:'อยู่ได้ 1-2 วันที่อุณหภูมิห้อง แช่เย็นได้ 3 วัน อุ่นเตาอบ 160 องศา 5 นาที' },
  tomyum: { name:'ต้มยำกุ้งน้ำข้น', short:'ต้มยำกุ้ง', unitWord:'ถุง', trayWord:'หม้อ', fixed:35,
    lines:[{ing:'lime',qty:3.0},{ing:'egg',qty:0.2}],
    aliases:['ต้มยำ','ต้มยำกุ้ง','กุ้ง','น้ำข้น','tomyum','ซุป'],
    allergens:'มีกุ้ง (สัตว์น้ำเปลือกแข็ง) นมข้น และน้ำปลา', shelf:'กินวันเดียวกัน หรือแช่เย็นไม่เกิน 1 วัน อุ่นให้เดือดก่อนกิน' }
};
// SHOP (demo data, no real address/phone):
const SHOP = { name:'ร้านพี่เล็ก (ตัวอย่าง)', open:'06:30', close:'16:00', closedDay:'วันอาทิตย์',
  where:'ซอยตัวอย่าง ย่านออฟฟิศ (ข้อมูลสมมติ)', pickup:'รับเองที่ร้าน หรือสั่งผ่านแอปเดลิเวอรี่' };
```

`Core` API (pure; exported object):
- `sanitize(raw, {min,max,int,fallback})` returns `{value, ok, msg}`. Empty, NaN or non-numeric -> `ok:false`, `value:fallback`, msg "กรอกเป็นตัวเลขนะครับ ตอนนี้ใช้ค่าเดิมให้ก่อน". Negative -> clamped to `min`, msg "ใส่ค่าติดลบไม่ได้ ปรับเป็น 0 ให้แล้ว". Above max -> clamped, msg "มากสุด {max} ปรับให้แล้ว". `int:true` rounds with `Math.round`. Never returns NaN or Infinity.
- `targetProduction(pre, targetWalk, stock, batch)` = `Math.max(0, Math.ceil((pre + targetWalk - stock) / batch) * batch)`. This is the README formula (guard `batch>0`, else return 0).
- `rawCost(menuId, prices, trimLossPct)`, `unitCost(menuId, prices, cfg)`, `margins(unitCost, cfg, gpPct)` -> `{storePct, storeProfit, deliveryNet, deliveryPct, deliveryProfit}`, `marginStatus(pct, target)` -> `'low'|'watch'|'ok'`.
- `scenario(key, plan, unitCost, sellPrice)` -> `{key, k, targetWalk, need, batches, P, available, demand, sold, revenue, prodCost, profit, soldFromNew, atRisk, lost, breakEvenUnits}` where `need = pre + targetWalk - stock`.
- `derive(state)` -> `{ menus:{[id]:{unitCost, margins, status, scen:{A,B,C}, chosen, available, soldSim, remaining(before chat orders)}}, active: derived.menus[state.menuId], priceDelta:{lime:{abs,pct},...} }`.
- Constants exported: `SCENARIOS = {A:{k:0,label:'เซฟ'}, B:{k:0.6,label:'สมดุล'}, C:{k:1,label:'รุก'}}`, `REALIZATION = 0.7`.

Hand-computed expectations the unit test asserts (croissant, benchmark prices):
- `unitCost = 260*0.045 + 4.2*0.3 + 7 = 11.7 + 1.26 + 7 = 19.96`
- plan `{pre:20, walk:30, stock:5, batch:24}`:
  - README example: `targetProduction(20,30,5,24)` = `ceil(45/24)*24` = **48**
  - A: P=24 (`ceil(15/24)=1`), available 29, demand 41, sold 29, profit `29*55 - 24*19.96` = 1115.96, atRisk 0, lost 12
  - B: P=48 (`20+18-5=33`, `ceil(33/24)=2`), available 53, demand `20+round(21)=41`, sold 41, revenue 2255, prodCost 958.08, profit 1296.92, soldFromNew 36, atRisk `12*19.96` = 239.52, breakEvenUnits `ceil(958.08/55)` = 18
  - C: P=48, the same as B (this is a legitimate result, not a bug)
- Edge cases: `targetProduction(0,0,10,24)` = 0 (stock covers it, never negative); `targetProduction(24,0,0,24)` = 24 (exact multiple does not add a batch); `targetProduction(1,0,0,24)` = 24.
- pork_rice benchmark: `rawCost` = 31.6 (180*0.12/0.75 + 2.1 + 0.7), `unitCost` = 41.6, store margin = 36.0%, status `low` at target 40.

---

## D. Part 3 — guided 3-step calculator (exact DOM and behavior)

Markup (`<main id="main">`, each step a `<section class="step" id="step1|2|3" aria-labelledby="...">` with an `<h2>`; the step number is a styled numeral in the heading):

**Step 1 `#step1` — h2 "วันนี้ของแพงขึ้นไหม"**. Intro line: "เทียบกับราคาอ้างอิงตัวอย่าง ไม่ใช่ราคาตลาดจริง". A ledger as a `<table class="ledger">` with a visually hidden `<caption>`, switching to stacked rows under 720px with `display:block` on `tr`/`td` and `data-label`. Rows are lime (บาท/ลูก, range 1-10, step .5), pork (บาท/กก., 130-250, step 5), egg (บาท/ฟอง, 3-6, step .2), butter (บาท/กก., 180-380, step 10). Per row:
- A `<label for="in-lime">` with the ingredient name and unit.
- A "อ้างอิง 3.50" cell and a "ครั้งก่อน 3.50" cell (the `lastKnown` value).
- A stepper: `<button type="button" aria-label="ลดราคามะนาว">` (minus icon) + `<input id="in-lime" type="number" inputmode="decimal" step min max>` (the big readout, font about 1.6rem, tabular) + plus button. All three controls are at least 48px high.
- Below the stepper, an `<input type="range" id="rg-lime" aria-label="เลื่อนปรับราคามะนาว">` bound to the same state, so the existing slider capability is kept.
- A change cell `<output id="chg-lime" aria-live="polite">`: an up/down icon plus text like "แพงขึ้น 12% จากอ้างอิง" / "ถูกลง 8%" / "เท่าอ้างอิง". Up uses the accent color and bold, down uses `--ok`, and the text always carries the meaning.
- A `<p class="field-msg" id="msg-lime" role="status">` for validation messages.
- Under the table: a button "ใช้ราคาอ้างอิงทั้งหมด" (id `btnResetPrices`, reset icon, sets `state.prices = {...BENCHMARK}`) and a "ลองสถานการณ์" group with 3 buttons: "ราคาปกติ", "มะนาวแพง", "หมูพุ่ง". They use the old preset values and set `aria-pressed` when they match exactly.

**Step 2 `#step2` — h2 "วันนี้คาดว่าขายได้เท่าไร"**.
- `<select id="menuSelect">` with a visible label "เมนูที่จะทำ" (3 options from `RECIPES`).
- A `<fieldset>` "วันนี้เป็นวันแบบไหน" with 3 preset buttons: "วันธรรมดา" `{pre:12, walk:30, stock:4}`, "วันหยุด" `{pre:8, walk:50, stock:0}`, "มีออเดอร์ออฟฟิศ" `{pre:40, walk:20, stock:4}`. They set the active menu's plan (batch unchanged), and `aria-pressed` is true when the three values match.
- 3 integer steppers, same pattern as Step 1, ids `in-pre` ("ออเดอร์ที่จองไว้แล้ว", help "ลูกค้าจ่ายแล้วหรือจองแล้ว"), `in-walk` ("คาดว่าลูกค้าหน้าร้านซื้อ"), `in-stock` ("ของเหลือจากเมื่อวานที่ยังขายได้"). Range 0-999.
- Batch size: a `<fieldset><legend>1 รอบ/ถาด ทำได้กี่ชิ้น</legend>` with 3 radio inputs `name="batch"` (12, 24, 50) styled as 48px segmented buttons. Visible text "12 ชิ้น", "24 ชิ้น", "50 ชิ้น". The hint "ระบบปัดจำนวนผลิตให้ตรงกับรอบจริงเสมอ".
- `<details id="adv">` "ตั้งค่าเพิ่มเติม (ไม่ต้องแตะก็ได้)" containing number inputs: ราคาขายหน้าร้าน (`in-sell`), ราคาบนแอปเดลิเวอรี่ (`in-deliv`), ค่าหักแพลตฟอร์ม GP % (`in-gp`), กำไรขั้นต่ำที่ต้องการ % (`in-target`, default 40), ของเสียจากการหั่น % (`in-trim`, disabled with the note "เมนูนี้ไม่มีของเสียจากการหั่น" when the recipe has no trim line).

Validation behavior (all numeric inputs, steppers included):
- On `input`: `Core.sanitize`. If `!ok`, show the message in that field's `.field-msg`, set `aria-invalid="true"`, and DO NOT rewrite the field text (so the user can keep typing). The calculation uses the last valid value.
- On `change`/blur: write the clamped value back into the field, clear the error, and keep or show the clamp message briefly. The message persists in the `role=status` region until the next valid edit.
- Stepper buttons use `Core.sanitize` on `value ± step` and fix floating error with `Math.round(v*100)/100`.

**Step 3 `#step3` — h2 "ผลลัพธ์: วันนี้ควรทำเท่าไร"** (the kitchen ticket).
1. `#headlineAnswer` (`aria-live="polite"`): one big line "ผลิต 48 ชิ้น = 2 ถาด" built from the chosen scenario `P`, `RECIPES[menu].unitWord` and `trayWord`. Sub-line: "แผน B (สมดุล): จอง 20 + หน้าร้านที่ตั้งเป้า 18 − ของเหลือ 5 = 33 → ปัดขึ้นเป็น 2 ถาด". If `P==0`: "วันนี้ไม่ต้องทำเพิ่ม ของที่มีอยู่พอแล้ว". The same headline text is mirrored in the sticky rail/strip (`#railHeadline`).
2. `#marginBanner` (`role="status"`) for the active menu: when low, an accent-colored band with the warn icon and bold text "ต่ำกว่าเป้า: {menu} เหลือกำไร {x}% ต่ำกว่าเป้า {target}%" plus one concrete tip ("หน้าร้านยังพอไหว แต่ผ่านเดลิเวอรี่เหลือ {y}%" when the store is OK and delivery is low; otherwise "ลองขึ้นราคา {suggest} บาท หรือลดส่วนผสมที่แพง"). When watch, an amber band with "เฝ้าระวัง". When ok, a quiet "ผ่านเป้า" line. `suggest` = `ceil(unitCost/(1-target/100))`.
3. **Scenario comparison** `<table class="ledger cmp">` with caption "เทียบ 3 แผน" and `<th scope="col">` per plan (แผน A เซฟ / แผน B สมดุล "แนะนำ" / แผน C รุก). Rows (`<th scope="row">`):
   - ผลิต (ชิ้น และ ถาด)
   - ต้นทุนผลิต
   - คาดว่าขายได้
   - กำไรที่คาด
   - เงินจมถ้าขายไม่หมด
   - ขายไม่ทัน (ชิ้น)
   - ต้องขายอย่างน้อยกี่ชิ้นถึงคุ้มทุน
   A "เลือกแผนนี้" `<button aria-pressed>` sits in each column header and sets `state.scenario`. The chosen column gets a 2px ink border plus the text "แผนที่เลือก". Show "แนะนำ" on B as text. When the P of two plans is equal, add under the second: "เท่าแผน {x} เพราะปัดเป็นถาดแล้วเท่ากัน". Money uses `toLocaleString('th-TH')` with `฿`. Money-at-risk cells are bold in accent when > 0.
   Footnote: "คาดการณ์โดยสมมติว่าลูกค้าหน้าร้านมาจริงราว 70% ของที่คาดไว้ (สมมติฐาน ไม่ใช่ข้อมูลจริง)".
4. **Per-dish net margin ledger** `<table class="ledger">`: เมนู | ต้นทุน/จาน | หน้าร้าน กำไร % | เดลิเวอรี่ (หลังหัก GP {gp}%) กำไร % | สถานะ. 3 rows from `derived.menus`. The status cell has an icon + word ("ต่ำกว่าเป้า {target}%" / "เฝ้าระวัง" / "ผ่านเป้า"); low rows also get a 4px accent left rule and bold.
5. `<details id="whyDetails">` summary "ทำไมถึงได้ตัวเลขนี้". Content, rebuilt with `textContent` on every update:
   - The formula text: `ผลิต = ปัดขึ้นเป็นรอบ( (ออเดอร์จอง + เป้าลูกค้าหน้าร้าน − ของเหลือ) ÷ ขนาดรอบ ) × ขนาดรอบ`, and under it the same with the user's numbers: `= ceil((20 + 18 − 5) ÷ 24) × 24 = ceil(1.375) × 24 = 2 × 24 = 48`.
   - A short list: เป้าลูกค้าหน้าร้าน = {walk} × {k} (สัดส่วนของแผนที่เลือก); ต้นทุนต่อชิ้น = วัตถุดิบ {raw} + ค่าอื่น ๆ {fixed} = {unit} ฿ (with the yield-loss line "หมูดิบ 0.12 กก. ÷ (1−25%) = 0.16 กก." when trim applies); กำไรที่คาด = ขายได้ {sold} × ราคา {price} − ต้นทุนผลิต {prodCost}.
   - Each term is wrapped in a `<span class="n">`, and the whole details panel also gets a one-line plain summary.
6. `<section id="wasteBlock">` h3 "หลังปิดรอบ: ของเหลือ" (replaces the old waste card). `in-waste` stepper (default 4), `<output id="wasteCost">` = `wasteQty * unitCost` (1 decimal + ฿, label "เงินจมในถังขยะ"), `#wasteRec` advice with the old thresholds (>6 / >0 / 0) as icon+text, and a button "บันทึกของเหลือวันนี้" -> pushes `{t:ISO date, menu, qty, cost}` to `WASTE_LOG_KEY` in try/catch (cap 30) and writes into `#wasteStatus` (`role="status"`): "บันทึกแล้ว (เก็บในเครื่องนี้เท่านั้น) · สะสม {n} ครั้ง". No `alert()`.

Persistence (`load()` / `save()`):
- `save()`: `try { localStorage.setItem(STORAGE_KEY, JSON.stringify(state)) } catch(e){}`. Debounce 200 ms.
- `load()`: `try { parse } catch { return null }`, then merge field by field through `Core.sanitize` and enum checks (menuId in RECIPES, batch in [12,24,50], scenario in A/B/C), so corrupted or hand-edited data never injects NaN or an unknown key. Anything invalid falls back to the default. `lastKnown` = loaded prices when valid.
- A small "ล้างข้อมูลที่บันทึกไว้" text button in the footer (removes both keys inside try/catch, reloads defaults, re-renders).

---

## E. Part 1 — customer chatbot demo (exact spec)

**DOM** (placed just before `</body>`, markup static, messages dynamic):
- `<button id="chatFab" type="button" aria-expanded="false" aria-controls="chatPanel" aria-label="เปิดแชทถามร้าน (เดโมสำหรับลูกค้า)">` with the chat icon and visible text "ถามร้าน". Fixed bottom-right (16px inset), min 48px high, ink background and paper text.
- `<section id="chatPanel" role="dialog" aria-modal="false" aria-labelledby="chatTitle" hidden>`:
  - `<header>`: `<h2 id="chatTitle">แชทลูกค้า (เดโม)</h2>`, a "ล้างแชท" button, and `<button id="chatClose" aria-label="ปิดแชท">`.
  - `<p class="chat-demo">` small label: "เดโมจำลองในหน้านี้ ไม่มีเซิร์ฟเวอร์ ของจริงจะใช้ Amazon Bedrock ผ่าน Lambda + LINE OA". Always visible, not collapsible.
  - `<div id="chatLog" role="log" aria-live="polite" aria-relevant="additions" tabindex="0" aria-label="ข้อความในแชท">`.
  - `<div id="typing" hidden>` typing indicator: 3 dots (CSS animation, disabled under reduced motion) plus visually hidden text "ร้านกำลังพิมพ์".
  - `<div id="chatChips" aria-label="ตัวอย่างคำถาม">` buttons (min 44px), re-rendered after every bot message.
  - `<form id="chatForm">`: `<label for="chatInput" class="sr-only">พิมพ์ข้อความถึงร้าน</label>`, `<input id="chatInput" maxlength="200" autocomplete="off" enterkeyhint="send">`, `<button id="chatSend" aria-label="ส่งข้อความ">`. Enter submits natively through the form.
  - `<details class="chat-sim">` "ปรับเดโม": `<label>` "ขายไปแล้วประมาณ {x}% ของยอดหน้าร้าน" + `<input type="range" id="chatProgress" min="0" max="100" step="10">`, bound to `state.chat.progress`.
- Desktop: panel 360x520, fixed bottom-right above the FAB. Under 640px: `position:fixed; inset:auto 0 0 0; width:100%; height:min(85dvh, 640px)`, a bottom sheet with a top border, and the FAB hidden while open. Opening/closing animates only `transform/opacity` (<=180 ms) and is skipped under `prefers-reduced-motion`.
- Also add a visible entry point in the intro/masthead: a button "ลองแชทกับบอทลูกค้า (เดโม)" that calls `Chat.open()`.

**Focus and keyboard:**
- Open: `hidden=false`, FAB `aria-expanded=true`, remember `lastFocus`, focus `#chatInput`. The greeting is the first bot message, added on first open only.
- Esc (keydown on the panel and on document while open): `Chat.close()`, which sets `hidden=true`, `aria-expanded=false`, and returns focus to `#chatFab` (or to the opener button).
- Tab stays inside the sheet when it is full-screen mobile (simple trap on first/last focusable); on desktop it's a non-modal popup with no trap.
- Chips are real `<button type="button">`. A chip click acts like a typed message. Every chip text is a short Thai phrase.
- A new message scrolls the log to the bottom (`scrollTop = scrollHeight`, no smooth scroll under reduced motion). Cap the log at 60 messages (remove the oldest).

**Message rendering (security, mandatory):** `addMsg(role, parts)` creates `<div class="msg msg-user|msg-bot">`, appends one `<p>` per line with `p.textContent = line`, plus `<time datetime>` HH:MM (`textContent`). User text goes through the same path. **`innerHTML`, `insertAdjacentHTML`, `outerHTML` and `document.write` must not appear anywhere in the chatbot code.** Also keep the quote of the user's text inside the fallback reply minimal (do not echo it back at all; echoing is unnecessary).

**Typing indicator:** after a user message, show `#typing` and disable chip clicks for the delay, then post the reply. Delay `350 + min(len,120)*4` ms (about 0.4-0.9 s). Under `matchMedia('(prefers-reduced-motion: reduce)')` use 150 ms and no dot animation, but still show the text "ร้านกำลังพิมพ์…". Use a `replyToken` so rapid sends do not interleave.

**NLU (`BotNLU`, pure):**
- `normalize(s)`: NFC; lowercase; map Thai digits ๐-๙ to 0-9; remove tone marks and thanthakhat (U+0E48-U+0E4C); remove zero-width chars and whitespace/punctuation; collapse any char repeated 3+ times to 1 (ครับบบ -> ครับ); trim to 200 chars.
- Keywords are normalized by the same function at load. Matching is **contains on the normalized string**, because Thai has no spaces. Latin keywords of <=3 letters (`hi`, `no`) match only as whole tokens of the original lowercase text (split on non-letters). This avoids "hi" matching inside other words.
- `classify(text, ctx)` -> `{intent, itemId|null, qty|null, score, matched}`. A score is the sum of the lengths of distinct matched keywords. The best score wins; ties go to the higher `prio`. No match -> `intent:'fallback'`. `human` has `prio:100`, and a score >=1 on it always wins (a safety valve).
- Item entity: match `aliases` plus `short` and `name` (normalized) from `RECIPES`. The longest alias match wins. Without an item, intents that need one answer for all 3 dishes.
- Quantity: first `\d+` after normalization (cap 99), or Thai numerals in words (หนึ่ง=1 … สิบ=10) via a small map.
- If `ctx.flow` is set (pre-order in progress), the flow handler gets the first chance (see below). The only global escapes are `ยกเลิก|cancel` -> abort the flow, and a `human` intent.

**Intent / keyword table** (keywords shown un-normalized for readability, normalize at load; "needs item" = answers per item if one is detected, otherwise lists all three):

| intent (prio) | keywords / synonyms | answer (built from live `derived`/`RECIPES`/`SHOP`) | chips after |
|---|---|---|---|
| `human` (100) | คุยกับคน, คุยกับเจ้าของ, เจ้าของร้าน, แอดมิน, พนักงาน, ขอคนจริง, ไม่ใช่บอท, ติดต่อร้าน, โทร, ร้องเรียน, human, admin | "ได้เลยครับ/ค่ะ เดี๋ยวให้เจ้าของร้านตอบทาง LINE ร้าน (เดโม: ยังไม่ได้ส่งข้อความจริง)". Stops any flow. | เมนูวันนี้, สั่งล่วงหน้า |
| `allergen` (60) | แพ้, แพ้อาหาร, ส่วนผสม, วัตถุดิบ, ผสมอะไร, ถั่ว, กลูเตน, แป้งสาลี, นม, ไข่, กุ้ง, ฮาลาล, มังสวิรัติ, เจ, allergen, ingredient | needs item: `RECIPES[id].allergens`. Always append "ถ้าแพ้รุนแรง กรุณายืนยันกับร้านก่อนสั่ง (ข้อมูลเดโม)". | ราคา, อายุการเก็บ |
| `shelf` (55) | เก็บได้กี่วัน, เก็บได้นาน, อยู่ได้กี่วัน, หมดอายุ, วันหมดอายุ, แช่เย็น, แช่ตู้เย็น, เก็บยังไง, เก็บไว้, ข้ามคืน, อุ่น, อุ่นยังไง, shelf life | needs item: `RECIPES[id].shelf` | ส่วนผสม, สั่งล่วงหน้า |
| `stock` (50) | หมดยัง, หมดไหม, หมดแล้วหรือยัง, เหลือกี่, เหลือเท่าไร, เหลืออีก, เหลือไหม, ยังมีไหม, ยังมีอยู่ไหม, ทันไหม, มีของไหม, สต็อก, sold out, stock | needs item. From `snapshot(id)`: "เหลือ {n} {unitWord} (อัปเดตจากแผนวันนี้)" or "หมดแล้ว ขออภัย ลองสั่งล่วงหน้าสำหรับพรุ่งนี้ได้" or "ใกล้หมด เหลือ {n}" when `remaining <= max(5, 20% of available)` | สั่งล่วงหน้า, ราคา |
| `preorder` (45) | สั่ง, ขอสั่ง, อยากสั่ง, จอง, พรีออเดอร์, preorder, pre-order, ซื้อ, อยากได้, ขอซื้อ, เอา…ชิ้น | starts the flow below | (flow chips) |
| `price` (40) | ราคา, เท่าไหร่, เท่าไร, กี่บาท, กี่บ, กี่ตังค์, ตังค์, ราคาเดลิเวอรี่, price, how much | needs item: "{name} {sellPrice} บาท (หน้าร้าน) · {deliveryPrice} บาทบนแอปเดลิเวอรี่". Values = `state.menuCfg[id]`. | เหลือไหม, สั่งล่วงหน้า |
| `hours` (35) | เปิดกี่โมง, ปิดกี่โมง, เปิดไหม, ปิดไหม, เวลาเปิด, เวลาทำการ, เปิดทำการ, กี่โมง, วันหยุด, หยุดไหม, hours, open | "เปิด {open}-{close} น. หยุด{closedDay} (ข้อมูลเดโม)" | ที่ตั้ง, เมนูวันนี้ |
| `location` (35) | อยู่ไหน, ที่อยู่, พิกัด, แผนที่, ร้านอยู่, ส่งไหม, ส่งถึง, จัดส่ง, เดลิเวอรี่, delivery, ค่าส่ง, ส่งฟรี, แกร็บ, grab, ไลน์แมน, lineman, ช็อปปี้ฟู้ด, รับเอง | "{where} · {pickup}" | เวลาเปิด-ปิด, ราคา |
| `menu` (30) | เมนู, menu, วันนี้มีอะไร, มีอะไรบ้าง, มีอะไรขาย, ขายอะไร, แนะนำ, ของวันนี้, รายการอาหาร, มีขนมอะไร | One line per dish: "{short} {sellPrice}฿ — เหลือ {n}/หมด" | ราคา, สั่งล่วงหน้า, เหลือไหม |
| `thanks` (20) | ขอบคุณ, ขอบใจ, ขอบคุน, thx, thanks, thank, บาย, ไว้เจอกัน | "ยินดีครับ/ค่ะ มีอะไรถามได้อีกนะ" | เมนูวันนี้ |
| `greet` (10) | สวัสดี, หวัดดี, ดีครับ, ดีค่ะ, ดีจ้า, hello, hi, เฮลโล | greeting that names the shop plus the demo notice | เมนูวันนี้, ราคา, สั่งล่วงหน้า, เวลาเปิด-ปิด, คุยกับเจ้าของ |
| `fallback` | (none) | "ขอโทษครับ/ค่ะ ยังไม่เข้าใจ ลองเลือกหัวข้อด้านล่าง หรือพิมพ์สั้น ๆ เช่น "ราคาครัวซองต์" หรือคุยกับเจ้าของร้านได้" | เมนูวันนี้, ราคา, ของเหลือไหม, สั่งล่วงหน้า, เวลาเปิด-ปิด, คุยกับเจ้าของ |

Conflict rule: `preorder` beats `price`/`stock` only if its keyword is the longest match. "ขอสั่งครัวซอง 3 ชิ้น" -> preorder with item and qty prefilled, skip to the next missing slot. "ครัวซองต์เหลือกี่ชิ้น" -> stock/croissant. "หมดยัง" -> stock/all dishes. Note `นม`/`ไข่`/`กุ้ง` also appear in ingredient context; they only route to `allergen` when the better-scoring intent isn't something longer (e.g. "ไข่ไก่ราคาเท่าไหร่" is a price question for no item).

**Pre-order flow (`flow = {step, itemId, qty, time}`):**
1. `item`: ask "จะสั่งเมนูไหนครับ/คะ" with one chip per dish showing remaining (`ครัวซองต์ (เหลือ 12)`); dishes with `remaining==0` are rendered as disabled buttons labelled "(หมด)". Free-text goes through the item alias matcher; no match -> repeat once with chips.
2. `qty`: "กี่{unitWord}ครับ/คะ" with chips 1, 2, 3, 5. Parse an integer 1..remaining. 0 or negative -> "ใส่ตั้งแต่ 1 ขึ้นไปนะครับ/คะ". More than remaining -> "ตอนนี้เหลือ {n} {unitWord} ลดจำนวนหน่อยนะครับ/คะ" (do not advance).
3. `time`: "รับกี่โมงครับ/คะ" with chips computed from `SHOP` hours (the 3 next slots on the hour between open and close, plus "พรุ่งนี้เช้า"). Free text accepted if it contains a digit time (`\d{1,2}[:.]?\d{0,2}`) or a chip value; otherwise re-ask. Store the pickup as a trimmed string (max 20 chars).
4. `confirm`: summary "สรุปออเดอร์: {name} × {qty} {unitWord} = {qty*sellPrice} บาท · รับเวลา {time} · ชำระที่ร้าน" with chips ยืนยัน / แก้ไข / ยกเลิก. ยืนยัน -> `chatRuntime.orders[itemId] += qty`, final "รับออเดอร์เรียบร้อย (เดโม: ยังไม่ได้ส่งออเดอร์จริง)", flow ends, and the snapshot remaining drops (visible on the next stock answer). แก้ไข -> restart at `item`. ยกเลิก -> end the flow.

**Bot data accessor:** `snapshot(id)` returns `{name, short, sellPrice, deliveryPrice, available, soldSim, remaining, status:'ok'|'low'|'out'}` using `derived.menus[id]` (chosen scenario `P + stock`, `soldSim` per decision 8) minus `chatRuntime.orders[id]`. `Chat.onStateChange()` is called by `update()`. If the panel is open, add one muted bot line "ข้อมูลร้านอัปเดตแล้ว" ONLY when `remaining` for some dish changed since the last post (debounced at 800 ms). Nothing else is pushed into the log, so it never spams.

---

## F. Part 4 — AWS architecture section (exact spec)

`<section id="aws" aria-labelledby="awsTitle">` h2 "สถาปัตยกรรมคลาวด์ AWS สำหรับ SME". Intro in plain Thai: "ระบบนี้จ่ายตามที่ใช้จริง ไม่มีเครื่องเปิดทิ้งไว้ ร้านหนึ่งร้านใช้ราวไม่เกิน $0.50 ต่อเดือน (ประมาณการ)". Subparts in this order:

1. **Flow diagram (inline SVG, generated by JS from a `AWS_NODES` array via `createElementNS`, so there is one source of truth).** Two layouts from the same data: wide `viewBox="0 0 960 440"` (>=720px) and tall `viewBox="0 0 360 780"` (<720px, text stays >=12px effective). Toggle with CSS (`.arch-wide` / `.arch-tall` `display:none` by media query). The container has `role="group" aria-label="แผนภาพการทำงานของระบบบน AWS"`, and each SVG gets a `<title>` and `<desc>` with a text summary of the flow.
   - Nodes (id, label, analogy, plain explanation):
     - `user` "มือถือพี่เล็ก / ลูกค้า" — "คนที่ใช้งาน"
     - `line` "LINE OA / เว็บแชท" — "ช่องทางที่ทุกคนใช้อยู่แล้ว" (dashed outline = ระยะที่ 2, labelled "ต่อยอด")
     - `cf` "CloudFront + S3" — "ป้ายหน้าร้านที่เปิดได้ทุกที่"
     - `apigw` "API Gateway" — "เคาน์เตอร์รับเรื่อง"
     - `lambda` "AWS Lambda" — "พนักงานที่มาทำงานเฉพาะตอนมีคนเรียก คิดเงินตามงานที่ทำ"
     - `ddb` "Amazon DynamoDB" — "สมุดจดสูตรและยอดขาย"
     - `bedrock` "Amazon Bedrock (Claude Haiku)" — "ผู้ช่วยอ่านใบเสร็จตลาดสดจากรูปถ่าย" (and, for the chatbot, "ผู้ช่วยเรียบเรียงคำตอบลูกค้า")
   - Each node `<g>` has `role="button"`, `tabindex="0"`, `aria-pressed`, `aria-label="{label}: {analogy}"`, a rounded rect with 1.5px ink stroke (paper fill), a small service icon (`<use href="#i-...">`), the label (bold) and the analogy line under it (wrapped by hand into <=2 `<tspan>` lines of <=18 Thai chars for the wide layout). Hit area >=44px. Selected node: 3px accent stroke + `aria-pressed="true"`.
   - Edges (arrows with `<marker>`): `user->cf`, `user->apigw`, `line->apigw`, `apigw->lambda`, `lambda->ddb`, `lambda->bedrock`. Edge numbers 1..5 are drawn as small circles. **Owner path** edges are solid ink. **Customer chatbot path** (line->apigw->lambda->bedrock+ddb) is accent-colored AND dashed AND tagged with the text label "เส้นทางแชทลูกค้า" in the diagram, plus a legend (solid = "เจ้าของร้านใช้", dashed = "ลูกค้าใช้แชท"), so meaning never relies on color alone.
   - Interaction: `click`, `Enter`, `Space`, and `focus` on a node set `activeNode`; the detail panel `#archDetail` (`aria-live="polite"`) shows: node label, the analogy as a large line, one sentence on what it does for พี่เล็ก, and "ใช้ในขั้นตอนที่ …". Not hover-only (hover may add a light highlight only).
2. **"ข้อมูลไหลยังไง"** — `<ol class="flow-steps">` with 5 `<li>`, each containing a `<button type="button" aria-pressed>`. Clicking/Enter highlights the related nodes/edges in the SVG (`data-nodes`, `data-edges`) and puts the step explanation into `#archDetail`:
   1. "พี่เล็กถ่ายรูปใบเสร็จตลาดสดแล้วส่งเข้า LINE หรือเว็บ" (nodes: user, line, apigw)
   2. "ระบบส่งรูปให้ผู้ช่วยอ่านใบเสร็จ (Bedrock) ดึงชื่อของและราคาออกมา" (apigw, lambda, bedrock)
   3. "ราคาใหม่ถูกจดลงสมุด (DynamoDB) แล้วคำนวณต้นทุนต่อจานใหม่" (lambda, ddb)
   4. "หน้าเว็บ (CloudFront + S3) แสดงแผนว่าวันนี้ควรผลิตกี่ชิ้น" (cf, user, lambda)
   5. "ลูกค้าถามในแชท บอทดูสต็อกและราคาจากสมุดเดียวกัน แล้วตอบ" (line, apigw, lambda, ddb, bedrock). A highlighted tag "แชทลูกค้าอยู่ตรงนี้ และเดโมในหน้านี้จำลองขั้นตอนนี้" plus a button "ลองคุยกับบอท" -> `Chat.open()`.
   Keyboard: the steps list is in the tab order and a node and its step highlight each other (the node click picks the first step that includes it).
3. **Cost table** `<table class="ledger">` caption "ค่าใช้จ่ายต่อเดือนต่อ 1 ร้าน (ประมาณการ ตัวเลขตัวอย่าง)". Columns: บริการ | ทำหน้าที่เหมือน | สัดส่วน | ≈ USD | ≈ บาท. Upper-bound rows at $0.50 and ~36 บาท/USD:
   - CloudFront + S3 15% $0.075 ≈2.7 บาท
   - API Gateway 20% $0.10 ≈3.6
   - Lambda 10% $0.05 ≈1.8
   - DynamoDB 15% $0.075 ≈2.7
   - Bedrock 40% $0.20 ≈7.2
   - Total row: 100% **$0.50 ≈ 18 บาท**
   The share is shown as text plus a thin CSS bar (`aria-hidden`). A note underneath: "ช่วงประมาณการ $0.15-0.50 (≈5-18 บาท) ที่ใช้งานวันละ 20-50 ครั้ง ตามเอกสาร README · ตัวเลขสมมติเพื่อสาธิต ไม่ใช่ใบแจ้งหนี้ AWS · เรทสมมติ 36 บาท/ดอลลาร์ · ยังไม่รวมแชทลูกค้า ถ้าลูกค้าถามเยอะ ค่า Bedrock จะเพิ่มตามจำนวนข้อความ".
4. **"จะขยายต่อยังไง"** — `<ol class="roadmap">`, a horizontal thin rule with 3 markers on desktop and a vertical rule on mobile: (1) "ระยะที่ 1 ต้นแบบ (ตอนนี้): CloudFront + S3, ทดลองกับร้านอาหาร 10 ร้านในชุมชน", (2) "ระยะที่ 2 LINE OA + Bedrock อ่านใบเสร็จ: ส่งรูปบิลผ่าน LINE ระบบคำนวณตอบกลับ และแชทตอบลูกค้า", (3) "ระยะที่ 3 เทียบราคาตลาดร่วมกัน: แจ้งเตือนล่วงหน้าเมื่อของแนวโน้มแพงขึ้น". Phase 1 has the text "ตอนนี้".
5. The old three bullets ("Zero Idle Cost" etc.) are folded into the Lambda/DynamoDB node text. Keep a single line "ข้อมูลแต่ละร้านแยกกัน" (still true per README tenant isolation) under the diagram. The old ASCII `.aws-arch-box` is deleted.

---

## G. Part 2 — visual redesign checklist

- Remove: gradients, `box-shadow` cards, rounded white uniform cards, emoji (including in `<title>`, footer and `<option>` text), the "AWS Serverless Ready" and "Real-time Recalc Active" badges, `outline:none`, and the `.aws-arch-box`.
- Masthead: `<header>` left aligned. Wordmark "SmartBatch" in 800 weight with a small SVG mark (`i-receipt`), the Thai tagline "วางแผนผลิตวันนี้ ใน 3 นาที" as the `<h1>`, one sentence under it: "ใส่ราคาของวันนี้กับยอดที่คาดไว้ ระบบบอกว่าควรทำกี่ชิ้น และกำไรเสี่ยงแค่ไหน". `<nav aria-label="เมนูหลัก">` with 3 links: วางแผน (#step1), สถาปัตยกรรม (#aws), ลองแชท (button). Skip link "ข้ามไปเนื้อหาหลัก" (first focusable).
- Disclaimer (kept honest, quiet): a bordered note styled like the foot of a receipt, a dashed top rule and small text: "หมายเหตุ: เครื่องมือช่วยตัดสินใจต้นแบบ ตัวเลขราคาและยอดขายทั้งหมดเป็นตัวอย่างเพื่อสาธิต ไม่ใช่ข้อมูลตลาดหรือยอดขายจริง ไม่ใช่คำทำนาย". Keep the Zero-History explanation in one short sentence: "ร้านที่ยังไม่มีประวัติยอดขายก็ใช้ได้ เพราะคำนวณจากสิ่งที่พี่รู้ตอนเช้า".
- Tables: top and bottom 1px ink rules, 1px `--rule` row lines, numeric columns right-aligned with tabular nums, no zebra and no cell borders. Section headings use a large step numeral in `--accent`. Spacing scale 4/8/12/16/24/40/64. Content max width 1120px.
- Controls: min height 44px (steppers/radios/chips 48px); buttons = ink fill with paper text, secondary = 2px ink outline; hover = `--paper-2` fill; `:active` translateY(1px); `:focus-visible` rule from decision 9 applied to buttons, inputs, summary, links and the SVG `g[role=button]`.
- Motion: only transform/opacity/background transitions of <=150 ms (button press, chat open, node highlight, a 600 ms underline flash on the headline number when it changes). `@media (prefers-reduced-motion: reduce){ *,*::before,*::after{animation:none!important;transition:none!important;scroll-behavior:auto!important} }`.
- Responsive: designed at 360px first. No horizontal scroll at 360px (`document.documentElement.scrollWidth <= innerWidth`). Breakpoints: 720px (tables un-stack, SVG wide/tall switch), 960px (two-column rail layout). Add `<meta name="theme-color" content="#F6F0E4">` and `<meta name="color-scheme" content="light">`.
- Contrast pairs to verify numerically (>=4.5:1 body, >=3:1 large/UI): ink/paper, ink-2/paper, accent/paper, paper/accent (text on accent buttons), paper/ink (FAB, headline block), ok/paper, warn/paper, ink/paper-2, ink-2/paper-2, accent/paper-2.
- Copy: short and concrete. No "ยอดเยี่ยม", "!" marketing, or "AI-powered" fluff. Remove the English jargon in labels (Yield Loss, Quantize, Walk-in, Pre-orders) and use the Thai equivalents above; keep English only in service names and the formula note in README.

---

## H. Ordered work items

Because everything is one file, intermediate states only need to open without console errors. Do the items in order.

- [ ] 1. Snapshot the original for reference: copy `index.html` to `.agents/tasks/index.original.html` (outside the shipped file).
      Files: `.agents/tasks/index.original.html`
      Verify: the copy exists and is 806 lines; `index.html` is untouched.

- [ ] 2. Write the pure logic first: `RECIPES`, `SHOP`, `BENCHMARK`, the `Core` block (sanitize, targetProduction, rawCost, unitCost, margins, marginStatus, scenario, derive), and the `BotNLU` block (normalize, keyword table, classify, qty parser), both between the `CORE:BEGIN/END` and `BOT:BEGIN/END` markers, in a new `<script>` section of `index.html` (the old UI code can stay for now).
      Files: `index.html`
      Verify: create the temporary script `.agents/tasks/verify/unit.mjs` that reads `index.html`, extracts the two marked blocks with a regex, runs them in `node:vm`, and asserts: (a) `targetProduction(20,30,5,24) === 48`, (b) the section C hand-computed croissant values for A/B/C (unitCost 19.96, P 24/48/48, profit 1115.96/1296.92, atRisk 239.52, lost 12, breakEven 18), (c) the edge cases (0, 24, 24), (d) pork_rice store margin 36.0% and status `low`, (e) `sanitize('')`, `sanitize('abc')`, `sanitize(-5)` (value 0), `sanitize(1e9)` (clamped), `sanitize('12.7',{int:true})` (13) all return finite numbers and `ok:false` where expected, (f) BotNLU: "ครัวซองต์ราคาเท่าไหร่" -> price/croissant, "ขอสั่งครัวซอง 3 ชิ้น" -> preorder/croissant/qty 3, "หมดยัง" -> stock/null, "เปิดกี่โมงครับบบ" -> hours, "ขอคุยกับเจ้าของร้าน" -> human, "แพ้ถั่วไหม" -> allergen, "เก็บได้กี่วัน" -> shelf, "ขอบคุณค่ะ" -> thanks, "สวัสดีครับ" -> greet, "<img src=x onerror=alert(1)>" -> fallback, "asdfgh" -> fallback. Run `node .agents/tasks/verify/unit.mjs`. Expected: all assertions pass, exit code 0.

- [ ] 3. Replace `<head>` and the CSS with the new design system, add the SVG icon sprite, the skip link, the masthead/nav, the disclaimer note, and the sticky rail/strip shell. Delete the old gradient/badge/card CSS. Then build Step 1 and Step 2 markup and their JS: `state`, `load/save`, `update()`, the stepper/range/radio/preset/reset bindings, validation messages, `renderStep1/2`, and the sticky headline. Remove the old `calculateAll/applyPreset/changeMenu` and every inline `on*=` handler for these sections.
      Files: `index.html`
      Verify: `node .agents/tasks/verify/unit.mjs` still passes. Open the page in headless Edge (see section I) and check there are no console errors, the price steppers change `#chg-lime` text, an empty/"abc"/-5 entry shows the Thai message with finite output, and a reload restores the edited values from `sb.state.v2`.

- [ ] 4. Build Step 3: `#headlineAnswer`, `#marginBanner`, the scenario comparison table with the choose-plan buttons, the per-dish margin ledger, the `#whyDetails` explanation with substituted numbers, and the waste block with `sb.wasteLog.v1` logging (no `alert`). Remove the old ROW 1-3 markup and its leftover script.
      Files: `index.html`
      Verify: in headless Edge set croissant, pre=20, walk=30, stock=5, batch=24 and confirm `#headlineAnswer` text contains "ผลิต 48 ชิ้น = 2 ถาด" and `#whyDetails` text contains `ceil((20 + 18 − 5) ÷ 24)`; select plan A and confirm "24 ชิ้น = 1 ถาด"; switch to pork_rice at defaults and confirm the margin banner shows "ต่ำกว่าเป้า"; set the target to 30 and confirm it changes to ok; click "บันทึกของเหลือวันนี้" and confirm `#wasteStatus` is populated and `sb.wasteLog.v1` has 1 entry.

- [ ] 5. Build the AWS section: JS-generated SVG (wide and tall), node and step interactions, `#archDetail`, cost table, roadmap strip. Remove the old ASCII box.
      Files: `index.html`
      Verify: headless Edge at 1280px and 360px: both SVGs exist, exactly one is visible, 7 nodes have `role=button` and `tabindex=0`, dispatching a `keydown` Enter on the Lambda node sets `aria-pressed="true"` and changes `#archDetail` text to the Lambda analogy, clicking step 5 highlights the 5 listed nodes, the cost-table total row reads `$0.50` and `18`, and `scrollWidth <= 360` at 360px.

- [ ] 6. Build the chatbot: widget markup/CSS, `Chat` module (open/close/focus, log, typing, chips, NLU wiring, the pre-order state machine, snapshot, progress slider, `onStateChange`), the entry buttons in the masthead and AWS step 5.
      Files: `index.html`
      Verify: `node .agents/tasks/verify/unit.mjs` still passes (BotNLU unchanged). In headless Edge: open the chat via `#chatFab` and confirm focus is on `#chatInput` and `#chatPanel` is not hidden; press Esc and confirm hidden and focus back on `#chatFab`; ask "ราคาครัวซองต์" and confirm the reply contains the sellPrice currently in the state (change `#in-sell` to 60, ask again, the answer now says 60); change `in-stock` and `in-pre` and confirm "ครัวซองต์เหลือกี่ชิ้น" returns the value computed by the formula in section E; run the full pre-order flow (สั่ง -> ครัวซองต์ -> 3 -> chip time -> ยืนยัน) and confirm the summary text, then the next stock answer is 3 lower; the HTML-injection test in section I passes.

- [ ] 7. Update `README.md` section 5 (exact edit in section J).
      Files: `README.md`
      Verify: open the file and confirm only section 5 changed; the diff against the original is limited to the added lines (compare with a before/after `Get-Content` line count: the new file is longer only by the added lines).

- [ ] 8. Final cross-check and cleanup, then write the review artifacts expected by the loop. Run the full verification in section I, fix anything it exposes, and delete the temporary files in `.agents/tasks/verify/`. Keep `.agents/tasks/index.original.html` and `plan.md`.
      Files: `index.html`
      Verify: every check in section I is green.

---

## I. Verification (run after items 2, 3-6 as listed and in full at item 8)

1. **Formula sanity (unit, not shipped):** `node .agents/tasks/verify/unit.mjs`. It must show PreOrders=20, WalkIn=30, Stock=5, BatchSize=24 -> `ceil(45/24)=2` -> **48**, plus all other assertions in item 2.
2. **Browser harness (temporary, deleted at the end):** `.agents/tasks/verify/cdp.mjs` starts Edge headless with `--headless=new --remote-debugging-port=9333 --user-data-dir=<temp dir> file:///C:/Users/User/OneDrive/Desktop/project_AWS/project%20AWS/index.html`, connects to the DevTools websocket using Node's built-in global `WebSocket` (no npm), and uses `Runtime.evaluate`, `Runtime.consoleAPICalled`/`Runtime.exceptionThrown` (collect errors), `Emulation.setDeviceMetricsOverride` (360x740 and 1280x800), and `Page.captureScreenshot` into the temp folder (view them with the image reader, then delete). If CDP cannot be made to work within 2 attempts, fall back to `msedge --headless=new --dump-dom` plus the Node vm unit checks, and say so in the final notes. Do not retry endlessly.
3. **Console:** no errors or warnings at load, after the actions above, at both viewport sizes.
4. **HTML-injection test:** open the chat and send these exact strings, one at a time: `<img src=x onerror="window.__pwn=1">`, `<script>window.__pwn=2</script>`, `"><svg onload=window.__pwn=3>`. Assert: `window.__pwn === undefined`; `#chatLog.querySelector('img,script,svg')` is null (the SVG sprite is outside the log); the user bubble `textContent` equals the typed string exactly; the bot answer is the fallback with chips. Also assert statically that the chat code has no `innerHTML`, `insertAdjacentHTML` or `outerHTML` (the file may keep `innerHTML` nowhere at all; if any remains it must be a constant template, with no variable interpolation, and the chat module must have none). Repeat the injection in the pre-order time step (`<b>x</b>`) and confirm it appears as literal text in the summary. Corrupt storage test: set `localStorage['sb.state.v2']='{"prices":{"lime":"<b>"},"menuId":"zzz"}'`, reload, and confirm the defaults load with no NaN on screen and no console error.
5. **Responsive/overflow:** at 360px width `document.documentElement.scrollWidth <= 360`; every `button, input, select, summary, [role=button]` that is visible has `getBoundingClientRect().height >= 44` (SVG nodes: >=44 hit height in the tall layout). At 1280px the two-column rail is visible and sticky.
6. **Accessibility spot checks (DOM):** every `input/select` has a matching `<label for>` or `aria-label`; exactly one `<h1>`; headings do not skip levels; `lang="th"`; `#chatLog` has `role=log` and `aria-live`; `#headlineAnswer` has `aria-live`; focus order reaches the skip link, nav, steppers, chips and the chat; `:focus-visible` outline is computed non-`none` on a button, an input, a `summary` and an SVG node. State: full WCAG validation needs manual assistive-technology testing, which this plan does not cover.
7. **Contrast:** add to `unit.mjs` a small WCAG luminance function and assert >=4.5 for the text pairs listed in section G (and >=3 for the UI-only pairs).
8. **Reduced motion:** with `Emulation.setEmulatedMedia` `prefers-reduced-motion: reduce`, assert that the computed `transition-duration` of `#chatPanel` and a `.btn` is `0s` and that the typing delay path uses the short branch (reply appears within 400 ms).
9. **Style regression scan (static, supplementary only):** the file has no `linear-gradient(`, `backdrop-filter`, `#2563eb`, `✨`, or any character in the ranges U+1F300-1FAFF / U+2600-27BF / U+2B50 (emoji). The only network reference is the Google Fonts `<link>`; no `fetch(`, `XMLHttpRequest`, `<script src`, or API keys. Offline degradation is checked by loading with the network blocked (Edge flag `--host-resolver-rules="MAP fonts.googleapis.com ~NOTFOUND, MAP fonts.gstatic.com ~NOTFOUND"`) and confirming the page renders with system fonts and no layout overflow at 360px.
10. **Persistence:** change a price, plan, menu and scenario; reload; confirm all four restored. Make `localStorage.setItem` throw (override `Storage.prototype.setItem`) and confirm there are no uncaught errors and the calculator still works.
11. **Cleanup:** delete `.agents/tasks/verify/` and any Edge temp profile/screenshots. Leave `index.html`, `README.md`, `.agents/tasks/plan.md`, `.agents/tasks/index.original.html`.

---

## J. README.md — exact section 5 edit

Change nothing outside section 5. In section "## 5. รายละเอียดไฟล์ต้นแบบ (Prototype Files)", leave the existing bullets and numbered items 1-4 as they are, and insert this block directly after item 4 (the line ending "...โดยไม่ต้องติดตั้ง Library เพิ่มเติม") and before the `---` that precedes section 6:

```markdown
  5. **อัปเดตล่าสุดของต้นแบบ** (ยังเป็นไฟล์เดียว เปิดได้ด้วยการดับเบิลคลิก ไม่ต้องมี Backend หรือ API Key):
     * **แชทบอทลูกค้า (เดโมจำลอง):** ปุ่ม "ถามร้าน" มุมขวาล่าง ตอบแบบจับคำสำคัญภาษาไทย (เมนูวันนี้ ราคา ของเหลือเท่าไร สั่งล่วงหน้า เวลาเปิด-ปิด ที่ตั้ง/เดลิเวอรี่ ส่วนผสมที่อาจแพ้ อายุการเก็บ คุยกับเจ้าของ) ดึงราคากับจำนวนคงเหลือจากตัวเลขในเครื่องคำนวณเดียวกัน แก้แผนผลิตแล้วคำตอบเปลี่ยนตาม ของจริงจะใช้ Amazon Bedrock ผ่าน Lambda + LINE OA
     * **เครื่องคำนวณ 3 ขั้นตอน:** (1) วันนี้ของแพงขึ้นไหม (2) วันนี้คาดว่าขายได้เท่าไร (3) ผลลัพธ์ที่บอกเป็นประโยคเดียว เช่น "ผลิต 48 ชิ้น = 2 ถาด" เทียบ 3 แผนด้วยกำไรที่คาดและเงินจมถ้าขายไม่หมด พร้อมส่วน "ทำไมถึงได้ตัวเลขนี้" ที่แทนค่าในสูตรด้วยตัวเลขของผู้ใช้เอง และเตือนเมื่อกำไรต่อจานต่ำกว่าเป้า (ค่าเริ่มต้น 40%) ข้อมูลที่กรอกถูกจำไว้ในเบราว์เซอร์ของเครื่องนั้น
     * **ส่วนสถาปัตยกรรม AWS แบบเข้าใจง่าย:** แผนภาพ SVG พร้อมคำเปรียบเทียบของแต่ละบริการ ลำดับ "ข้อมูลไหลยังไง" 5 ขั้น (รวมจุดที่แชทลูกค้าจะอยู่) ตารางประมาณค่าใช้จ่ายต่อเดือนต่อร้านที่ไม่เกิน $0.50 และแผนต่อยอด 3 ระยะ ตัวเลขทั้งหมดเป็นตัวอย่างเพื่อสาธิต
```

Verification of this item is in item 7 of section H.

---

## K. Assumptions and gaps

- The README says the Bedrock model is "Claude 3.5 Haiku" in section 4 and "Claude Haiku" in the task; the UI says "Claude Haiku" to match the task and avoid pinning a version. README section 4 is not edited.
- The README sets the break-even/spoilage cost as `leftover * unitCost`; this plan keeps that definition for `atRisk` and `wasteCost`.
- The cost-share percentages (15/20/10/15/40) are an illustrative split chosen to sum to the README's $0.50 upper bound; they are labelled as an estimate in the UI.
- The shop name, hours, address and contact in `SHOP` are made-up demo data; no real PII is used.
- Orders placed in the chat reduce only the bot's own remaining count; they do not change the calculator inputs (deliberately out of scope).
- The workspace is not a git repo, so there is nothing to commit; the review loop should diff against `.agents/tasks/index.original.html`.
