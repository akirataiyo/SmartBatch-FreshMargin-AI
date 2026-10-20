# Verification record (iteration 1)

Files changed: `index.html` (rewritten, still one file, inline CSS+JS, about 1,820 lines), `README.md` section 5 only (one added item "5. อัปเดตล่าสุดของต้นแบบ" with 3 sub-bullets). `.agents/tasks/index.original.html` is a copy of the original for diffing. Nothing committed (workspace is not a git repo).

Tooling: Microsoft Edge (headless, `--headless=new`) driven over the DevTools protocol with Node 24's built-in WebSocket (no npm packages). Scripts and screenshots lived in `%TEMP%\sbverify` (outside the project) and were deleted at the end. I ran the whole suite twice; the numbers below are from the final run, after the fixes listed at the bottom.

## 1. Formula sanity (not shipped in the UI)
`node unit.mjs index.html` extracts the `CORE:` and `BOT:` blocks and runs them in `node:vm`. Result: **ALL 11 GROUPS PASSED**.
- PreOrders=20, WalkIn=30, Stock=5, Batch=24: ceil(45/24)=2, so **48**. Pass.
- Edges: (0,0,10,24) gives 0; (24,0,0,24) gives 24; (1,0,0,24) gives 24. Pass.
- Croissant at benchmark prices: unitCost 19.96; plans A/B/C give P = 24/48/48; profit A 1115.96, B 1296.92; atRisk B 239.52; lost A 12; break-even B 18 units. All match the hand calculation in plan.md.
- pork_rice: raw 31.6, unitCost 41.6, store margin 36.0%, status `low` at target 40.
- `sanitize('')`, `'abc'`, `-5` (becomes 0), `1e9` (clamped), `'12.7'` int (13), `Infinity`, `null`, `undefined`, `'NaN'`: every result is finite, `ok:false` for empty/non-numeric.
- BotNLU: 29 phrases classified as expected (price/croissant, preorder/croissant/qty 3, stock, hours, human, allergen, shelf, thanks, greet, menu, location, fallback for `<img src=x onerror=alert(1)>` and `asdfgh`, and every quick-reply chip label maps back to its intended intent).
- WCAG contrast (computed): ink/paper 14.6, ink-2/paper 6.75, accent/paper 5.27, paper/accent 5.27, white/accent 5.98, ok/paper 5.61, warn/paper 5.22, ink-2/paper-2 6.01, accent/paper-2 4.69, light text on ink 10.5. All text pairs are at least 4.5:1. `--rule` (1.54:1) is used only for decorative row lines, never for text or for a control boundary (controls use 2px ink borders).

The same value was also read from the running page: `Core.targetProduction(20,30,5,24)` evaluated in the page gives **48**. With croissant, pre=20, walk=30, stock=5, batch=24 typed into the real fields, `#headlineAnswer` reads "ผลิต 48 ชิ้น = 2 ถาด" and `#whyBody` shows `= ceil((20 + 18 − 5) ÷ 24) × 24 = ceil(1.375) × 24 = 2 × 24 = 48`. Selecting plan A gives "ผลิต 24 ชิ้น = 1 ถาด".

