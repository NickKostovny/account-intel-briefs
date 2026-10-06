# account-intel-briefs

Stage 5 of 5 in Invert's account-intelligence pipeline. Renders one static HTML brief per account plus an index, and deploys them behind Vercel SSO. Upstream: every other stage. Canonical code: [account-intelligence](https://github.com/NickKostovny/account-intelligence).

Live: https://account-intel-invert.vercel.app (Invert Vercel team SSO, or Nick's `_vercel_share` link). Build date on the live board: 2026-09-30.

## Why it looks the way it does

The reps rejected prose. The first briefs had hypothesis paragraphs; the rebuild rule became "no paragraphs": every former prose block is bullets, chips, a ledger, a timeline or a meter, and prose is admissible only as quoted evidence. The build asserts it (the prose guard: zero `<p>` tags, every `<blockquote>` inside `.ev-src`).

On 2026-09-10 Nick showed the system to the head of sales. The build had optimised for evidence integrity and org-map depth on two accounts, so a rep opening any other brief met a wall of zeros. The layout flipped to lead with the complete layers, and Nick added a second rule the same day: **no explanatory text on the page.** No legends, footers, coverage notes, map hints, filter help or source labels. The page carries the header with "generated <date>", stat chips, tables, noun section titles and one numeric summary line. Method notes live in the repos, not on the page.

## What a brief shows, in order

1. Header, "generated <date>", link to all accounts.
2. Stat chips: pipeline status, deal owner, HQ, revenue, sites and KEEP count, named heads, LinkedIn-verified, publications; requisitions and org units when present; digital/IT location; news triggers; sites by modality.
3. **Sites, KEEP first.** Columns: Site, City, Verdict (badge; hover shows the fit reason), Fit (verdict, reason, source link; basis in the title attribute), Named site head (LinkedIn link, Clay title, "matched on LinkedIn <date>", else the site-map prose labelled "site-map evidence"), Functions, Modality, Hiring (only on sites that make or develop product), News, and Reqs, Units, Coverage on swept accounts.
4. Publications, talks and news (relevance band when present).
5. The posting layer: coverage bar, gap chips, pins, org map, corpus query, digital/IT ledger. Folded inside `<details>` unless the account has at least one observed org unit; the summary says "not swept yet" or "N requisitions indexed, no teams reconstructed yet".
6. Acquired-company lineage.
7. Hiring fold (collapsed; it was 79 to 83 percent "unclassified" and read as noise).

The index groups Live pipeline (Active Pipeline, Active Prospect, Active Partnership, Closed Won), No deal yet, Closed lost, sorted by KEEP-site count. Each card: status, owner, KEEP n of total, named heads, LinkedIn count, publications, plus reqs and units when present.

## Files

| File | Role |
|---|---|
| `build-briefs.py` | renders `briefs/<account>.html`, `briefs/index.html`, `briefs/jobs-<account>.json`. `--account <aid>` for one. `--allow-prose` bypasses the guard (do not). |
| `orgmap.py` | layout and SVG for the org map (copy of the canonical one). |
| `stage_deploy.py` | copies `briefs/` to `deploy/`, writes `vercel.json` (cleanUrls false because briefs cross-link by `<account>.html`, noindex and no-store headers), re-points `deploy/.vercel`. `--check <url>` fails unless the unauthenticated fetch lands on the SSO redirect with none of the confidential markers in the full body. `--check-all` asserts the `-invert` alias is SSO and `-two` is 404. |
| `deploy/vercel.json` | the headers and cleanUrls setting. |
| `launch.json.example` | the two local preview entries (`account-intel` on port 8830, `account-intel-map` on 8831) for `.claude/launch.json`. |
| `aiq.py` | shared library copy. See `SHARED.md`. |

## Run and deploy

```bash
python3 build-briefs.py
python3 stage_deploy.py
cd deploy && /opt/homebrew/bin/vercel deploy --yes --scope invert && cd ..
/opt/homebrew/bin/vercel alias set <hash-url> account-intel-invert.vercel.app --scope invert
python3 stage_deploy.py --check-all
```

