---
name: listing-photo-order
description: Rearrange an existing listing's photos into room-grouped, non-randomized order and apply whatever room/category labels the platform offers, given a URL to the photos page or the property page on Airbnb, Vrbo, Zillow Rental Manager, or TurboTenant. Use when the user wants an already-published listing's photos reordered, grouped by room, or labeled — not for adding/removing/replacing photos (see listing-photo-sync) and not for new-listing onboarding.
---

# listing-photo-order skill

Reorders an existing, already-published listing's photo set so it reads as one clean room-by-room
walkthrough instead of a randomized grid, and applies room/category labels wherever the platform
exposes them. Sibling to [[listing-photo-sync]] (which adds/removes/dedupes) — this skill never
adds or deletes a photo, it only reorders and labels what's already there. Also distinct from the
onboarding skills ([[vrbo-listing]], [[zillow-listing]], [[turbotenant-listing]]) — this is post-publish tidy-up from a bare URL, not a wizard.

**Input is a single URL** to either the photos/media page or the general property page for one
listing on one platform. Detect the platform from the URL's host and route from there (see
"Routing from the given URL" below). If the user gives a property page URL, navigate in-app to the
photo editor rather than guessing a direct URL blind — capture the real direct URL along the way so
later runs on the same property can skip straight there (mirror [[listing-photo-sync]]'s "Direct
URLs — skip the menu-click chain" table; add confirmed IDs to that table when you find them).

## Tools this skill uses

- `mcp__playwright__browser_*` — drive the real Chrome profile (same persistent automation profile
  as the other listing skills).
- `Bash` — `curl` small thumbnail variants for inspection (see [[listing-photo-sync]]'s "Finding bad
  photos... without burning vision calls" section — the same technique applies here for room
  classification, not just defect-finding).
- `Read` — view a downloaded thumbnail or screenshot. Minimize calls; see below.
- `AskUserQuestion` — only when a photo's room genuinely can't be determined (see "When to ask"
  below), or a platform's reorder/label mechanism doesn't match what's documented here.

## Standing assumptions

- **Login:** assume the browser session is already authenticated in this profile (same as the
  sibling skills). Stop and ask only on an actual login wall or bot-check — see
  [[listing-photo-sync]]'s "Login walls" section, which applies unchanged here.
- **Autosave:** all four platforms' photo editors save reorders/labels as you go — there's no
  separate "Publish" gate for a pure reorder the way there is for a new listing. No mid-task
  checkpoint is needed; just give one consolidated summary at the end (what order was applied, what
  got labeled, anything left ambiguous).
- **Scope:** reorder and label only. Never delete, add, or replace a photo in this skill — if the
  user also wants photos swapped in/out, that's [[listing-photo-sync]]; do both in the same session
  if asked, but treat them as two separate passes (sync first, then order, so you're ordering the
  final set once rather than reordering twice).

## Routing from the given URL

| Platform | Host pattern | Photos/media direct URL | If given a property page instead |
|---|---|---|---|
| Airbnb | `airbnb.com` | `https://www.airbnb.com/hosting/listings/editor/{propertyId}/details/photo-tour` | Guest-facing `airbnb.com/rooms/{id}` and the host editor use the same numeric id — substitute it into the editor URL directly. From a dashboard/overview page, grab the id from the page URL or the "Photo tour" nav item. |
| Vrbo | `vrbo.com` | `https://www.vrbo.com/supply/pe/photos?propertyId={propertyId}` | From `vrbo.com/p/properties/...` or a dashboard row, extract `propertyId` and build the URL directly. |
| Zillow Rental Manager | `zillow.com` | No confirmed direct deep-link to the expanded Media section yet — go to `https://www.zillow.com/rental-manager/properties/{propertyKey}/listing` and click the "Media" accordion header to expand it. | Extract `propertyKey` from the property page URL. |
| TurboTenant | `rental.turbotenant.com` | `https://rental.turbotenant.com/owners/properties/edit/{listingId}` then click "EDIT" on the "Photos & video tour" card | Extract `listingId` from the property edit page URL. |

Known property IDs live in `property-ids.md` (see [[listing-photo-sync]]) — reuse them instead of
re-discovering if the user names one of those properties; re-derive if they 404 (IDs do rotate).

## The core ordering rule (applies to every platform)

**Every room/area's photos form one unbroken block — a room's photos are never split across two
separate points in the final sequence.** A sequence that goes kitchen → exterior → bathroom →
*more kitchen* is wrong. Once the walkthrough has moved past a room, it never comes back. This is
the same rule [[vrbo-listing]]'s "Photo selection & ordering" section already documents for new
listings; apply the identical algorithm here to an existing, already-live photo set:

1. **Classify every photo into a room/area first**, before touching order. See "Classifying
   existing photos into rooms" below — don't interleave classification and reordering.
2. **Pick the lead photo(s)** — the single most inviting shot (or, on Airbnb, the 4 that make the
   cover grid — see "Airbnb specifics" below). Actively compare candidates; don't default to
   whatever currently sits first.
3. **The lead's room goes first**, its remaining photos grouped immediately behind it.
4. **Fill out any remaining "best of" lead-in slots** (Vrbo's best-4 concept, Airbnb's 4-photo cover
   grid) with the next-best standout shots — these can span a couple of rooms since it's a
   highlights reel, but still pull only each contributing room's single best shot into the lead-in;
   the rest of that room's photos stay together in its own block later, not scattered.
5. **Then walk the property room by room** in the order a guest would actually encounter it:
   exterior/curb appeal → main living areas (living room, kitchen, dining) → each bedroom in turn →
   each bathroom → outdoor/amenity spaces (pool, patio, yard) → misc. Each room a single contiguous
   block, no repeats.
6. **Leave any platform-generated, non-host photos exactly where the platform put them** — see
   "Airbnb specifics" for the neighborhood-attraction photos. Don't fold auto-generated content into
   the room-block logic; it isn't a room.

### When to ask

If a photo's room genuinely can't be determined (ambiguous framing, cropped-out context, could be
either bathroom), don't guess — use `AskUserQuestion` or describe it to the user and ask, the same
standing convention as the onboarding skills. Don't silently misfile a photo into a plausible-but-
wrong room block.

## Classifying existing photos into rooms

The listing is already live, so there's no filename or upload-batch context to lean on — every
photo's room has to be determined from the pixels (or from a platform's existing per-photo room
metadata, if one exists — check first, it's free):

1. **Check for existing room structure before doing any vision work.** Airbnb's photo-tour editor
   already organizes uploaded photos into per-room "space" buckets (Bedroom 1, Kitchen, Bathroom,
   etc.) — that grouping is ground truth; the work there is mostly about the *order of the room
   sections* and the lead-in grid, not re-classifying individual photos. Vrbo/Zillow/TurboTenant's
   photo pages are flat grids with no inherent per-photo room metadata as far as observed — confirm
   live each time rather than assuming, and update this section once confirmed.
2. **Check for AI alt-text hints next**, same as [[listing-photo-sync]]'s pool-detection trick —
   Vrbo's `img[alt*="..."]` attributes carry content keywords for free; query before looking at
   pixels.
3. **Fetch small thumbnails and `Read` them**, same technique and same CDN-thumbnail-size guidance
   as [[listing-photo-sync]] — cheaper and more reliable than one giant scrolled screenshot. Build a
   simple photo-url → room table once per run; don't re-derive a photo's room after you've already
   looked at it.
4. Stitched contact-sheet caution applies identically here — see [[listing-photo-sync]]'s warning
   about index-mismatch bugs; only use a contact sheet if you keep one single sorted file-list object
   for both the download and the sheet-assembly steps.

## Applying labels — check what actually exists before assuming

None of the sibling skills document per-photo labeling in detail, so confirm live on each platform
and update this list once verified rather than assuming a mechanism that isn't there:

- **Airbnb**: the room-tour editor's per-photo "space" assignment (Bedroom 1, Kitchen, Bathroom,
  Living room, Outdoor, etc.) *is* the label — assigning a photo to the correct room bucket both
  orders and labels it in one action. Check whether individual photos within a room can carry a
  further caption/tag beyond the room assignment; apply if present.
