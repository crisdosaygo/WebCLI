---
name: web-cli
identifier: dosaygo-web-cli
description: Use the `web` CLI to browse, search, inspect, and act on live web pages from an agent workflow. Prefer this skill when a task needs browser state, web navigation, stable refs, form/text interaction, page text, or inspectable actions instead of screenshots.
---

# Web CLI

A real, persistent, logged-in browser with stable numbered action refs. You
**perceive** the live page, **decide**, and **act** — you are not writing brittle
automation scripts. Treat every site as a problem-solving domain and the `web`
CLI as your toolkit.

This skill is organized as:

- **Core loop** — the rhythm of every session.
- **Perceiving** — inspect, find, read, and how to find what's not visible.
- **Acting** — clicks, text, forms, selects, files.
- **Obstacles** — overlays, blocked states, auth, getting unstuck, and reporting tool issues (see "When you're stuck" and `report-issue`).
- **Context** — frames, tabs, profiles, waiting, recovery.
- **Reference** — full command list, shell composition, diagnostics.

---

## Core loop

**Look → act → re-inspect. Be a human at a browser.**

```sh
web inspect          # look — numbered actions on the page
web do 3             # act on a ref
web inspect          # re-inspect — refs reset after any state change
```

Don't see what you want?

```sh
web hover 20         # hover a ref in the area you expect the content
web scroll           # scroll from there → hits the right panel
web inspect          # look again
```

**Scroll first, then inspect.** Most things are below the fold.
`--ignore-viewport` is a last resort, not a first move.

**Efficiency is successful completion, not minimum commands.** On dynamic pages,
re-reading after meaningful state changes is often faster than trying long
chains against stale refs.

**Complex widgets need room.** For large calendars, maps, editors, drag/drop
surfaces, or multi-panel pickers, expand the browser viewport before interacting
so the whole widget stays visible while the pointer moves and the page updates.
After resizing, verify the viewport actually changed and re-inspect so refs are
regenerated before using the widget.

### Session start checklist — every session, before your first action

```
1. web inspect      — raw, no grep. Orient completely.
2. web tab list     — confirm which tab you are targeting.
3. web frame list   — note which frames are present.
4. Dismiss overlays — if inspect showed a modal/banner, close it NOW.
```

Skipping this is the single biggest cause of wasted actions. It costs 4
commands and saves 40.

### Refs and sheet epochs

Every `find` mints a new ref sheet (`Sheet: s104` in the output), and so does
`inspect` **whenever the actionable refs actually changed**. Re-inspecting an
unchanged page is cheap and safe: `inspect` keeps the same sheet id and exits
zero, so `web inspect && …` won't break just because you looked twice. Refs are
valid only for the current sheet — they reset on any action that changes page
state. When that happens the CLI tells you:

```
↳ sheet s104 → s110 (refs reset)
↳ exit: non-zero because refs changed; stop chained commands and re-inspect before acting
```

Any ref you were holding is now invalid — re-inspect before the next action.
Agents often chain commands together with shell operators such as `&&`; that is
fragile because many browser interactions invalidate refs. BrowserBox attempts
to detect ref-epoch changes and may intentionally return a non-zero exit code
after a successful command when refs become invalid, so chains stop before using
stale refs. This is a guardrail, not a guarantee: if a flow still behaves
strangely, fall back to the strict look → act → re-inspect loop. Silence means
refs are stable.

After every click, **read the confirmation label**:

```
ok clicked tab=5 Settings
```

If it doesn't match what you intended, you clicked the wrong element. Stop,
re-inspect, and correct before continuing. The label says *what* was clicked;
the epoch notice says *the map changed* — two signals for the same event.

> **Tip — assume refs expire after every action.** Treat re-inspect as
> mandatory after every `do`, `click`, `type`, `scroll`, `frame switch`, or
> navigation. If refs survive unchanged the CLI will tell you (same sheet id,
> exit zero) and you can act immediately — but that's a bonus, not something
> to count on. The correct mental model is: *one action → one re-inspect →
> next action*. Optimise only when you have confirmation the sheet is stable,
> never by assumption.

---

## Perceiving the page

### inspect, find, and grep

**`web inspect` is the complete map of everything actionable on the page.** It
is the default: use it when orienting, after navigating, when overlays or errors
might be present, or whenever you haven't seen this page this session. Nothing
replaces it for first contact.

**`web find` + `web do` is the fast path once you know a label** — usually
quicker than a full inspect for navigation:

