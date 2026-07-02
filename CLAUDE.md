# World Cup Draft Board — Project Guide

Static site showing a fantasy football league's draft order, driven by 2026 World Cup knockout results. Each league member was randomly assigned a national team; draft position is determined by how far that team goes. **Champion drafts #1; first team eliminated drafts #10.**

Live at: `https://worldcup.jordanhorrillo.com` (GitHub Pages, `main` branch root, custom domain via `CNAME` file). Page is `noindex` on purpose — do not remove the robots meta tags, and do not link to this site from anywhere else.

## Files

- `index.html` — the entire app. Self-contained (no build step, no dependencies beyond Google Fonts). Renders four sections: Draft Order, radial Bracket, Upcoming Matches, Potential Outcomes. **All four render from one data object; you should almost never need to edit this file.**
- `results.json` — the only file that changes during the tournament. The page fetches it on load with cache-busting (`?t=` + timestamp, `cache:no-store`) and falls back to a data snapshot embedded in `index.html` if the fetch fails.
- `CNAME` — contains `worldcup.jordanhorrillo.com`. Do not delete; GitHub Pages drops the custom domain if this file disappears from the branch.

## The routine task: updating results

When a match finishes, edit `results.json`, commit, push. Live in ~30s. Nothing else.

### results.json schema

```json
{
  "snapshot": "July 1, 2026",          // shown in the page footer as "Updated: ..."
  "results": {
    "r32":  [ /* 16 entries */ ],       // winner codes of the 16 R32 matches, null = not played
    "r16":  [ /* 8 entries  */ ],
    "qf":   [ /* 4 entries  */ ],
    "sf":   [ /* 2 entries  */ ],
    "champ":[ /* 1 entry    */ ],
    "elimOrder": ["GER","NED"]          // LEAGUE teams only, in real-world elimination order
  },
  "matches": [ /* upcoming league-team matches, see below */ ]
}
```

**Winner arrays.** Index `i` of `r32` is the winner of R32 match `i`. Match `i` at any level pairs entries `2i` and `2i+1` of the level below (level below r32 is the fixed `ORDER` array in index.html — the real bracket's 32 teams top-to-bottom). Winner arrays cascade: `r16[j]` pairs `r32[2j]` vs `r32[2j+1]`, etc. Use `null` for undecided. Team codes are 3-letter (FIFA-style) and must match the `TEAM` map keys in index.html.

**elimOrder.** Only the 10 league-assigned teams, appended in the order they are eliminated in real life (same-day: by kickoff/finish order). This alone determines draft picks 10, 9, 8, ... The champion (or last league team standing) gets the best remaining pick automatically. Do NOT put non-league teams here.

**matches array.** Only upcoming/live matches involving a still-alive league team. Fields:

```json
{"round":"R16","day":"Saturday · July 4","time":"10:00 AM PT","venue":"NRG Stadium · Houston",
 "tv":"FOX","live":false,"L":"MAR","person":"Jason","O":"CAN","p":76}
```

- `L` = league team code, `person` = owner's name, `O` = opponent code, `p` = league team's implied % chance to ADVANCE (0–100 integer; converting from moneyline/decimal odds is fine, strip the vig roughly).
- `round` accepts R32, R16, QF, SF, F. `live:true` adds a red "Today" tag.
- **All times Pacific**, labeled `PT` (not PST — it's daylight time in summer; the owner approved "PT").
- Remove matches once played; remove a member's matches once their team is out.

### After every update, verify

1. `results.json` is valid JSON (a syntax error silently drops the site to its embedded fallback — the page will render, just with stale data, which is worse than an error).
2. Winner codes appear in the correct array **index** (wrong index = winner drawn on the wrong line of the bracket).
3. Any newly eliminated league team is appended to `elimOrder`.
4. Load the live URL and check the footer shows the new `snapshot` string — that confirms the fetch, not the fallback, is rendering.

## League assignments (fixed)

Adam–France, Cory–Spain, Alex–Argentina, Chris–England, Daniel–Portugal, Jordan–Brazil, David–Netherlands, Jason–Morocco, Gordy–Belgium, Scott–Germany.

State as of July 1, 2026: Germany (Scott) and Netherlands (David) eliminated in the R32 → picks 10 and 9. France, England, Brazil, Morocco through to R16. Spain, Portugal, Argentina, Belgium had R32 matches pending.

## How index.html works (only if you must touch it)

Everything derives from the data object `D`:
- `buildLevels()` cascades winner arrays into `levelCodes[0..5]` and computes node angles for the radial layout (level 0 = outer ring of 32, level 5 = center/champion).
- `buildSVG()` draws the bracket: right-angle connectors (radial spokes + concentric arcs), gold-ringed flags + gold owner names (18px) for league teams, red ✕ overlays on eliminated league teams at their deepest node, winner paths animate in (class `lit`).
- `draftRows()` computes picks: reversed elimOrder tail + champion/last-survivor head + "Still in" placeholders.
- `buildOutcomes()` per alive member: next opponent (sibling node, or "X / Y" if undecided), advance odds from `matches`, earliest possible league-vs-league clash (`meetLevel()` — walks level-0 indices up by halving until they collide), and a 1–10 pick strip with dead picks ✕'d.
- The embedded `FALLBACK` object mirrors results.json. If you make a significant data update, consider syncing FALLBACK too (not required — it only shows if the fetch fails).

Design system: dark `#09090b` base, gold `#e8b64c` accents, Barlow Condensed for display type, Inter for body. Atmosphere layers (halo/grain/vignette/motes) are `position:fixed` behind content; all animation respects `prefers-reduced-motion`. Keep copy terse — the owner explicitly stripped verbose copy; don't add explanatory text.

## Known judgment calls (already decided — don't relitigate)

- Same-day eliminations ordered by kickoff time (Germany before Netherlands on June 29).
- If two league teams both reach the semifinals and lose, the third-place playoff result should determine their order in `elimOrder` (loser of the 3rd-place match is "out" later... actually: append them by when their elimination became final — semifinal losses — unless the owner says the 3rd-place game counts; ASK if this arises).
- "PT" labeling over "PST".
- Site stays noindex, unlinked from jordanhorrillo.com.

## What the owner may ask for

- "X beat Y" / "update the results" → edit results.json per above, commit, push, verify live.
- Odds refresh → update `p` values in matches (search current market odds, convert to implied advance %).
- If a claude.ai scheduled task hands over a regenerated results.json, diff it against the current one for sanity (schema intact, elimOrder only appended-to) before committing.
