# What this room knows about learning and work

A single-file, live-updating visualization of a Google Form. People in the room
answer four questions on their phones; the page reads the response sheet every
15 seconds and redraws itself as a Minard-style flow diagram — each band is a
theme, its width is how many answers mention it, and the bands flow left to
right across the four questions.

**Live:** https://educoreproject.github.io/ed3_10_29/
**Sample-data mode (no sheet needed):** add `?demo` to the URL

The whole thing is `index.html` — about 450 lines of HTML, CSS and vanilla JS.
No build step, no dependencies, no server, no API key.

---

## Why it looks like a 19th-century troop map

The layout is lifted from Charles Joseph Minard's 1869 *Carte figurative des
pertes successives en hommes de l'Armée française dans la campagne de Russie
1812–1813* — the flow map of Napoleon's march on Moscow that Tufte called "may
well be the best statistical graphic ever drawn."

Minard's trick is that the thickness of the band **is** the army. 422,000 men
cross the Niemen as a wide tan river, thin steadily on the way east, and come
back as a black thread of 10,000 — direction, position, headcount and attrition
all legible at once, with no legend and barely an axis. And the title says the
real subject out loud: *pertes successives*, successive losses. The map isn't
about the campaign. It's about what the campaign cost.

That's the borrowed idea here. The question isn't "what does this room think" —
it's **what does this room know from experience that doesn't survive into its
own picture of the future.** Attrition is the subject, so Minard's grammar fits.

### What maps onto what

| Minard, 1812 | This page |
|---|---|
| Band width = soldiers still alive | Band width = answers mentioning a theme |
| x-axis = longitude, Kaunas → Moscow → back | x-axis = four questions in life order: how we learn → what first jobs taught → what we hire for → what post-K12 should be |
| **Tan** band = the advance | **Tan** band = theme carries into post-K12 (≥60% of its peak) |
| **Black** band = the retreat | **Black** band = theme lost before post-K12 (<30% of its peak) |
| Temperature plotted below, sharing the axis | Response timeline plotted below, sharing the width |

The thinning middle case — a theme that makes it but weakened — gets its own mid
brown, so the three-way read is *survived / thinned / died*.

Two smaller inheritances:

- **Vertical ordering follows the fate of the band.** `geometry()` sorts by
  survival ratio so carried themes ride along the top and lost ones sink toward
  the bottom. The code comment says it plainly: `// carried themes on top; lost
  ones sink to the bottom, like Minard's retreat`.
- **The "lost" color is the ink color.** `--lost` is defined byte-identical to
  `--ink` in both light and dark themes (`#1c1a17` / `#efe6d4`), and `--on-lost`
  equals `--paper`. So a dying theme is always drawn at maximum contrast against
  the page, whichever way the theme flips — dark mode inverts the palette without
  softening the point.

The typography is doing the same job: EB Garamond for body text and IM Fell
English — a digitization of 17th-century punches — for headings and band labels,
on a paper-and-ink palette. It should read as a lithograph someone pulled off a
press, not a dashboard.

### Where the sidebar fits

The **"For context"** panel is the same figure drawn plainly: of every 100 U.S.
ninth-graders, 87 graduate on time, 54 enroll the next fall, 32 finish a
credential within six years, 68 reach the workforce without one — about 25 of
100 at high-poverty high schools. That's a dwindling band too. It sits next to
the flow diagram on purpose: thematic attrition in the chart, human attrition in
the margin.

### Where it departs from Minard

Worth being clear about, because the visual form implies more rigor than the
data underneath actually has:

- **Mass isn't conserved.** Minard's army can only shrink; a band here can grow
  from one column to the next. These aren't flows of a fixed quantity splitting
  and rejoining, so it is **not a Sankey diagram** — each band is independently
  re-measured at each column and then smoothly interpolated between them.
- **Bands double-count.** One answer can match several themes, so the total
  stack thickness exceeds the number of respondents. Minard's 422,000 was a
  headcount; these counts are mentions.
- **The x-axis is neither space nor time.** It's rhetorical order — four
  questions arranged as a life arc. Position means "asked fourth," not "600 miles
  from home."
- **Width measures keyword hits, not a measured quantity.** Minard's encoding was
  literal. Here it's a proxy for attention, produced by the keyword matcher
  described below — which is exactly why the page footer hedges the bands as a
  first pass rather than a coding of the answers.
- **It's live, not final.** Minard drew his map 57 years after the retreat, from
  a closed dataset. This one redraws every 15 seconds in the middle of the
  conversation it's describing, which is why the bands tween rather than cut.

---

## How it reads the Google Form

### 1. The form writes to a sheet; the page reads the sheet