```sh
web find "Create"          # locate by label (a different path than inspect)
web do 2                   # act
web find "Submit" --all    # --all when there may be duplicates
```

After `find`, use **only** the refs that command returned — never mix them with
refs from a prior inspect. In `&&`-chained commands the first command's refs are
already stale by the time the second runs.

**Grepping inspect output trades safety for brevity.** It shows only what you
already expected — you miss overlays, loading states, unexpected controls, and
errors that would change your next move, and one blind spot compounds into
several wrong actions. Reach for it only when:

- you already have a full inspect of this same page load in context, and just
  need the current ref number for a known element, or
- the page is large, stable, and familiar (repeat visit, no loading, no
  overlays) and you know exactly what label you want, or
- context is tight and preserving it outweighs the orientation you'd gain.

```sh
web inspect | grep -i "submit"   # acceptable only when already oriented
web inspect                      # after ANY grep, your next inspect is raw
```

**The discipline:** never open a session with grep, never grep right after
navigation. It's a focused second look, never a substitute for the first.

### Reading text

```sh
web read                         # visible text
web read --full                  # whole document
web read --headings | --links    # structure or link list
```

### When inspect doesn't surface what you need

Escalate in order; stop as soon as something appears:

```sh
web find "Submit" --all          # 1. text search — different path than inspect
web hover <ref-in-area>          # 2. hover a ref near where it should be …
web scroll && web inspect        # 3. … scroll that panel and re-inspect
web frame list                   # 4. is it inside a frame?
web frame switch f1              # 5. switch and retry
web inspect --layer active       # 6. foreground layer (modal/dropdown open)
web inspect --ignore-viewport    # 7. last resort — offscreen elements
```

**No find matches + frames visible = switch the frame first.** Content inside an
iframe is invisible to `find` on the top frame.

Bare numbers are refs; force text matching when a label *is* a number:

```sh
web click text:"8"
```

### Scrolling

```sh
web scroll              # page down       web scroll up        # page up
web scroll <ref>        # bring ref to center of viewport (less sticky-header occlusion)
```

Pages have many scrollable containers; bare `web scroll` hits whichever one the
pointer is over. **To scroll a specific panel reliably, hover a ref inside it
first** — exactly how a human aims a mouse:

```sh
web hover 44            # a ref inside the panel you want to move
web scroll              # fires from there → correct container
web inspect             # always re-inspect after scrolling — refs and layout change
```

---

## Acting on the page

### Clicks and refs

```sh
web do <n> ["text"]              # run action n; optionally supply text/option value
web click <n|"text"> [--new]    # click a ref or a label; --new for new tab
```

Prefer `web do <n>` for inspect refs — it takes a non-selector path that is more
robust than label clicks.

**Text-query disambiguation.** `web click <text>` returns matching refs when
more than one element matches. Use this as a fast disambiguation flow when you
know what you're looking for:

```sh
web click Issues           # returns all matching refs if ambiguous
web do 5                   # pick the right one
```

This is often faster than a full inspect when the label is known. `--new` works
with text queries too: `web click Issues --new` opens the best match in a new tab.

### Text input

```sh
web type 3 "hello"      # key events — framework-managed / validated fields, comboboxes
web say "long text"     # direct paste — plain text fields, no key events
web clear               # clear the focused field
```

**Pipe a file or heredoc into a field with `-`.** Anywhere text is expected,
pass `-` to read it from stdin (read to EOF; a single trailing newline is
trimmed, interior newlines are preserved):

```sh
cat body.md | web do 4 -        # type a file into the inspect ref's field
echo "$TITLE" | web say -       # paste piped text into the focused field
web type 3 - <<'EOF'            # heredoc into a ref
multi-line value
stays intact
EOF
```

`-` is only special when it is the *entire* value — `web say "a - b"` types a
literal hyphen.

**`web say` does not accept a ref.** It pastes text into whatever element is
currently focused/active. Click to focus first if needed. `web type` takes a
ref; `web say` does not — paste only, no targeting.

`type` and `say` work without a ref when a field is already focused — click to
focus first if needed. **Use `web clear` to empty a field**; `ctrl+a`/`cmd+a`
are unreliable in browser context.

**Clear before typing into a ref with existing content.** If a field may already
have a value, run `web clear` first — typing into a pre-filled field appends or
merges unpredictably:

```sh
web clear               # empty whatever is in the field
web type 5 "new value"  # now type cleanly
```

