---
name: turbotenant-listing
description: End-to-end TurboTenant rental listing creation from an intake form (PDF or form URL) through publishing — leasing basics, permissions, utilities & amenities, listing description, photos & videos, publish, showing scheduling, and Marketing Turned On/Finish. Use when the user gives an intake form/questionnaire for a property and wants it listed on TurboTenant, or asks to create/finish/edit a TurboTenant rental listing.
---

# turbotenant-listing

Creates a TurboTenant listing end-to-end: parse the intake form, run "Create New Property" through Publish and the post-publish screens.

Built from two screen recordings of a host doing this by hand (Aug 2026). They had no captions, so the narration was transcribed locally (mlx-whisper) and cross-checked against frames sampled every 2–3 seconds and tiled into contact sheets. Labels are grounded in real screens. If a screen doesn't match, **trust the screen** and flag it.

**Positioning:** for `profile.business.positioning: furnished-monthly`, this is a flexible, fully furnished, month-to-month rental aimed at tenants (`profile.business.target_renters`), not vacation guests.

**Confidence gaps:**
- **The "Create New Property" landing step (address, property type, probably bed/bath/sqft) was never shown** — the recording starts on Leasing Basics for an existing draft. Proceed carefully and ask if it doesn't match expectations.
- **The end of Photos & Videos** (finishing reorder + its Continue) wasn't captured exactly; it's a normal drag-to-reorder grid with a FEATURED slot.

## Invocation

`/turbotenant-listing <path-to-intake-pdf>` — or "list this on TurboTenant: ~/Downloads/123-main-st-intake.pdf". Accept a photo folder too, or a form-tool URL.

## Before anything else: load the host profile

Read `host-profile.yaml` from the current working directory. If missing, offer to create it from `config/host-profile.example.yaml` and ask for the values. `profile.x.y` = that key. Blank or missing = ask per property.

## Tools

- `Read` — parse the intake form.
- `mcp__playwright__browser_*` — drive the real site in the dedicated Chrome profile (`profile.browser.playwright_config`). **Login wall → stop and tell the user.**
- `WebSearch` / `WebFetch` — light research for the location paragraph only.
- `AskUserQuestion` — real gaps and undocumented screens (especially the landing step).
- Screenshots: save under Playwright MCP's allowed root (`~/.playwright-mcp/` by default).

## Keeping cost down

- **`browser_snapshot` over screenshots** for state checks. Screenshots for photo curation, irreversible confirms, or a visual summary.
- **Read back flaky-looking fields with `browser_evaluate`**, not a screenshot.
- **Parse the form once.**
- **Several properties: one subagent/session each.**

## Run shape