## 2. Screenshots (375 and 1280 wide, plus 360 and keyboard-focus shots), all viewed
Captured: top of page, step 1, step 2, step 3 (result), AWS section, chat open, at 375px and 1280px; plus the AWS diagram at 360px (2x), margin table, AWS step highlight, keyboard-focus state, chat via keyboard, and the 360px page with font hosts blocked.
What I saw and fixed:
- 375: left-aligned masthead, sticky strip "1 ราคา 2 ยอดขาย 3 ผลลัพธ์ + ผลิต N จาน", stacked price rows with a 3.50 steppers and slider, ticket-style result. Found "3 / นาที" breaking onto its own line in the h1, so I made "ใน 3 นาที" non-breaking. The margin rows showed a doubled red left bar; fixed. The plan-comparison "ผลิต" cell wrapped awkwardly; now "48 จาน" with "(2 รอบ)" on a second line.
- 375/1280: a stray red dashed fragment at the bottom edge of the page. The cause was the legend's `.chat` class colliding with the chat panel's `.chat` style (fixed-position). Renamed the panel class to `.chatbox`. Gone.
- Both diagrams (wide and tall) were showing at the same time because `.arch svg.arch-svg {display:block}` out-ranked the `display:none` rule. Fixed with higher-specificity selectors. Now 1280 shows only the wide diagram (1088px) and 360/375 only the tall one (328px).
- Diagram node text ran past the bottom edge of the LINE OA, Lambda and Bedrock boxes. Moved the text up and tightened line spacing; the DOM check (text bounding box vs box) reports no overflow in either layout.
- Desktop margin table header columns ran together; added padding.
- Mobile cost table: the total row had an empty row and the share % column was hidden; now the share % shows (bar hidden on mobile).
- Final look: warm paper background, ink text, one chili-red accent, ledger tables with thin rules, ticket-style result box, no gradients, no shadows, no emoji, inline-SVG icons.

## 3. Console
Zero `console.*` calls, zero uncaught exceptions at load, on interaction, in both viewport sizes, and in the chat, persistence and corrupt-storage tests. Two things appear in the log channel, neither a page defect:
- Offline run only: `Failed to load resource: net::ERR_NAME_NOT_RESOLVED` for the Google Fonts stylesheet (expected when the host is blocked). The page still renders with system Thai fonts, `scrollWidth` 360, headline correct.
- A browser warning `The specified value "abc" cannot be parsed` appeared once, because my test script wrote "abc" into a number field programmatically. A user cannot type that into a `type=number` field in Edge; the app handled it (message shown, last valid value kept).

## 4. Chatbot drive (bot answers; shop numbers come from the same state as the calculator)
Opening via the FAB: panel visible, focus lands on `#chatInput`, `aria-expanded=true`. Greeting names the shop and says it is a demo. Results:
- "เมนูวันนี้": "ข้าวหมูกรอบ 65 ฿ — เหลือ 12 จาน / ครัวซองต์ 55 ฿ — เหลือ 21 ชิ้น / ต้มยำกุ้ง 150 ฿ — เหลือ 4 ถุง" (chips: ราคา, สั่งล่วงหน้า, เหลือไหม).
- "ราคาเท่าไร": lists all three dishes, store and delivery prices.
- "ของหมดยัง": per dish "เหลือ 12 จาน", "เหลือ 21 ชิ้น", "ต้มยำกุ้ง ใกล้หมดแล้ว เหลือ 4 ถุง".
- Also checked: ราคาครัวซองต์, ครัวซองต์เหลือกี่ชิ้น, เปิดกี่โมง, ร้านอยู่ไหน ส่งไหม, แพ้ถั่วไหม (allergen list plus "ยืนยันกับร้านก่อนสั่ง"), เก็บได้กี่วัน, ขอบคุณค่ะ, อยากคุยกับเจ้าของร้าน (hand-off message marked demo). All routed correctly.
- Unknown message "พรุ่งนี้ฝนจะตกไหม": polite fallback plus 6 quick-reply chips.
- **Calculator changes the bot's answer:** "ครัวซองต์เหลือกี่ชิ้น" gave 21. After setting the calculator's leftover-stock field to 25 (plan changes from 48 to 24 pieces) it answered 17; after changing the sell price field to 60, "ราคาครัวซองต์" answered 60 บาท (was 55). A muted "ข้อมูลร้านอัปเดตแล้ว" line appeared in the chat.
- **Pre-order flow end to end:** สั่ง, then chips for the three dishes with remaining counts; "ครัวซองต์", then quantity chips 1/2/3/5; "3", then time chips; "10:30", then summary "สรุปออเดอร์: ครัวซองต์เนยสดพรีเมียม × 3 ชิ้น = 180 บาท · รับเวลา 10:30 · ชำระที่ร้าน" with ยืนยัน/แก้ไข/ยกเลิก; "ยืนยัน", then "รับออเดอร์เรียบร้อยค่ะ … (เดโม: ยังไม่ได้ส่งออเดอร์จริง)". The next stock answer dropped from 17 to 14. Also: "ขอสั่งครัวซอง 3 ชิ้น" prefills item and quantity and skips to the time step; "0", "999", "2" at the time step are rejected with the opening-hours message; "ไม่รู้" re-asks; แก้ไข restarts at the item step; ยกเลิก ends the flow.
- **HTML injection (all four strings rendered as literal text):** `<img src=x onerror=alert(1)>`, `<img src=x onerror="window.__pwn=1">`, `<script>window.__pwn=2</script>`, `"><svg onload=window.__pwn=3>`. For each: the user bubble's `textContent` equals the typed string exactly; `#chatLog` contains no `img`, `script`, `svg` or `b` element; `typeof window.__pwn` is `undefined`; no JavaScript dialog opened; the bot replied with the fallback and chips. In the pre-order time step, `<b>x</b> 11:00` appears literally in the summary text. Static check: the file contains none of `innerHTML`, `insertAdjacentHTML`, `outerHTML`, `document.write`; and no inline `on*=` attributes.
- Esc closes the panel and focus returns to the opener (FAB, or the "ลองคุยกับบอท" button when it opened the chat).