- **Vrbo**: check the photo editor for a caption/tag field per photo (not confirmed as of this
  writing). If present, use short factual captions matching the room ("Primary bedroom", "Kitchen")
  rather than marketing copy.
- **Zillow**: Rental Manager's Media section is a documented gap in [[zillow-listing]] (never
  observed in the source recordings) — when you actually open it, check for a per-photo room-
  category dropdown (common in real-estate listing tools: Kitchen/Bathroom/Bedroom/Exterior/etc.)
  and apply it if present.
- **TurboTenant**: [[turbotenant-listing]] confirms a drag-to-reorder grid with a single **FEATURED**
  slot (first position) but no per-photo labels observed. Check for one when you're in there; if
  none exists, ordering is the only lever available on this platform.

Only apply a label the platform's UI actually offers for that photo — don't force a room name into
a free-text field that isn't meant for it, and don't label a photo you're not confident about (see
"When to ask" above — the same ambiguity rule covers labels, not just placement).

## Airbnb specifics

- **Cover = a 4-photo grid, not a single image.** The lead-in for the listing card is 4 photos
  shown together, not one hero shot. Curate the 4 best/most inviting photos for these slots per
  step 4 of the core ordering rule above — verify live exactly how Airbnb lets you designate the
  grid (an explicit "choose cover photos" picker vs. simply the first 4 in sequence) and document
  the confirmed mechanism here once you've seen it.
- **The last 3 photos in the "All photos" view are Airbnb-generated nearby-attraction images**, not
  host content (e.g. auto-pulled shots of a nearby landmark/resort). **Leave these exactly where
  Airbnb puts them — always last, always 3, never folded into a room block, never reordered or
  relabeled.** Confirm you're looking at the real attraction photos (not a host-uploaded exterior
  shot that happens to sit last) before treating them as untouchable — check their alt text/source
  or ask if it's unclear which photos these are.