**Comboboxes / custom dropdowns:** open, then hover-and-scroll to reveal options
rather than trusting an in-list filter (filters are sometimes scoped to the
current selection):

```sh
web do 23               # open the combobox
web hover 24            # an item visible near your target
web scroll              # scroll the list (pointer is in the right container)
web inspect
web do 30               # your target
```

### Forms and selects

```sh
web form                        # all fields, current values, and errors
web form | grep -F "[error:"    # validation errors only
web type 4 "value"
web choose 7 "Option name"      # native <select> or range / aria-slider
web options 7                   # list a native select's options
web submit                      # submit the nearest form
```

### File choosers

```sh
web do 1 --file "/absolute/path/to/file.ext"
```

`--file` is required — passing the path as a positional arg will error. Use this
surface for uploads; do not trigger file inputs via `eval`.

### Resistant fields

`web clear` fires synthetic events, then Space+Backspace key events, and if the
old value persists, one Backspace per remaining character. The output's
`strategy=semantic/fallback` is informational — no manual intervention needed.

---

## Obstacles

### Overlays, dialogs, and blocked states

**Two tiers — only one demands immediate action:**

- **Blocking** — `aria-modal="true"`, `role="dialog"`/`alertdialog`,
  focus-trapped, or covering the central interaction zone. **Dismiss before
  doing anything else.** Overlays pollute inspect output and block the clicks
  you actually want; the cost of dismissal is one action, the cost of ignoring
  it is 5–10 wasted ones.
- **Informational** — fixed/sticky sidebars, nav rails, summary panels,
  persistent UI. Surfaced so you *can* dismiss them if they're in your way, but
  they don't require action. Some aren't fully dismissable at all (a navigation
  sidebar may be permanent) — don't burn actions fighting one. Check
  `Current frame actions`: page refs remain available regardless.

**Dismissal ladder** — stop as soon as it clears:

```sh
# 1. surfaced recovery actions (check inspect output)
web do 1            # usually "Press Escape"
web do 2            # usually "Click close button" / "Click blank space"
web press Escape    # or directly, if no close ref is surfaced

# 2. find the close control by label
web find "close" --all
web do <N>

# 3. pointer activity — some overlays dismiss on movement
web hover 1 ; web hover 2 ; web hover 3
web inspect

# 4. identify the structure
web snap overlay-check.png
web inspect --layer active
```