## 5. Calculator behaviour (real fields, 1280)
- pork_rice defaults: banner "ต่ำกว่าเป้า: ข้าวหมูกรอบ เหลือกำไร 36.0% ต่ำกว่าเป้า 40% ลองขึ้นราคาเป็น 70 บาท …" (red band with warning icon and text). Target set to 30: changes to "เฝ้าระวัง" (amber, eye icon). Per-dish ledger marks pork_rice with an icon, the text "ต่ำกว่าเป้า 40% (หน้าร้านและเดลิเวอรี่)", a bold row and a left rule.
- Validation: empty gives the message "กรอกเป็นตัวเลขนะ ตอนนี้ใช้ค่าเดิมให้ก่อน" and the headline keeps the last valid value; -5 gives "ใส่ค่าติดลบไม่ได้ ปรับเป็น 0 ให้แล้ว" and 0 is used; 99999 is clamped to 999 with a message; "abc" in a price field is refused. No NaN or Infinity shown anywhere.
- Price stepper: lime 3.50 +0.50 gives 4.00 and "แพงขึ้น 14% (+0.50 ฿)"; preset มะนาวแพง gives 9.00, "แพงขึ้น 157%", `aria-pressed=true`; "ใช้ราคาอ้างอิงทั้งหมด" returns 3.50 and "เท่าอ้างอิง".
- Waste: 4 leftover croissants = 79.8 ฿, advice "มีของเหลือ รอบหน้าลองผลิตลง 1 รอบ", log button writes `sb.wasteLog.v1` and shows "บันทึกแล้ว … สะสม 1 ครั้ง" (no `alert`).
- Plan table (croissant 20/30/5/24): A 24 (1 ถาด), B 48 (2 ถาด), C 48. Profit 1,116 / 1,297 / 1,297 ฿; money at risk 0 / 240 / 240 ฿; missed sales 12 / 0 / 0; break-even 9 / 18 / 18. Note shown: "แผน C ผลิตเท่าแผน B เพราะปัดเป็นถาดแล้วเท่ากัน".

## 6. Persistence
Changed pork price to 210, menu to ต้มยำกุ้ง, pre-orders to 17, plan to C; after reload all four restored, and "ครั้งก่อน" for pork now shows 210. Corrupt storage (`prices.lime="<b>"`, `menuId:"zzz"`, `batch:7`, `scenario:"Q"`, `gpPct:-3`) loads defaults with no NaN text and no console output; invalid JSON also loads defaults silently. With `Storage.prototype.setItem` throwing, the calculator still works, no uncaught error, and the waste button shows a friendly "บันทึกในเครื่องนี้ไม่ได้" message.