The page never talks to the Forms API. In Google Forms, **Responses → Link to
Sheets** creates a spreadsheet where each submission is appended as a row. That
sheet is the data source.

The sheet must be shared **"Anyone with the link → Viewer."** If it isn't, the
request fails and the page falls back to sample data (see below).

The sheet ID is hardcoded near the top of the script:

```js
const SHEET_ID = "12O50Kkw1B6QJM2_v9ZE4UrLJX229J3TBphEfUkojZdA";
const SHEET_TAB = "";   // blank = first tab, where Forms writes responses
const POLL_MS   = 15000;
```

You can point the page at a different sheet without editing anything by passing
`?sheet=<id or full URL>` in the query string.

### 2. Fetching it without CORS

Google's `gviz` endpoint doesn't send CORS headers, so `fetch()` would be
blocked from a github.io origin. Instead the page uses **JSONP**: it injects a
`<script>` tag at

```
https://docs.google.com/spreadsheets/d/<ID>/gviz/tq
  ?tqx=out:json;responseHandler:__sheetcb1&headers=1&_t=<cachebuster>
```

and Google calls back into a one-shot global function with the parsed table.
`sheetRows()` wraps that in a Promise with a 12-second timeout, flattens the
`{cols, rows}` payload into a plain array-of-arrays (header row first), and
cleans up the global and the script tag afterward.

`poll()` runs this immediately on load and then every 15 seconds.

### 3. Figuring out what the columns mean

`ingest()` never assumes a fixed column order — it reads the header row and
classifies each column by regex, so form questions can be reworded or reordered
without touching the code:

| Header matches | Treated as |
|---|---|
| `timestamp` / `submitted` | submission time (drives the timeline chart) |
| `how old` / `age` / `age range` | grouping + dot color |
| `joining` / `where are you` / `location` | grouping, if no age column |
| `email`, `name`, `score` | **dropped — never rendered** |
| `insights` | curated facilitator notes, shown in the sidebar |
| anything else non-empty | a question column |

Each question column is then assigned to one of four lanes in the flow diagram,
again by regex on the question text:

```js
{ id:"learn",  short:"How we learn",                   re:/how .*learn|learn best/ },
{ id:"job",    short:"What first jobs taught",         re:/job|work(ed)? experience/ },
{ id:"hire",   short:"What we hire for",               re:/hire|hiring|characteristic|look for/ },
{ id:"future", short:"What post-K12 should look like", re:/post-?k-?12|post-?secondary|.../ }
```

Unmatched questions are dropped into whatever lanes remain, in order, so a form
with four free-text questions always fills the diagram even if the wording
drifts.

### 4. Grouping answers into themes

Themes are **keyword-matched, not model-classified.** There are 13 hand-written
themes, each defined as a list of stems:

```js
["ethic", "Work ethic & showing up",
  ["hard work","work ethic","show up","on time","reliab","dependab", ...]],
["solve", "Problem solving",
  ["problem solv","solve","critical think","figure out","resourceful", ...]],
// …11 more
```

Each stem is compiled to `/(^|[^a-z])<stem>/i` — a cheap word-boundary check
that still matches prefixes, so `reliab` catches *reliable*, *reliability*, and
*reliably*. Every answer is tested against every theme, so **one answer can
carry several themes**; an answer that matches none is bucketed as *Other /
unthemed*.

This is why the footer says to read the bands as a first pass rather than a
coding of the answers. It's transparent and instant, but it will miss anything
phrased outside the keyword lists.

### 5. Turning theme counts into the diagram

`counts()` produces, for each theme, a 4-element vector: how many answers in
each of the four questions mention it. From there:

- **Band thickness** at each column = that count, scaled to fit 390px of
  vertical space (unit height capped at 26px, 4px gaps).