**Hard modal — escalation ladder** (when 1–4 don't clear it):

If standard actions and pointer movement haven't worked, escalate through these
before handing off to a human. Stop as soon as it clears.

```sh
# 5. random mouse trace — dislodges overlays that dismiss on pointer movement
#    but only respond to real cursor travel, not hover-on-ref
web eval "(() => { const steps = 20; for (let i = 0; i <= steps; i++) { const x = Math.round(100 + (window.innerWidth - 200) * i / steps); const y = Math.round(100 + (window.innerHeight - 200) * Math.sin(i / 3)); document.dispatchEvent(new MouseEvent('mousemove', {clientX: x, clientY: y, bubbles: true})); } })()"
web inspect

# 6. screenshot + coord click — when the close button is visible but not in the
#    action sheet (e.g. image-map area, canvas-rendered UI, non-DOM overlay)
web snap before-click.png             # look at the image to find the button
web click --coords 450,30             # click the pixel coords of the close button

# 7. programmatic investigation — when the overlay structure is opaque
#    (image maps, canvas, shadow DOM, custom elements)
web eval "document.querySelector('[usemap]')?.usemap"
web eval "[...document.querySelectorAll('map area')].map(a=>({alt:a.alt,coords:a.coords,href:a.href}))"
web eval "document.querySelector('canvas') ? 'canvas overlay' : 'no canvas'"
# then act on what you learn:
web click --coords <x>,<y>            # coords from area element or visual inspection
```

**Image-map modals** (`img[usemap]` + `<area>` elements): a retro pattern used
by some survey/consent overlays to hide cancel/no buttons inside `<area>` coords.
`web status` will flag this with a `dismissalHint` in its output and surface a
ready-made eval action. Run that eval, read the `coords` of the cancel/no area,
then `web click --coords <cx>,<cy>` where cx/cy is the centre of that area rect.

If 1–7 don't clear it — **hand off cleanly, don't loop:**
- no unsaved state: `web human-drives` → human closes it → `web agent-drives`
- unsaved state: `web pause "Please close the overlay in the browser window."`
- nothing works: say so explicitly.

**Notes:** While blocked by a modal, interaction commands still work — the
focused element always appears in inspect, so you can type directly into dialog
inputs. If a click reports `action_occluded`, don't retry the same ref —
re-inspect, dismiss the cover, or `web snap` to see what's on top.

### Auth, blockers, and handoff

**Hard rule: CAPTCHAs, login credentials, MFA, passkeys, and payment pages
always require a human. Switch to `human-drives` the instant you encounter one
and ask the human for help. Do not attempt to solve, work around, or wait out
any of these — they are not your job.**

```sh
# The moment you see a CAPTCHA, login form, or MFA prompt:
web human-drives
# then tell the human exactly what you need:
# "Please complete the CAPTCHA on tab 1 and call `web agent-drives` when done."
```

A blocked tab is dead weight — handle the handoff *now*, not after exploring
other tabs. If several tabs in one profile hit the same gate, batch them into a
single handoff message rather than calling `human-drives` once per tab.

**What falls in this category (always hand off immediately):**

| Gate type | Rule |
|---|---|
| Login credentials / SSO | Hand off before typing the password field |
| CAPTCHA (any kind) | Hand off on sight — do not retry or reload to dodge it |
| MFA / 2FA / passkey | Hand off — never attempt automation |
| Payment / billing page | Hand off — PCI scope, never automate |
| Age gate requiring ID | Hand off |
| Access-denied / 403 | Hand off if credentials are required to resolve it |

**What you handle yourself (self-resolvable):**

Cookie consent banners, age-confirmation checkboxes, marketing opt-ins, and
similar low-stakes gates — dismiss these immediately without handing off.

**Modals and overlays are your job**, not a handoff reason. Work the full
dismissal ladder (steps 1–7 above) before considering a handoff. The only
exception is a modal that is genuinely undismissable after exhausting all
escalation steps — in that case you may use `web human-drives` as a last resort,
but explain clearly what you tried and what you need the human to do.

**Login gates** — agent fills public fields, hands off for credentials:

```sh
web type 5 "user@example.com"
web human-drives
# "Please enter the password and complete any MFA on tab 1, then run: web agent-drives"
web agent-drives
sleep 3
web inspect
```

**Mid-flow gates (CAPTCHA, sudo re-auth)** — hand off immediately; the human
resolves it in the visible browser while you wait:

```sh
web human-drives
# "A CAPTCHA appeared mid-flow. Please solve it and run: web agent-drives"
web wait url:"/next-step" --timeout 120000
web agent-drives
```

Do not use `web wait` alone hoping a CAPTCHA resolves itself — switch to
`human-drives` first. `human-drives` restarts the daemon and can lose form
state, so always hand off *before* mid-form navigation when you can anticipate
the gate.

**Pre-baked profiles for persistent logins.** Human→agent handoff inside an
existing session often works, but some sites remember how the session began and
keep earlier risk signals. When persistent logins matter, create or choose a
fresh profile, enter Human Drives first, manually authenticate, close the
browser, then use that authenticated profile from Agent Drives. Starting from a
profile that already contains trusted login state can reduce verification loops.

**Bot gates** — navigate to sign-in and use `human-drives` to bake a real
session. If the site requires it even after a human session: `web set human-mode`
(experimental human-mode presentation; may break some sites).

**Last resort — default profile copy (human consent required):**

If a site consistently rejects the agent browser even after human-assisted login
— bot detection based on browser fingerprint, cookie history, or account
trust-scoring — the human can opt into copying their *real* default Chrome
profile for `web` to use. This imports their existing sessions, cookies, and
site-trust history directly.

**Never do this without the human's explicit consent.** Explain what it means
before suggesting it:

> "This will copy your real default Chrome profile (including all your logged-in
> sessions, saved passwords reference, and cookies) into a separate directory for
> web CLI use. The copy is isolated — web CLI changes won't affect your real
> browser — but it does read your live profile data. If you want to proceed, first
> log into [site] in your regular Chrome, then tell me and I'll run the copy."

Suggested workflow once the human agrees:

```sh
# Human: log into the problem site in their regular Chrome browser first,
# so that session/cookie is fresh in the real profile.

# Then agent runs:
web set default-profile          # copies real default Chrome profile to web CLI
web status                       # confirm the profile is active
web go https://problem-site.com  # session should now be present
```

