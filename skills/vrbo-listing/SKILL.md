---
name: vrbo-listing
description: End-to-end Vrbo listing creation from an intake form (PDF or form URL) through publishing and post-publish edits — bedrooms/bath/address wizard, rates & fees, rules, photos, amenities, location, health & safety, rules & policies (24-hour booking type), welcome guide, and the post-publish calendar/minimum-stay fix-up. Use when the user gives an intake form/questionnaire for a new property and wants it listed on Vrbo, or asks to create/finish/edit a Vrbo listing.
---

# vrbo-listing

Creates a Vrbo listing end-to-end: parse the intake form, run the "List Your Property" wizard, publish, then fix up everything the wizard doesn't get right the first time in the Property Editor and Calendar.

Built from screen recordings of a host doing this flow by hand (new-listing wizard + post-publish editing, Aug 2026). Field labels were checked against the actual UI frames, not just the narration. Vrbo changes its UI; **if a screen doesn't match what's documented here, trust the screen and flag the mismatch to the user rather than forcing the old label.**

## Invocation

`/vrbo-listing <path-to-intake-pdf>` — or ask naturally: "list this on Vrbo: ~/Downloads/123-main-st-intake.pdf". Accept a photo folder path too.

## Before anything else: load the host profile

Read `host-profile.yaml` from the current working directory. If it doesn't exist, tell the user, offer to create it from `config/host-profile.example.yaml` in this plugin, and ask for the values before continuing. Below, `profile.x.y` means that key in the file. A blank or missing key means "ask per property".

## Tools

- `Read` — parse the intake PDF and any reference material.
- `mcp__playwright__browser_*` — drive the real Vrbo site in the dedicated Chrome profile set in `profile.browser.playwright_config`. Login persists in that profile between runs. **If you hit a login wall, stop and tell the user. Never guess or enter credentials.**
- `WebSearch` / `WebFetch` — light research only: nearby landmarks for the title/description and "Location and View" checkboxes, and directions for the Welcome Guide. Never used for financial or address data.
- `AskUserQuestion` — for real gaps the intake form leaves open (see "When to stop and ask").
- Screenshots: Playwright MCP only writes inside its allowed roots (by default `~/.playwright-mcp/`). Pass a filename there, not a temp/scratchpad path (rejected) and not a bare name loose in the home dir.

## Keeping cost down

- **Use `browser_snapshot` (accessibility tree), not `browser_take_screenshot`, to verify wizard state.** A field value, a checked box, or an enabled Continue button is a text read. Save screenshots for things that need eyes: photo curation/ordering, an irreversible confirm dialog right before clicking it, or a visual pre-publish summary if the user wants one.
- **Read back a value that might not have stuck with `browser_evaluate`, not a screenshot.** Some fields visually show a value and silently revert (see the Zillow credit-score quirk). A DOM read doesn't lie.
- **Parse the intake form once** in Step 0 and keep the fields in working memory. Don't re-open it mid-wizard.
- **Several properties in one request: one subagent/session per property.** Otherwise property N pays for the transcript of properties 1..N-1. A fresh context per property keeps cost flat.

## Run shape

- **Login:** assume the browser is already logged in. Only stop if a login wall actually appears.
- **One checkpoint:** run the whole wizard (Phase A) autonomously, then **stop once, before Publish**, and show a summary for approval. After approval, Phase B runs to completion and ends with a final summary.
- **Title & description:** always agent-drafted (see below).

## Step 0 — Parse the intake form

Extract: full address incl. unit, bedroom/bathroom count, bed type per bedroom (plus extras like a sofa bed), bedsheets provided, bathroom type/amenities, guest capacity, amenities, pet policy, nightly rate, cleaning fee, pet fee, check-in/out times, real minimum stay, cancellation policy, parking, lock/access type and instructions, Wi-Fi, and any explicit overrides of the profile defaults.

### When to stop and ask

Never guess anything in `profile.always_ask` (by default: address, bed/bath count, nightly rate, parking), or pet policy if both the form and the profile are silent. For everything else, fall back to the profile and **record which fields came from the form vs. the profile** for the summary.