## Reordering mechanics — confirm live, document once verified

None of the existing skills document manual drag-to-reorder for an already-live listing in detail
(the onboarding skills cover upload-time ordering; [[listing-photo-sync]] covers delete/add, not
reorder). Expect drag-and-drop (`mcp__playwright__browser_drag`) or a numbered "move to position"
control — whichever the live UI actually offers — and update this section with the confirmed
mechanism, selector, and any quirks (e.g. TurboTenant's known one-action-per-`browser_evaluate`-call
limitation on deletes may have an analogous batching quirk on drag reorders; verify rather than
assume it doesn't).

**Vrbo — confirmed 2026-09-01, full room-order emulation (one property matched to a sibling property's room order):**

- The grid uses **dnd-kit** (each tile is a `div[role="button"][aria-roledescription="sortable"]`,
  two levels above the `<figure>`; look for `cursor: grab` / `touch-action: none` in its inline
  style as a tell). It is *not* HTML5 native drag (no `draggable` attribute anywhere) and not
  react-beautiful-dnd despite a superficially similar "Draggable item X was dropped over droppable
  area Y" status message — don't go looking for `data-rbd-*` attributes, they don't exist here.
- **Selector pitfall:** the obvious `button:has(img[src*="HASH"])` or a class-based `:has()`
  selector resolves to **two** elements per photo — the real grid tile *and* the tile's own "More
  Details" kebab menu, which apparently embeds a duplicate reference to the same figure/img. Scope
  every selector to the grid root to disambiguate:
  `[data-testid="photo-grid"] div[aria-roledescription="sortable"]:has(img[src*="HASH"])` — this
  `data-testid="photo-grid"` container is unique (unlike the `.photo-grid` *class*, which matches
  several small unrelated sub-grids elsewhere on the page — don't use the class as a scope).
- **`browser_drag`'s drop semantics are "insert before the end target."** Dragging tile A onto tile
  B moves A to sit immediately before B (confirmed: dragging a pool-aerial shot onto the current
  cover tile made the pool shot the new position 1, pushing the old cover to position 2).
- **This selector survives reorders** (it's re-resolved fresh by Playwright on each call from the
  live DOM, unlike a `browser_snapshot` `ref` which goes stale the moment any element moves) — so a
  large multi-item reorder can be done as a plain sequence of `browser_drag` calls keyed by CDN
  hash, with no re-snapshot needed between them. For an N-item reorder, walk the *target* order
  left-to-right from position 2 (position 1 already holds the lead photo) and, for each target item
  not already sitting immediately after the correct predecessor, drag it onto whatever wrong item
  currently occupies that slot — that "wrong item" is usually the *same* blocking tile for several
  consecutive drags in a row (it keeps getting pushed one slot further down as correct items are
  inserted ahead of it), so track it as a single reusable `endTarget` until it finally lands in its
  own correct spot, then switch to the next blocker. This turned a naive 41-swap job into 27 drags
  for a 42-photo listing.
- **Verify with a full page reload, not just the live in-page DOM.** The reorder is saved
  server-side as you go (autosave, per the standing assumption above) — confirmed by reloading
  `.../photos?propertyId=...` from scratch afterward and re-reading `[data-testid="photo-grid"]`
  fresh; the Cover badge and the guest-facing first photo both updated to match the new position-1
  tile automatically, no separate "set cover" action needed.

General cautions carried over from [[listing-photo-sync]], all apply here too:
- **Never identify a photo to move by DOM position/index after a prior reorder or reload** — always
  re-query fresh at the moment of the action; a grid that already re-rendered once will silently
  target the wrong photo.
- **Vrbo's left nav drawer intercepts clicks near the top of the viewport** — collapse it once via
  the hamburger/"Supply navigation header toggle" before doing any drag actions.
- **Zillow's bot-check** ("Press & Hold to confirm you are human") can appear semi-randomly on
  property pages — never click through it programmatically; ask the user to clear it.
- Screenshots/temp files: the playwright MCP tools can **only** write under `~/.playwright-mcp` or
  elsewhere in the user's home directory — a scratchpad path is rejected outright, not just
  resolved elsewhere. See [[listing-photo-sync]]'s note on this for the confirmed allowed roots and
  the exact error text.

## Verifying the final order

After reordering, re-read the grid/room sections back (fresh DOM query, not a cached list from
before the drags) and confirm: room blocks are contiguous with no repeats, the lead-in slots hold
the intended photos, and — on Airbnb — the 3 attraction photos are still last and untouched. Give
one consolidated summary at the end: the room-by-room order applied, which labels were set (and on
which platforms labels weren't available at all), and anything left ambiguous that needs the user's
input.

**Large batches (roughly 5+ properties in one request):** same guidance as
[[listing-photo-sync]] — consider one session/subagent per property rather than one long session,
since a single session's cost grows with how many prior properties' transcripts are still in
context. For 2-4 properties, one session is usually still cheaper than the rediscovery overhead of
splitting.
