---
name: listing-photo-sync
description: Sync photo updates (remove bad photos like greenish-pool shots, add new exterior/pool photos, dedupe, or bulk-delete everything before/after a reference photo) across a property's listings on Airbnb, Vrbo, Zillow Rental Manager, and TurboTenant. Use when the user wants to add/remove/replace photos on an existing published listing across multiple platforms, delete a whole range of photos anchored to a named/shown photo (e.g. "delete everything after the X pic"), or asks to "clean up" or "update photos" for a property.
---

# listing-photo-sync skill

Automates bulk photo maintenance (delete bad photos, add new ones, verify no duplicates) across
the same physical property's listings on multiple platforms. Built 2026-08-27 from a live session
doing exactly this for three properties across Airbnb, Vrbo, Zillow, and TurboTenant — every
technique and quirk below was learned the hard way in that session; follow them to avoid repeating
the same expensive trial-and-error.

## Tools this skill uses

- `mcp__playwright__browser_*` — drive the real Chrome profile (same persistent automation profile
  as [[vrbo-listing]] / [[zillow-listing]] / [[turbotenant-listing]], set in
  `host-profile.yaml` → `browser.playwright_config`).
- `Bash` — `curl` to fetch thumbnail images for inspection, `python3`/PIL only when a stitched
  contact sheet is actually cheaper than direct fetches (see "Verifying photo content" below).
- `Read` — view a downloaded image or screenshot. This is the expensive step; minimize calls to it.

## Property IDs

Keep a small table of each property's ID per platform (Airbnb propertyId, Vrbo propertyId, Zillow
property key, TurboTenant listing id) in `property-ids.md` next to `host-profile.yaml`, and add to it
whenever you discover one. Re-derive if an ID 404s — sites do rotate IDs.

## Direct URLs — skip the menu-click chain

Once you have the property ID, go straight to the photo editor. Don't navigate
dashboard → property → edit → photos each time; that's 3-4 wasted round trips per property.

- **Airbnb**: `https://www.airbnb.com/hosting/listings/editor/{propertyId}/details/photo-tour`
  (room-by-room; "Additional photos" bucket at `.../photo-tour/additional-photos`)
- **Vrbo**: `https://www.vrbo.com/supply/pe/photos?propertyId={propertyId}` (flat "All photos" grid)
- **Zillow**: `https://www.zillow.com/rental-manager/properties/{propertyKey}/listing` then click
  the "Media" accordion header to expand it (no direct deep-link to the expanded section found)
- **TurboTenant**: `https://rental.turbotenant.com/owners/properties/edit/{listingId}` then click
  "EDIT" on the "Photos & video tour" card

Collect all property IDs for all platforms in one pass at the start (one properties-list snapshot
per platform), then work platform-by-platform, property-by-property using direct URLs. Don't
interleave discovery and action.

## Login walls

Airbnb and Vrbo sessions are normally already authenticated in the automation profile. Two exceptions to
watch for, both require a human — stop and ask, don't try to automate past them:
- **Vrbo/Expedia Partner Central** can show a full login wall on `expediapartnercentral.com` even
  when `vrbo.com/p/properties` itself is logged in — use the vrbo.com host, not expediapartnercentral.
- **Zillow** shows a "Press & Hold to confirm you are human" bot-check semi-randomly when opening a
  property's overview/listing page. This is real bot detection — never attempt to click through it
  programmatically. Ask the user to clear it in the browser window, then continue.

## Finding bad photos (e.g. greenish pool water) without burning vision calls

Priority order, cheapest first:

1. **Check for platform-generated alt-text first.** Vrbo auto-tags photos with AI-generated alt
   attributes including content keywords — critically, `"pool visible"` appears literally in the
   alt text of any photo showing a pool. Query it for free:
   ```js
   Array.from(document.querySelectorAll('img[alt*="pool visible"]')).map(i => i.src)
   ```
   This turns "check every photo" into "check the 2-5 photos already flagged as pool photos."
   Airbnb and Zillow/TurboTenant do not expose this — fall back to step 2 there.
   **Airbnb has no alt-text at all** (`img.alt` is always `""`), including for the text baked into
   an image's own pixels — e.g. host-uploaded "neighborhood attraction" promo cards (a designed
   graphic like "CASINO RESORT · 25 MINUTE DRIVE" with icon bullets) read as blank in the
   DOM. Airbnb does flag these (and other low-quality photos) with a visible "Needs attention"
   badge — find them for free via the DOM (see "Reference-anchored bulk deletion" below for the
   exact query), then `curl` + `Read` just those flagged thumbnails to confirm content by pixels.