## Defaults (from the profile unless the form overrides)

| Field | Source |
|---|---|
| Minimum age | `profile.defaults.minimum_age` |
| Cleaning fee | form, else `profile.defaults.cleaning_fee` |
| Pet fee | form, else `profile.defaults.pets.per_stay_fee` |
| Pets | form, else `profile.defaults.pets` (types, weight limit) |
| Events / children / smoking | `profile.defaults.*` |
| Booking window | `profile.vrbo.booking_window_months` |
| Banking country | `profile.vrbo.bank_country` |
| Property type | `profile.vrbo.property_type` (usually Residence, not hotel) |
| Host type | `profile.vrbo.host_type` |
| Rate automation | **OFF whenever `profile.business.channel_manager` is set** — pricing lives there. Never enable it without being told to. |
| "10% new listing promotion" | Keep opted in if `profile.vrbo.keep_new_listing_promo` (it's Vrbo's default state) |
| Last-minute / early-booking promotions | Leave **not** opted in unless the form says otherwise |
| Calendar sync in the wizard | Leave unsynced ("open calendar") when a channel manager is set — it gets linked there |
| Contactless check-in/out | `profile.defaults.contactless_checkin` |
| Booking type | `profile.vrbo.booking_type` (switched post-publish) |
| Cancellation policy | form, else `profile.vrbo.cancellation_policy` |

## Title & description

Write the title yourself in the house style shown in `profile.title_examples`.

- Lead with the neighborhood/area or the single best feature; mention bed count or property type when useful.
- A well-known nearby landmark is good for search, **but only if a quick web check on the address confirms it's actually close. Never invent one.**
- "Feature | Neighborhood" pipe format is fine.
- No emoji, no ALL CAPS, no exclamation points. Aim for ~45–70 characters.

Description: warm but factual, built from the form's rooms, amenities and location. No emoji — Vrbo has been seen auto-inserting a smiley on paste into the description field; strip it.

## Photo selection & ordering

The rule: **each room's photos form one unbroken block. Once the sequence moves past a room, it never comes back.** Kitchen → exterior → bathroom → more kitchen is wrong.