If the copy is stale (the human logged in after the last copy), refresh it:

```sh
web set default-profile          # re-copy to pick up the fresh login
```

### When you're stuck

**First, notice it.** Signs: identical inspect output twice, the same action
failing repeatedly, rotating tabs without advancing any, rising uncertainty
about what to try. The moment you spot one — stop and reorient.

Then work this escalation, stopping as soon as something moves:

1. **Re-read this SKILL.md** — the relevant section, properly, not skimmed.
2. **`web --help` or `web <command> --help`** — a forgotten flag is often the unlock.
3. **Try an unused command** — `read --full`, `observe`, `inspect --layer active`,
   `inspect --ignore-viewport`, `describe <n>`, `eval` for diagnostics.
4. **Do something seemingly random** — reload, navigate back and in, hover a
   different ref, scroll the other way, move the mouse. State changes unstick UIs.
5. **Stop and think** — re-read the last inspect carefully. What does the page
   structure tell you? What would a human do here?
6. **If the problem appears to be in WebCLI itself** (command broken, behavior
   inconsistent with docs or ladders, missing capability, repeated failure after
   recovery steps, tool reliability issue): consider `web report-issue`.

**Report-issue workflow.** If WebCLI appears to have a bug, missing capability,
or reliability issue, consider opening an issue on the project repository. This
helps improve the tool and provides maintainers with real-world failure cases.
Depending on the environment, you may ask the user for approval before creating
the issue. The important thing is making "report issue" part of the obstacle
ladder rather than treating failures as dead ends. `web report-issue` stages a
pre-filled report (with session context) under the hood; review with `web status`
or the pending files if needed.

By default `web report-issue` navigates to the GitHub new-issue URL **in the
current web CLI session** (same browser). Pass `--system-browser` to open it
in your OS default browser instead:

```sh
web report-issue                    # opens GitHub in the active web CLI tab
web report-issue --system-browser   # opens GitHub in your system browser
web report-issue --batch            # opens all pending issues (web CLI session)
```

**Never loop.** Same action twice with no result → change approach, not repeat.
Two tries is the limit; a third is a loop.

**Keep a lab book when overwhelmed.** Juggling many tabs/profiles, write a
scratch file and reason from it — it prevents re-running dead ends:

```sh
echo "## Persistent overlay on checkout
- Tried: Escape x2, surfaced close control, click blank space
- Observed: re-mounts after every dismiss, z-index 9999
- Not tried: pointermove loop, inspect --layer active, reload
- Hypothesis: JS listener re-mounts on pointer events" >> /tmp/labbook.md
```

**Don't abandon a site.** Keep working the problem — a different command, entry
point, frame, or the active layer. If genuinely stuck, ask the human rather than
silently moving on. Only drop a site when the human says to.

---

## Context: frames, tabs, profiles

### Frames

```sh
web inspect             # read the "Frames (visible)" section
web frame switch f1     # sticky — all commands now target f1
web inspect
web frame switch top
```

Many single-page and portal-style apps render their real content inside an
iframe, so the top frame looks nearly empty. Inspect tells you what's in each
frame before you switch:

```
Frames (visible):
[f1] "content-frame" https://...
     ↳ "Account settings" — 40 buttons — e.g. Create, Manage
```

Switch to the frame with the buttons you need before acting. Frame labels reset
after every `web go`.

### Tabs

```sh
web tab new https://example.org
web tab list
web tab switch 2 | "partial title" | t3
web tab close 2
web quit --all          # stop all profile daemons
```

Close unneeded tabs periodically, especially before a handoff. When titles are
ambiguous, use the number from `web tab list`.

### Profiles

```sh
web set profile --name work     # persistent — all commands use this profile
web --profile other status      # one-shot override
web unset profile               # back to default
```

When driving several profiles at once, rotate through all of them every 3–5
actions — a silent tab may be sitting on a blocker. **Each time you touch a tab,
complete a unit of work** (at least one meaningful step); inspecting the same
state again is a micro-loop. If you land on a blocked tab, resolve or hand it off
before moving on.

### Waiting

Prefer `sleep` + `inspect`. Use `web wait` only for a known exact condition:

```sh
web wait text:"Dashboard"
web wait url:"/dashboard"
web wait title:"Home" --timeout 10000
```

### Recovery from degraded states

- `state=degraded:page.error` — often still partially usable. `web reload`, then
  re-inspect; `find` and `read` on visible text usually still work.