- **Login:** assume logged in.
- **One checkpoint: stop before clicking "Publish listing"** on the "Publish Listing" screen. That's the actual publish action. Summarize, wait for approval, then click through the post-publish screens.
- **Title & description:** agent-drafted directly (no copy-paste through another chatbot).
- **Property type:** `profile.turbotenant.property_type` — normally **Condo for any single unit**, never "Apartment" (TurboTenant buckets Apartment as a 5+ unit building).
- **House-wide values from the profile, applied without asking:** utilities paid by landlord (`profile.defaults.utilities_included` → check every utility), pet policy (`profile.defaults.pets`), listing contact, date available (`profile.defaults.date_available: today` → today's date, ignore the form), lease term.

## Step 0 — Parse the intake form

Extract: address, beds, baths, sqft, monthly rent, deposit, occupancy limit, interior amenities, appliances, floor coverings, AC/heating presence and type, laundry, parking (spaces + type), furnished, renters insurance, military-discount note.

### When to stop and ask

Never guess `profile.always_ask` fields (address, rent, deposit, occupancy limit, parking). Also ask about **floor coverings** if the form is silent — intake forms written for short-term listings often skip it. For AC, heating and appliances, you may infer from listing photos when the form is silent (e.g. a visible wall unit → Wall AC); otherwise ask. Note every defaulted or inferred field.

## Description

Same house voice as the other skills, but for tenants: they care about commute, furnished, and utilities, not attractions.

- Lead with **flexible, fully furnished, month-to-month**.
- Name the target renters.
- Military discount only if the profile or form supports it.
- A paragraph on the space (bed/bath, kitchen, laundry, Wi-Fi, climate control, approx. sqft).
- A commute paragraph: highways, drive time downtown, airport distance — **verified by lookup, never invented.**
- Say utilities are included if the profile says so.
- No emoji. Short paragraphs.
- **Title:** follow `profile.monthly_title_pattern`, e.g. "Furnished, Flexible Monthly Rental in {city}". There's a ~100-character limit; stay well under.

## The wizard

**(Create New Property — unconfirmed) → Leasing Basics → Permissions → Utilities & amenities (2 sub-steps) → Listing Description → Photos & Videos → Publish Listing → Marketing Turned On → Showing scheduling → Finish.**

0. **Create New Property** *(unconfirmed)* — Properties → Create New Property. Expect address, property type, and likely bed/bath/sqft. Property type per the profile. If a "Looks like I have already created this" resume prompt appears, resume the draft.
1. **Leasing Basics** — Date Available per profile. Lease term radio → `profile.defaults.lease_term` (e.g. Monthly). Target Rent and Target Deposit from the form. Skip "Run Report". Continue.
2. **Permissions** — Smoking per profile. Pets: allowed/types per `profile.defaults.pets`; enter the weight limit if a field exists, otherwise note it in the summary; pet deposit = `profile.defaults.pets.deposit`. Occupancy limit → Yes, stepper to the form's number. Continue.
3. **Utilities paid by landlord** (sub-step a) — Cable, Gas, Sewage, Electricity, Internet, Water, Garbage, Recycling, Satellite TV. If `utilities_included`, **check every box.** Continue.
4. **Amenities page** (sub-step b, one long scroll, one Continue at the bottom):
   - Amenities — only what the form/photos confirm; **always check Furnished** for furnished-monthly. Don't check building amenities (pool, gym, secure building) unless real.
   - Appliances — from the form.
   - Floor coverings — from the form, else ask.
   - Air conditioning / Heating — Yes/No + type; form, else infer from photos, else ask.
   - Laundry — keep consistent with Appliances (Washer + Dryer → "included in unit").
   - Parking — space count + type from the form, else ask.
5. **Listing Description** — title + description. Continue.
6. **Photos & Videos** — upload. Min 1, 5+ recommended, min 900×600. **Hard cap of 20 photos.** Ignore the paid "Professional Photos" upsell. Drag the best photo to the **FEATURED** first slot, then room order (see vrbo-listing's ordering rules). Video tour URL only if provided. Continue.
7. **Publish Listing** — screen with the address, syndication logos, and **Save & Exit** / **Publish listing**. **Stop here.** Summarize and wait for approval.
8. On approval → **Publish listing**. A "Marketing Turned On!" toast appears → Continue.
9. **Free Easy Showing Scheduling** — Skip For Now / Add Availability. Use `profile.turbotenant.showing_scheduling` (default skip: furnished-monthly renters' schedules are too unpredictable to pre-commit slots).
10. **Marketing Turned On** info screen → **FINISH**. Lands on the property's Marketing tab. Don't act on the vacancy checklist (showings, leases) unless asked.

## Final summary (always)

- Published, or sitting ready at Publish Listing.
- Fields from the form vs. profile defaults vs. inferred from photos, and anything the owner should add to their intake form (parking, floor coverings).
- TurboTenant syndicates to third-party sites within ~48 hours.
- Showing scheduling skipped (if it was); can be added later from the Marketing tab.
- Applicant screening happens later, per applicant. Not part of listing creation.

## Known TurboTenant quirks

- **Single unit = Condo**, not Apartment.
- **"Utilities & amenities" is two sub-steps** under one header; the first has its own Continue.
- Copying the amenities panel as text loses which boxes are checked.
- **20-photo hard cap**, enforced.
- **The publish action is the "Publish listing" button** right after Photos & Videos. Showing scheduling and Finish come after and aren't gates. If a future run shows a different order, treat whichever screen actually publishes as the checkpoint.
- No screening-criteria step (unlike Zillow).
- In-progress drafts for the same address get resumed.