1. **Pick the cover first** — the single most inviting shot. Actively compare; don't default to whatever uploaded first.
2. **The cover's room leads.** Its other photos follow right behind it. If you later swap the cover to a different room, move that room's photos up behind it and fold the old cover's room back into its walkthrough position.
3. **Fill the rest of the "best 4" lead-in** with the next-best shots overall. These can come from different rooms (it's a highlights reel), but take only one shot per room here; the rest of that room stays in its own block later.
4. **Then walk the property in the order a guest meets it:** exterior → living / kitchen / dining → each bedroom → each bathroom → outdoor/amenities (pool, patio) → misc.
5. In Phase B's room-photo sections, add each room's photos together and double-check multi-bedroom properties — it's easy to misfile which bedroom a shot belongs to.

If photos don't clearly sort into rooms, **ask instead of guessing.**

## Phase A — New listing wizard

From the Vrbo dashboard → **List Your Property**. Three progress sections: "About your property", "Staying at your property", "Managing your property". Steps 1–16 were confirmed against real screens. **Steps 17–22 (Section 3) are lower-confidence** — the source recording cut off, so they come from narration only. Anywhere in the flow: documented step matches the screen → proceed; anything else → stop and `AskUserQuestion` with a description of what's on screen.

1. Bedrooms & bathrooms — from the form.
2. Address — full address incl. unit, verify the map pin.
3. Basic amenities — check only what the form confirms. Don't guess on ones it's silent about.
4. Bed details per bedroom — bed type, extra sleeping arrangements, extra bed rows, "bedsheets provided".
5. Bathroom details — type + amenities.
6. Photos — if provided, upload and order per the rules above. Otherwise skip and do it in Phase B.
7. Headline — the drafted title.
8. Description — the drafted description. Remove any auto-inserted smiley.
9. Number of guests — from the form.
10. Minimum age — profile default.
11. Rules — events, pets (fill the pet details Vrbo asks for if allowed), children, smoking.
12. Rates & fees — bank country; nightly rate from the form; **"Enable rate automation" stays OFF** (see defaults).
13. Promotions — confirm "10% new listing promotion" shows "Added to your listing"; leave the other two off.
14. Cleaning fee.
15. Pet fee — if pets are allowed, check the box and enter it.
16. Check-in/check-out times — from the form; verify, don't assume.
17. **Minimum stay** — Vrbo's wizard caps this at **27** whatever you type. If the real minimum is > 27, enter 27 as a placeholder and fix it in Phase B. Otherwise enter the real value.
18. "How many months before check-in will you accept bookings?" — profile default.
19. Calendar — "open calendar" / leave unsynced (if a channel manager is set).
20. Host type — profile default. Leave the trailing optional field blank unless the form has content for it.
21. Property type — profile default.
22. Taxes — if "How do you collect taxes?" appears, choose "Taxes are a percentage of the rental amount and collected at booking." It may not appear, or Vrbo may let you publish without it after resuming a draft. Don't force an answer nothing is asking for.
23. **Stop before Publish.** Summarize: title, description, address, bed/bath, nightly rate, cleaning fee, pet fee, minimum-stay placeholder vs. real value, booking window, and every field that used a profile default or was ambiguous. Wait for explicit approval.
24. On approval → Publish.

## Phase B — Post-publish editing

### Calendar tab
1. Verify the nightly rate.
2. Promotions — exactly **one** active. Vrbo has been seen duplicating the new-listing promo; remove the extra.
3. Additional charges — verify cleaning and pet fees saved. Vrbo has been seen silently dropping these on first save; re-enter if blank.
4. Availability → **Minimum Nights** — replace the 27 placeholder with the real value. This is what the placeholder was for; don't forget it.
5. Cancellation policy — set per defaults.

### Property → Property Details → Edit Property
1. Description — confirm it carried over.
2. Photos — re-check the cover, re-run the full ordering rules (this is where they matter most), and add bathroom/bedroom photos in the room-photo sections, matching the right bedroom. Saves automatically.
3. Amenities — check every amenity the form confirms.
4. Location — verify the exact-location pin. In "Location and View", only check descriptors you can verify from the address or a quick lookup.
5. Health and Safety → Enhanced cleaning practices — contactless check-in and check-out per `profile.defaults.contactless_checkin`. Answer differently if the form describes an in-person handoff.
6. Contact information — verify.
7. Taxes — should already be set; act only if it looks unset.
8. Rules and Policies:
   - **Property protection** — don't opt in on your own. Flag it for the owner.
   - Check-in & out times — verify.
   - House rules — usually done in Phase A.
   - Rental agreement — skip.
   - **Booking type** → switch from "Instant Booking" to the profile's booking type (e.g. "24-hour Review"). Vrbo shows a single-choice exit survey; pick "I'm worried about getting a double booking", Continue, Save.
   - Payment terms — if the tax question shows up here instead, answer as in Phase A step 22.
9. Welcome Guide:
   - Directions — can be generated from the address with a quick lookup. **Parking must come from the form; never guess it.**
   - Access — for a smart lock, check "Smart lock" and say "enter the code provided before check-in" unless the form says more.
   - Wi-Fi — fill if known; otherwise flag as pending.
   - Amenity access (pool, gym) — if applicable.
   - Checkout instructions — clear steps from the form.

## Final summary (always)

- If `profile.business.channel_manager` is set: **tell the user to sync the calendar there right away.** This skill doesn't do it, and until it's done there's a real double-booking risk across platforms.
- Every field that used a profile default vs. came from the form, and everything flagged for the owner (ambiguous pet policy, Property protection, Wi-Fi pending, etc.).
- Whether the minimum-stay placeholder fix-up was completed.

## Known Vrbo quirks

- Minimum stay caps at 27 in the wizard. Expected; fix post-publish.
- The new-listing promotion can be duplicated. Check Calendar → Promotions after publishing.
- Cleaning/pet fees sometimes don't persist on first save.
- Layout can look broken mid-wizard. Cosmetic.
- The site is slow between steps. **Wait for spinners to clear instead of re-clicking** — re-clicking can double-submit.