## 7. AWS section
Only one diagram is visible per width (wide at 1280, tall at 360/375). 7 nodes have `role=button tabindex=0`. Focusing the Lambda node and pressing Enter sets `aria-pressed=true` and `#archDetail` shows "พนักงานที่มาทำงานเฉพาะตอนมีคนเรียก คิดเงินตามงานที่ทำ …ใช้ในขั้นตอนที่ 2, 3, 4, 5". Clicking step 5 dims `user` and `cf`, leaves the five related nodes lit, shows the "แชทลูกค้าอยู่ตรงนี้…" tag and a "ลองคุยกับบอท" button that opens the chat. Cost table total row reads 100% / $0.50 / 18 and is labelled as an estimate. Roadmap shows three phases with "ตอนนี้" on phase 1.

## 8. Keyboard, responsive, accessibility
- Real key events: Tab order starts at the skip link, then nav, hero buttons, the rail links, then each price stepper (minus, number, plus, slider). 24 consecutive tab stops all had a visible 3px outline (`:focus-visible` true). Computed outline also checked on a `summary`, a button, a number input and an SVG node (3px solid ink).
- Keyboard-only chat: Tab from the last footer button lands on the FAB, Enter opens the chat with focus in the input, typing plus Enter sends and gets the price reply, Tab reaches the send button, Esc closes and focus returns to the FAB.
- 360px: `scrollWidth` 360 (no horizontal scroll); 375px: 375. 1280px: two-column layout, sticky rail (`position: sticky`, columns 250px + 782px).
- No visible button, input, select, summary, link or SVG node is under 44px high at 360 or 1280 (radio segments are 48px).
- `lang="th"`, one `h1`, headings in order, every input has a label or `aria-label`, no unnamed buttons, no duplicate ids, `role=log` + `aria-live` on the chat log, `aria-live` on the headline.
- Reduced motion (emulated): `.btn` transition 0s, chat open animation none, typing dots animation none, scroll-behavior auto; the typing delay uses the 150ms branch (reply observed in about 440ms including DevTools polling overhead).
- Static scan: no `linear-gradient`, `backdrop-filter`, `#2563eb`, `✨` or emoji characters; no `fetch`/`XMLHttpRequest`/`<script src>`. The only network references are the Google Fonts `<link>`s; the page degrades to system Thai fonts when blocked. The one `outline: none` left is `.node:focus:not(:focus-visible)` (mouse focus on SVG nodes only).

## Not verified / limits
- Not tested on a real phone or real screen reader; WCAG conformance needs manual testing with assistive technology and an expert review. Only Edge (Chromium) was tested; Firefox/Safari were not.
- The "simulated sold" number in the chat is a deterministic estimate (pre-orders + walk-in × slider), not real sales.
- Chat order confirmations reduce only the bot's own remaining count; they do not change the calculator inputs.
- Shop name, hours and address in the demo are made-up; cost shares per service (15/20/10/15/40%) are an illustrative split of the README's $0.50 upper bound.
- Product decision to flag: the default menu is pork_rice, which is below the 40% target at benchmark prices (36.0%), so the page opens with a red warning. The data is unchanged from the original; say if you would rather open on a passing dish.


---

# Iteration 2 (review fixes)

Browser: Microsoft Edge headless (`--headless=new`) driven over the DevTools protocol with Node 24 (no npm packages). Scripts lived in `%TEMP%\sbverify2` (outside the project) and were deleted. Screenshots are kept in `.agents/tasks/screenshots/` (375-top, 375-result, 375-aws, 375-chat-open, 1280-top, 1280-result, 1280-aws, 1280-chat-open, keyboard-focus). I viewed 375-chat-open and 1280-chat-open: no overflow or overlap, Thai wraps correctly, demo/Bedrock label visible, chips 44px+, the chat sits over the page at bottom right on desktop and full width on 375. I did not re-view the other shots in this iteration (layout code untouched; only chat logic and one chip CSS rule changed).

