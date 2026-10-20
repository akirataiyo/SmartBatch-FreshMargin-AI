# SmartBatch single-file redesign: guided calculator, customer chatbot demo, AWS explainer (pass 2)

The change rewrites `index.html` in place (one vanilla file, inline CSS and JS, no build, no secrets) and adds one item to README section 5. It replaces the blue-gradient card layout with a warm paper and ledger look, turns the calculator into a 3-step flow on a single `state -> Core.derive -> render` pipeline, adds a keyword-based Thai customer chatbot that reads the same derived state, and replaces the ASCII AWS box with an SVG flow diagram, walk-through, cost table and roadmap. This is the second pass. Both blocking defects from pass 1 are fixed in the code and covered by the coder's iteration-2 evidence, and I found no new blocking concerns.

Watch for: (1) the page opens on a red "ต่ำกว่าเป้า" banner because pork_rice has a 36.0% store margin at benchmark prices (confirmed, a deliberate and flagged product decision, data unchanged from the original); (2) `.agents/tasks/index.original.html` is the only backup of the old page and should be removed once the user accepts the result (confirmed, non-blocking).

**Verdict**: APPROVED

## High-level view

The calculator and chatbot share one state object. Every input handler goes through `update()`, which recomputes `derived`, re-renders, and notifies the chat. The bot reads price and remaining stock from `derived`, not from the DOM, so changing a price or quantity changes its answers. The coder's runs show the croissant count going 21, 17, 14 as the plan and an order changed it, and the sell price 55 to 60. The formula block implements the README formula exactly (`targetProduction(20,30,5,24)` = 48 in the running page).

Both pass-1 defects are closed. Order confirmation now tests negation first and requires the whole normalized answer to equal a yes word, so "ไม่ใช่" and "ไม่ยืนยัน" fall through to the "กด ยืนยัน แก้ไข หรือยกเลิก" prompt. Chips use `aria-disabled` while the bot is "typing" and focus moves to the first new chip (or the input) after the reply, so a keyboard user never lands on `<body>`. The pickup time is now compared in minutes against 06:30-16:00.

User text reaches the DOM only through `textContent`. A static search of the current file finds no `innerHTML`, `insertAdjacentHTML`, `outerHTML` or `document.write`, and no `linear-gradient` or `backdrop-filter`. The injection strings render as literal text per the coder's runs. The demo label naming Bedrock, Lambda and LINE OA is permanent in the panel. The screenshots I viewed (375 top, 1280 AWS) show the paper and ink identity, the inline-SVG diagram with plain-Thai analogies per node, a dashed LINE OA node for the chat extension path, and ledger-style numbers; the only overlap is the fixed "ถามร้าน" button over page content, which is expected for a floating button.

<details>
<summary>Issues (3)</summary>

1. **Opens on a red margin warning (non-blocking, confirmed)** — pork_rice is below the 40% target at benchmark prices, so first load shows the warning. Decide whether to default to a passing dish; the coder already asked this.
2. **Backup file left in task directory (non-blocking, confirmed)** — `.agents/tasks/index.original.html` is the only copy of the old page (no git repo). Delete it after the user accepts.
3. **Evidence gaps (non-blocking, confirmed)** — only Chromium (Edge) was tested; no real phone, screen reader, Firefox or Safari. The calculator-to-bot link was not re-run in iteration 2 beyond the baseline value of 21. Do a manual assistive-technology and cross-browser pass before calling it WCAG-conformant.

</details>

<details>
<summary>Details</summary>

## Fix verification for the pre-order flow

I read the new code rather than re-running the suites. In `flowStep`, `negated = /ไม|^no(pe)?$/.test(norm)` is evaluated first, and confirmation requires `CONFIRM_WORDS.indexOf(answerCore) !== -1` after trailing particles are stripped, so the old substring match on "ใช" is gone. The edit branch is also skipped when negated. None of the accepted yes words contains "ไม", so the negation test cannot block a legitimate confirmation. In `next()`, `hadFocus` is computed before `renderChips()` replaces the chips, then focus goes to the first enabled chip or the input; the chip click handler ignores presses while `busy`. The time step parses H, H:MM and HMM, rejects minutes of 60 or more, and uses `hhmmToMin(SHOP.open/close)`. The coder's evidence shows "6" and "16:45" rejected, "16:00" accepted, and chip focus never on `<body>` across five steps.

## Evidence limits

I read verification.md (both iterations), the prior review, the changed chat code, and two screenshots. I did not re-run the verification suites or open the page, since no specific doubt remained. Requirements (1) to (4) in the brief are each covered by recorded evidence: chatbot intents and fallback chips, shared state, keyboard and aria-live, demo label and escaping; no gradients, glass, emoji or card-soup, with contrast ratios computed at 4.5:1 or better, 3px focus rings, reduced-motion, 44px targets and 360px with no horizontal scroll; the three-step calculator with the 3 scenarios, the "ทำไมถึงได้ตัวเลขนี้" explanation, validation and guarded localStorage; and the AWS diagram, walk-through, cost table, roadmap and chat placement. The only network reference is the Google Fonts link, with a system Thai fallback.

</details>

<details>
<summary>File map</summary>

`index.html`: rewritten with the new CSS system, SVG icon sprite, 3-step calculator, AWS section, chat panel, and inline CORE, BOT and APP scripts.
`README.md`: section 5 only, one added item "อัปเดตล่าสุดของต้นแบบ" with three sub-bullets.
`.agents/tasks/index.original.html`: backup of the old page, not part of the product. There is no git repo, so diff against this file.

</details>