- **Band color** comes from `klass()`: compare the post-K12 count to the largest
  of the first three. Ratio ≥ 0.6 → **tan** (carries through); < 0.3 → **black**
  (lost before post-K12); in between → **mid brown** (thins out) — the advance
  and retreat colors, per [the Minard section](#why-it-looks-like-a-19th-century-troop-map).
- **Vertical order** sorts by that same survival ratio, so carried themes ride on
  top and lost ones sink to the bottom.
- **Shape** is one SVG path per theme: cubic Béziers along the top edge left to
  right, then back along the bottom edge, closed. Between renders the page tweens
  from the previous geometry over 850ms with a cubic ease-out, so new responses
  make the bands visibly swell rather than jump.

Hovering a band shows the two most recent quotes behind it per question; clicking
it filters the whole page (quote feed, spotlight, sidebar) to that theme.

### 6. What else the page derives

- **Discussion prompts** (`renderPrompts`) — rule-based, generated from gaps in
  the data, max four at a time. The main rule: find themes that are strong in the
  lived-experience questions but weak in the post-K12 answers, and ask why they
  fall out of the design. Others surface the top post-K12 theme, the top learning
  style, the hire-vs-design gap, and — if age data exists and there are 3+ people
  on each side — the single biggest under-35 vs. 35-and-over split. Prompts only
  appear once there are at least 3 responses.
- **Spotlight** — rotates the longest answers (>12 chars) every 9 seconds, newest
  first; pausable, and respects the active theme filter. New responses jump the
  queue.
- **Timeline** — a cumulative step chart of submissions over time, with each dot
  colored by the respondent's age bucket.
- **Voices feed** — up to 120 recent answers, tabbed by question, with new ones
  flashing on arrival.
- **Readings & patterns** — anything typed into an `Insights` column in the
  sheet, split on an em dash or colon into headline + detail. This is the manual
  escape hatch for saying something the keyword matcher can't.

### 7. When the sheet can't be read

If the first poll fails — sheet not shared, bad ID, offline — the page prints the
reason, flips a **SAMPLE DATA** badge on, and runs `startDemo()`: ten built-in
responses that trickle in one every 6 seconds, so the whole thing still demos on
a plane. If the sheet was working and *then* fails, it keeps the real data on
screen and shows "Reconnecting…" instead. Force demo mode anytime with `?demo`.

---

## How it got live on github.io

GitHub Pages serves static files straight from a repo. Because this project is
one self-contained HTML file with no build step, there's nothing to compile —
Pages just hands `index.html` to the browser, and all the data fetching happens
client-side from the visitor's machine.

What was done:

1. **Created the repo** `educoreproject/ed3_10_29`, public. (Pages needs a public
   repo unless you're on a paid plan.)
2. **Uploaded the HTML file** (commit `58db731`, "Add files via upload") — drag
   and drop into the GitHub web UI, no git client required.
3. **Renamed it to `index.html`** (commit `e461ac5`). This is the step that
   matters: Pages serves `index.html` as the directory root, so the site answers
   at the bare URL instead of `…/transitions-map_2.html`.
4. **Turned on Pages**: repo **Settings → Pages → Build and deployment**, Source
   = *Deploy from a branch*, Branch = `main`, folder = `/ (root)`, Save.
5. Waited about a minute for the first build. The URL appears at the top of that
   same Settings → Pages panel:

   ```
   https://educoreproject.github.io/ed3_10_29/
   ```

Every push to `main` triggers a rebuild automatically.

### Things worth knowing

- **Shipping an edit:** change `index.html`, commit, push to `main`. Live in
  under a minute. GitHub's CDN caches aggressively — hard-refresh
  (<kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>R</kbd>) if the change doesn't show.
- **The sheet must stay public.** Pages is static; there's no server to hold a
  credential, so the browser fetches the sheet directly as an anonymous user. A
  sheet set to "Restricted" drops every visitor into sample-data mode.
- **Don't put anything sensitive in the sheet.** The sheet ID is visible in the
  page source, and anyone with it can read every column — including the ones this
  page deliberately hides (email, name). Keep the form anonymous.
- **HTTPS is automatic** on `*.github.io`, which matters because the Google
  endpoint is HTTPS-only.
- **Running it locally:** open `index.html` directly in a browser. JSONP works
  fine from `file://`, so no local server is needed.

---

## Reusing this for a different session

1. Build a Google Form with four free-text questions (plus an optional age or
   location question) and link it to a response sheet.
2. Share the sheet "Anyone with the link → Viewer."
3. Either edit `SHEET_ID` in `index.html`, or just load the page with
   `?sheet=<the sheet id>`.
4. If the question wording is very different from the originals, update the `re:`
   patterns in `COLS` so questions land in the right lanes — or don't, and let
   the fallback assign them in order.
5. Tune the `THEMES` keyword lists to the vocabulary the room actually uses. This
   is the highest-leverage edit in the file.
6. Hit **Present** for fullscreen before screen-sharing.

## File map

Everything lives in `index.html`, in this order:

| Lines | What |
|---|---|
| ~10–100 | CSS: paper/ink palette, auto dark mode, layout |
| ~100–150 | Markup: header, spotlight, SVG containers, sidebar |
| ~155–205 | Config (`SHEET_ID`, `POLL_MS`), column lanes, theme keywords |
| ~210–225 | JSONP sheet reader |
| ~227–256 | `ingest()` — header detection, row parsing, theming |
| ~258–325 | Flow geometry, path building, animation, hover card |
| ~327–360 | Discussion-prompt rules |
| ~362–408 | Sidebar panels, timeline, spotlight |
| ~410–445 | Polling, status line, sample data, boot |