2. **Fetch small thumbnails directly, don't screenshot-and-scroll the grid.** Get the `img.src`
   list via `document.querySelectorAll('img')`, filter to the CDN domain, `curl` each thumbnail
   URL (use the smallest size variant the CDN offers — e.g. Vrbo's `rw=334`, Zillow's
   `rentals_medium_500_500`, TurboTenant's `--380x380`) to the scratchpad dir, then `Read` them.
   Reading 20-30 small thumbnails individually is more reliable than one giant screenshot.
3. **Stitched contact sheets are a false economy unless the code is right the first time.** A PIL
   contact sheet turns N image-Reads into 1, but a single indexing bug (mismatched sort order
   between the download loop and the sheet-assembly loop) silently mislabels every photo and costs
   a full debug-and-redo cycle — worse than just doing N individual Reads. If you use a contact
   sheet, sort the *same* pre-built file list once and reuse that exact list object for both the
   download loop and the sheet loop; never re-glob between steps.
4. **When in doubt about one specific photo, fetch it directly by its exact CDN hash/URL and Read
   just that one** rather than re-generating a whole sheet. Cheaper and unambiguous.
5. Never trust a filename or grid position to identify a photo without having actually looked at
   the pixels behind that specific URL at least once — position-based assumptions caused a real
   mistake in the source session (see "Costly mistakes to avoid").

## Deleting photos

General pattern: click the photo's menu/kebab (may need a "More Details" or "..." button first),
click "Delete", confirm in a dialog if one appears. Platform-specific gotchas:

- **Vrbo**: the left nav drawer intercepts clicks anywhere near the top of the viewport, even on
  elements that are visually clear of it — you'll get a `TimeoutError` claiming the drawer
  "intercepts pointer events." Fix once per session: click the hamburger/"Supply navigation header
  toggle" button to collapse the sidebar before doing any delete/upload clicks. After that, real
  `browser_click` calls work normally; you don't need JS-click workarounds once it's collapsed.
- **TurboTenant — critical, causes silent partial failure**: the delete (trash) button is
  `button._9Su6Y` inside a `.t6OQG` photo-cell container. **Firing multiple deletes in one
  `browser_evaluate` call only actually deletes one of them** — React batches the state updates
  within a single JS task, so N synchronous `.click()` calls all read the same stale array and the
  net effect is -1, not -N. You MUST issue one delete per separate `browser_evaluate` tool call
  (i.e., one per model turn / round trip), re-querying `.t6OQG` fresh each time. Always verify the
  count after a batch of deletes — don't assume N clicks removed N photos.
- **Airbnb**: two different UIs exist and they're not interchangeable —
  - *Room-bucket photo tour* (`.../details/photo-tour`): per-photo "Open remove and delete options"
    → "Delete from listing" (not "Remove from room or space", which just unassigns it from that
    room without deleting) → confirm in the "Permanently delete" dialog. Fine for one-off deletes.
  - *Flat "All photos" grid* (`.../details/photo-tour/photos`): supports real multi-select bulk
    delete — see "Reference-anchored bulk deletion" below. Use this one for anything beyond ~3
    photos; the room-bucket UI has no multi-select and doing N deletes there is N round trips.
- **Zillow**: click the photo to open "Edit photo" modal, then `[data-testid="PhotoDetailsModal-
  DeletePhotoButton"]` — no confirmation dialog, it deletes immediately on click.

## Reference-anchored bulk deletion — "delete everything after/before photo X"

A distinct task shape from "delete the bad ones": the user names or shows one photo and wants
everything on one side of it gone, in whatever order the grid currently shows. Confirmed working
end-to-end on Airbnb 2026-09-01; other platforms not yet exercised for this specific pattern — the
position-reconstruction technique (step 2) should transfer, but re-confirm the selection/delete
mechanics live before assuming they match.

1. **Get to the flat grid, not the room-bucket view.** Airbnb:
   `https://www.airbnb.com/hosting/listings/editor/{propertyId}/details/photo-tour`, click **"All
   photos"** (top right) to reach `.../details/photo-tour/photos`. Two promo modals ("Improve your
   photo tour", "Lead with your best photos") can pop up on these pages — click **"No thanks"** on
   each; don't click "Arrange photos"/"Let's go", which reorders the grid out from under you.
2. **Reconstruct true grid order from live positions — the grid is not virtualized.** All photos
   render in the DOM at once (confirmed up to ~80 photos) even though they sit in a scrollable
   container, so one `browser_evaluate` gets everything without scrolling:
   ```js
   () => {
     const seen = new Map();
     document.querySelectorAll('img').forEach(img => {
       const m = img.src.match(/original\/([a-f0-9-]{20,})\.(png|jpe?g)/i);
       if (!m) return;
       const r = img.getBoundingClientRect();
       if (r.width < 30) return; // skip avatar/icon imgs
       if (!seen.has(m[1]) || seen.get(m[1]).w < r.width) seen.set(m[1], {hash:m[1], top:Math.round(r.top), left:Math.round(r.left), w:Math.round(r.width)});
     });
     return Array.from(seen.values()).sort((a,b) => a.top - b.top || a.left - b.left);
   }
   ```
   Sorting by `(top, left)` gives the actual row-major grid order. Never assume DOM index order
   matches visual order — always sort by rect, not by `querySelectorAll` insertion order.
3. **Locate the reference photo.** If the user gave a screenshot, don't trust its crop position
   (it may be scrolled past earlier rows you haven't seen) — find the photo on the *live* page
   instead. If it's one of Airbnb's "Needs attention"-flagged photos (common for host-uploaded
   graphic/text cards, which have no alt-text to search — see "Finding bad photos" above), locate
   the flags first, then walk up from each to the nearest `<img>`:
   ```js
   () => {
     const matches = Array.from(document.querySelectorAll('*')).filter(el => /needs attention/i.test(el.textContent||''));
     const innermost = matches.filter(el => !matches.some(o => o !== el && el.contains(o)));
     return innermost.map(el => {
       let node = el, img = null;
       for (let i=0;i<10 && node;i++) { img = node.querySelector?.('img'); if (img) break; node = node.parentElement; }
       return {img: img?.src};
     });
   }
   ```
   `curl` the small size variant of each candidate to the scratchpad and `Read` it to confirm which
   one is the named photo — don't guess from position alone.
4. **Compute the target set** from the sorted array in step 2: everything with `top` greater than
   the reference's `top` (or same `top` and greater `left`) for "after"; the inverse for "before".
5. **Enter multi-select and click each target.** Click **"Manage photos"** (top right of the "All
   photos" page) to enter "Select photos" mode — this overlays room-tag labels and turns each photo
   tile into a toggle. For each target photo, find the nearest `<button>` ancestor of its `<img>`
   and click it:
   ```js
   async () => {
     const hashes = [/* target hashes from step 4 */];
     for (const h of hashes) {
       const img = document.querySelector(`img[src*="${h}"]`);
       let el = img; for (let i=0;i<6;i++) { if (el.tagName === 'BUTTON') break; el = el.parentElement; }
       el?.click();
       await new Promise(r => setTimeout(r, 120)); // see note below — do not remove
     }
     return document.body.innerText.match(/(\d+) selected/)?.[0];
   }
   ```
   **The `setTimeout` delay between clicks is load-bearing.** This is a different failure mode than
   TurboTenant's stale-closure delete bug (see "Deleting photos" above) — here the risk is React
   event-batching swallowing rapid synchronous clicks in the same task. A real per-click delay (not
   a tight loop) avoids it; this was verified against the "N selected" header count matching the
   exact target count (34/34) before proceeding. Confirm the header count yourself before deleting
   — don't trust that N clicks produced N selections.
6. **Delete and verify the confirmation dialog's own count.** The bulk-delete button is
   `button[aria-label="Delete"]` in the top toolbar (next to `button[aria-label="Move photos"]`).
   Clicking it opens a dialog reading "Permanently delete N photos from your listing?" — **check
   that N matches your computed target count before clicking the confirm "Delete" button inside
   it.** A mismatch means your selection step missed or double-hit something; stop and recheck
   rather than confirming.
7. **Verify the end state**, not just that clicks happened: re-run the step-2 query, confirm the
   new total equals `original - deleted`, and confirm the reference photo's hash is still present
   (i.e. you didn't delete the anchor itself) plus that one deleted hash is now absent.

## Uploading photos

- Trigger the hidden `input[type=file]` directly via `input.click()` in `browser_evaluate`, or
  click the visible "Add photos"/"Click or drag to upload" control — either can work, but **if the
  first click produces no visible change after a reasonable wait, do not assume it silently
  failed and retry** — it may have actually succeeded and just not synced state yet (this cost a
  duplicate-upload cleanup in the source session, see below). Check the actual photo count before
  retrying an upload.
- Multi-file `browser_file_upload` with 20-30 files at once works on all four platforms tested, but
  large originals (5-15MB each) take 20-40s to fully register. Poll the count rather than fixed-wait
  once, and don't fire the same upload twice out of impatience.
- **TurboTenant has a hard cap of 20 photos per listing**, enforced with a red warning. If a
  listing is already at or near 20, you cannot just add new photos — you must delete some first
  (see "Making room" below).

## Making room on a capped listing (TurboTenant)

When at 20/20, identify near-duplicate angles of the same room (e.g. 4-6 shots of one kitchen or
living room from slightly different positions) and cut the weakest 2-4, keeping the best 2 per
room. This is a judgment call — ask the user once per listing type ("swap redundant interior shots
for new exterior photos, or skip adding exterior photos here?") rather than deciding unilaterally
and silently deleting their content; get the "yes, swap redundant angles" confirmation once, then
apply the same policy to sibling properties without re-asking.

## Verifying no duplicates — do this with JS, never with your eyes

After any upload, confirm the new count and check for exact-duplicate CDN hashes for free:
```js
() => {
  const imgs = Array.from(document.querySelectorAll('img'));
  const set = new Set();
  imgs.forEach(i => { const m = i.src.match(/([a-f0-9]{8,32})\.(jpg|webp)/); if (m) set.add(m[1]); });
  return set.size; // compare to expected total
}
```
If the unique count is less than expected, something double-uploaded — find and remove the extras
(see "Costly mistakes" below for exactly this happening and how it was fixed) rather than shipping
a listing with duplicate photos.

## Costly mistakes to avoid (both happened in the source session)

1. **Don't identify a photo to delete by DOM position/index after any prior deletion or reload.**
   A stale JS closure (`imgs[0].closest(...)`) targeted the wrong photo — a live-room "Cover" photo
   instead of the intended greenish pool photo — because the grid had already re-rendered once
   between calls. Always re-query fresh (`document.querySelectorAll(...)` at the moment of the
   click) and, for anything irreversible, confirm via a screenshot of the actual delete-confirm
   dialog *before* clicking the final "Delete" — every platform shows the photo being deleted in
   that dialog. If you catch a wrong deletion: check if the local source folder for that property
   still has the same photo (filenames often match, e.g. `unit-a (14).jpg`) and re-upload it rather
   than leaving the listing short a photo.
2. **A single `input.click()` upload attempt can silently succeed later than expected.** In the
   source session, retrying an upload that "didn't seem to work" after a 20s wait resulted in the
   same 9 files uploading twice (18 blob entries instead of 9). Always re-check the count with a
   longer wait or an explicit poll before concluding an upload failed and re-triggering it.

## Screenshots and temp files — the scratchpad dir is NOT reachable from this tool

`mcp__playwright__browser_take_screenshot` / `browser_snapshot`'s `filename` param is sandboxed to
its own allowed roots — confirmed 2026-09-01 as `~/.playwright-mcp` and the user's home dir only.
An absolute path into the session scratchpad (e.g. under `/private/tmp/...`) is
**rejected outright** ("File access denied ... outside allowed roots"), it doesn't just silently
resolve relative to home. Pass a path under `~/.playwright-mcp/` (its default output dir — bare
relative filenames already land there) or another absolute path under `~/`; either way
don't let clutter accumulate loose in `~/` itself. `Read` the saved file afterward same
as any screenshot. This restriction is specific to the playwright MCP's own tools — `Bash`
(`curl`, PIL, etc.) has no such limit and should still target the scratchpad dir as normal.

## Suggested workflow order for a "sync photos across all platforms" request

1. Confirm with the user: which properties, what to remove (e.g. greenish pool), what to add (new
   photo folder), and whether "Airbnb only" or "all platforms."
2. Do one platform fully across all properties before moving to the next — you'll have the
   direct-URL pattern and any login/bot-check friction resolved by the second property, making the
   third nearly free.
3. Within each property: find bad photos (alt-text or thumbnail scan) → delete → verify count →
   upload new photos → verify count and dedupe → save/exit.
4. Give one consolidated summary at the end covering all platforms × properties, not a running
   commentary after each one — the user wants the final state, not a play-by-play.

**Large batches (roughly 5+ properties in one request):** consider splitting into one session/
subagent per property (or small property group) instead of one long session for the whole batch.
The "one platform fully before switching" ordering above is still the right shape *within* a
session — it amortizes login/nav discovery — but a single session covering a large batch pays for
every earlier property's transcript on every later one. Weigh that against the rediscovery cost
before deciding; for 2-4 properties one session is usually still cheaper overall.