Local preview: `python3 -m http.server 8830 --directory briefs`.

Verify in a browser: the index generated date is today; AstraZeneca shows stat chips, Sites table KEEP first, Fit cells with source links, Named site head LinkedIn links, Functions and Modality chips, a News column; the posting layer fold opens; the org map toggles change the map, not only the control.

## The deploy rules and why

- **Never `vercel --prod`.** On 2026-09-10 Vercel assigned the project a default production alias (`account-intel-two.vercel.app`, because `account-intel.vercel.app` was taken). That default alias sits outside Vercel Authentication on this plan and served the briefs at HTTP 200 unauthenticated for about 40 minutes on a hostname nobody had been given. Caught by reading `targets.production.alias` via `vercel api /v9/projects/<id>`; removed with `vercel alias rm`. Deploy a preview, set the alias, run `--check-all`.
- **Create a project empty and prove the gate before content goes up.** An earlier attempt on a new project returned 200 with no SSO wall for about 4 minutes. Protection `all` is rejected on this plan for production deployments, so preview only.
- **Read the whole body in the check.** The first 20 KB is CSS, which once let a leak check pass that should have failed. `--check` also strips the login page's `next=` URL echo before scanning, or a marker that is also a path slug false-positives.
- **Never share `account-intel.vercel.app`.** It belongs to an unrelated third party.
- **Shareable link.** The Vercel MCP cannot mint one for this project and the harness blocks the protection-bypass PATCH, so Nick creates it himself: `echo '{}' | vercel api -X PATCH "/aliases/account-intel-invert.vercel.app/protection-bypass?teamId=<team>" --input -`. Revoke with `{"revoke":{"secret":"<secret>","regenerate":false}}`. Anyone holding the link can read the briefs.
- Project: `invert/account-intel`. Viewers must be members of the Invert Vercel team (SAML), or Nick presents it.

## Brief internals that must not break

- **DOM-order contract.** The radios, gap inputs and `#f-supports` must stay siblings of `.map` at `.wrap` level and precede it, even when the posting layer sits inside `<details>`. Every toggle is `#id:checked ~ .map` CSS with no JS. `.ev-empty` stays the last child of `.dock`. `%(js)s` stays after all markup.
- Layout is computed in Python and baked into SVG. No CDN, no layout library. Works with JS off.
- Provenance is `:target` driven and deep-linkable: `astrazeneca.html#ev-<unit_id>` lands a colleague on the exact node.
- The judgment overlay lives in `localStorage` under `aiq:v1:<account>` (pin, confirm, reject, note) with content-addressed refs, an export path to `org_overrides.json`, and a reconciliation prompt (default keep) for refs that no longer resolve.
- Never-scanned sites get a hatched full-length bar (an absent bar would read "small"); scanned-and-empty gets a 3px tick. Unknown tech renders a dashed `stack ?` chip. A parent named but not reconstructed becomes a ghost node that takes layout space.
- Label vocabulary: `NO REPORTING LINE FOUND IN ANY POSTING · N TEAMS`, `INFERRED — NAMED IN A POSTING ONLY`, `MEDIUM CONFIDENCE · 2 postings, 2 verified quotes`, `TEAM` not `ORG UNIT`.
- `named_site_head` placeholders (`PENDING-CLAY`, `n/a`, "Not identified", "None surfaced") read as empty via `head_text()`. A literal `None` in `deal_owner` is null.
- Model name and p-values appear only in title attributes.
- Two pure-CSS toggles once shipped inert because their `:checked ~` rules had no matching sibling (a zoom, removed; gap chips, fixed by hoisting the inputs). A toggle needs a browser assertion that the target changed, not only the control.

## Open

- Any brief change ships only after `--check-all` passes.
- The share link has no expiry; rotate it if it spreads.

## Data

Not in this repo. The briefs themselves are generated output and are not committed either; they name real people and deal status.