## Fixes
1. Order confirm (blocking): negation (`/ไม|^no(pe)?$/` on the normalized text) is tested first; confirmation now needs the WHOLE normalized answer (after stripping trailing particles คะ/ครับ/นะ/เลย) to equal one of ยืนยัน, ตกลง, โอเค, ok, okay, ใช่, yes, y, ได้, เอา. The edit (แก้ไข) match is also skipped when negated.
2. Focus after chip (blocking): busy chips now get `aria-disabled="true"` instead of `disabled`, so the focused chip is not blurred while the bot "types". After the reply, if the send came from a chip and focus was on a chip or lost to body, focus moves to the first enabled new chip (else the input). Added CSS for `.chip[aria-disabled="true"]`.
3. Time step (non-blocking): the input is parsed as H, H:MM or HMM and compared in minutes against SHOP.open (06:30) and SHOP.close (16:00); minutes >= 60 are rejected. "พรุ่งนี้ 05:00" is also range-checked, "พรุ่งนี้"/"เช้า" with no digits is still accepted.
4. index.original.html kept as the only backup until the user accepts.
5. Screenshots retained (see above).

## Results (real page, Edge)
- Formula in page: `Core.targetProduction(20,30,5,24)` = **48**. Edges (0,0,10,24)=0, (24,0,0,24)=24, (1,0,0,24)=24.
- Console: zero console messages, zero exceptions, zero log-channel entries across the whole run (load at 375 and 1280, chat, order flow, keyboard). No JavaScript dialog opened.
- scrollWidth 375 at 375px (no horizontal scroll); 1265 at 1280 (viewport less scrollbar).
- Chat: "เมนูวันนี้" -> ข้าวหมูกรอบ 65 ฿ เหลือ 12 / ครัวซองต์ 55 ฿ เหลือ 21 / ต้มยำกุ้ง 150 ฿ เหลือ 4. "ราคาเท่าไร" -> three dishes with store and delivery prices. "ของหมดยัง" -> per-dish remaining. "พรุ่งนี้ฝนจะตกไหม" -> fallback plus chips. `<img src=x onerror=alert(1)>` -> fallback; user bubble textContent equals the typed string; no img/script in #chatLog; `window.__pwn` undefined; no dialog.
- Pre-order: "ขอสั่งครัวซอง 2 ชิ้น" -> asks time. "6" and "16:45" -> rejected with the opening-hours message. "10:30" -> summary 110 บาท. **"ไม่ใช่" -> "กด ยืนยัน แก้ไข หรือยกเลิก ได้เลยค่ะ" (not confirmed). "ไม่ยืนยัน" -> same prompt**; chips still ยืนยัน/แก้ไข/ยกเลิก. "ยืนยัน" -> "รับออเดอร์เรียบร้อยค่ะ ครัวซองต์ × 2 ชิ้น รับเวลา 10:30". "16:00" accepted; "ok" confirms.
- More negation/yes variants at the summary: "ไม่ยืนยันค่ะ", "ไม่เอา", "no", "ไม่", "books ok maybe" all give the prompt (no order); "ใช่ค่ะ" confirms.
- Calculator link (unchanged this iteration, re-checked baseline): bot croissant remaining 21 from calculator state; earlier iteration showed 21 -> 17 after changing leftover stock and 55 -> 60 after changing the sell price. Not re-run in iteration 2 beyond the baseline 21.
- Keyboard-only (real key events): Enter on the FAB opens chat with focus in #chatInput. Shift+Tab from the input reaches the "ปรับเดโม" summary then the chips (all focus-visible, 3px outline). Enter on chip "สั่งล่วงหน้า" -> focus stays on a button chip during the wait (`BUTTON` before the reply) and lands on the first new chip after the reply at each of the 5 steps (item, qty, time, summary, confirm). `document.activeElement` was never BODY. Esc closed the chat and focus returned to the FAB. A chip click by script/mouse with focus on the input leaves focus on the input.
- First 12 Tab stops of the page all have `:focus-visible` true and a solid 3px outline.

## Not verified
Real phone, screen reader, Firefox/Safari. WCAG conformance needs manual assistive-technology testing. Static check re-run: 0 occurrences of innerHTML / insertAdjacentHTML / outerHTML in index.html.