- `state=degraded:page.action_occluded` — don't retry the same ref. Re-inspect,
  dismiss the cover, or `web snap` to diagnose.
- ARIA live regions in inspect (`[polite]`/`[assertive]`) report real-time state
  even when the document has errors — read them.

---

## Navigation philosophy

**Navigate like a person.** Start from a homepage or main section and click
toward your target. Constructed deep-link URLs are fragile — they skip app
initialization, break session state, and trip bot detection. Use a direct URL
only when you know the app produces it as a stable, shareable link.

The ladders here are high-signal starting points, not scripts. Real pages are
messy: overlays misbehave, content loads late, structures differ from spec. When
a pattern doesn't fit, improvise the way a human with a browser would. Prefer
core commands first, then advanced, then debug tools.

---

## Reference

### Shell composition

Pipe high-volume **structured** output — not `inspect`:

```sh
web form | grep -F "[error:"
web status --json | jq '.data.states[] | select(.kind=="form.error") | .details.errors'
```

`web inspect --json` is for diagnostics, not for jq-filtering refs — use
`web find` for that. After any grep of inspect, your next inspect must be raw.

### Diagnostics

```sh
web status                      # session + page state + blocked conditions
web describe <n>                # tag, role, selector, visibility
web eval "document.title"       # JS diagnostics ONLY — never for interaction
web transcript --path           # full .jsonl history path
```

Screenshots are a last resort and token-heavy (`web snap` = `web eval`) — use
only when text/inspect/read can't answer. (`web capture` is an alias for `web snap`.)

### Recording and demos

- `web click <n> --trace-pointer` on important actions — the pointer trail shows intent.
- `web highlight <n>` to call attention to what you see.
- Don't `web snap` for recordings — the terminal + browser already show everything.

### Command reference

```
Core
  go <url>                     Navigate; also accepts history keywords
  inspect [--ignore-viewport] [--verbose] [--layer active] [--layers]
  do <n> ["text"]              Run action n; optionally supply text / option value ("-" = stdin)
  click <n|"text"> [--new] [--trace-pointer] [--coords X,Y]
  type <n> "text"              Key events — framework-managed / validated fields, comboboxes ("-" = stdin)
  say "text"                   Paste into the focused field (no key events; "-" = stdin)
  clear [<n>]                  Clear a field
  read [--full|--ignore-viewport] [--headings|--links]
  find "text" [--all] [--ignore-viewport|--within-viewport] [--verbose]
  disambiguate "text"          All elements matching a label, each with href
  form [--ignore-viewport]     Form fields with current values and errors
  submit                       Submit the nearest form
  press <key>                  Enter, Escape, Tab, ctrl+a, …
  choose <n> <value>           Native <select> or range / aria-slider
  options <n>                  List options for a native <select>
  scroll [down|up|<ref> [up]]  Scroll page, or hover a ref then scroll its panel
  reveal <n>                   Scroll a ref into view without clicking
  hover <n>                    Hover (triggers tooltips and menus)
  back / forward / reload
  status                       Session + page state + blocked conditions
  tab new|list|switch|close|find
  frame list|switch
  quit

Handoff
  human-drives / agent-drives / pause / resume

Configure
  set / unset / get            Sticky settings: profile, headless, ref-shape, …
  use <browser>
  teach                        Write skill file into agent config directories

Advanced
  wait <condition>             Wait for text, URL, title, or selector
  observe --json [--ignore-viewport]   Emergency: status + read + inspect + form
  refs / last
  focus / caret / select-text
  snap [<file>]                Screenshot to a file (alias: capture)
  search <query>
  connect / disconnect

Debug
  eval "<js>"                  Diagnostics only — never for interaction
  describe <n>
  highlight <n>
  transcript [--path]
  report-issue [--system-browser]
  version / readme
```

**Stuck on syntax? `web <command> --help`.**

<!-- web-cli-skill-start -->
## Driving the Web

This project uses the `web` CLI for browser automation and real web interaction.

The web CLI skill sheet is installed at:

- `.claude/skills/web-cli/SKILL.md`
- `.grok/skills/web-cli/SKILL.md`
- `.gemini/skills/web-cli/SKILL.md`
- `.copilot/skills/web-cli/SKILL.md`
- `.codex/skills/web-cli/SKILL.md`

Load the relevant file for your agent before performing any browser task. Run `web teach` to install or refresh these files.
<!-- web-cli-skill-end -->
</content>
</invoke>
