---
name: zillow-listing
description: End-to-end Zillow Rental Manager listing creation from an intake form (PDF or form URL) through publishing — property info, rent & costs, media, screening criteria, description & amenities, final details (lease terms, owner contact, tours), and review & publish. Use when the user gives an intake form/questionnaire for a property and wants it listed on Zillow (Rental Manager), or asks to create/finish/edit a Zillow rental listing.
---

# zillow-listing

Creates a Zillow Rental Manager listing end-to-end: parse the intake form, run "List your rental property" through Publish.

Built from screen recordings of a host doing this by hand (Aug 2026). Labels were checked by pulling the videos' captions and matching them to real UI frames. If a screen doesn't match, **trust the screen** and flag the mismatch.

**Positioning:** when `profile.business.positioning` is `furnished-monthly`, this is a flexible, fully furnished, month-to-month listing for `profile.business.target_renters` — not a standard 12-month lease and not a nightly stay. Write and configure accordingly.

**Confidence:** every step is confirmed against real screens **except Media (photos)**, which never appeared in the recordings. Treat it as a normal upload; if it shows something unexpected, ask.

## Invocation

`/zillow-listing <path-to-intake-pdf>` — or "list this on Zillow: ~/Downloads/123-main-st-intake.pdf". Accept a photo folder too. If the intake is a form-tool response at a URL, read it with `WebFetch`/`Read`.

## Before anything else: load the host profile

Read `host-profile.yaml` from the current working directory. If it's missing, offer to create it from `config/host-profile.example.yaml` and ask for the values. `profile.x.y` below means that key. Blank or missing = ask per property.

## Tools

- `Read` — parse the intake form.
- `mcp__playwright__browser_*` — drive the real Zillow site in the dedicated Chrome profile (`profile.browser.playwright_config`). **Login wall → stop and tell the user.**
- `WebSearch` / `WebFetch` — light research only: nearby highways, drive time downtown, airport distance for the description. Never for financial or address data.
- `AskUserQuestion` — for real gaps, and for any screen this skill doesn't document.
- Screenshots: save under Playwright MCP's allowed root (`~/.playwright-mcp/` by default), not a temp path.

## Keeping cost down

- **`browser_snapshot` over screenshots** for verifying state. Screenshots only for photos, irreversible confirms, or a visual summary.
- **The "Minimum credit score" dropdown is exactly where a screenshot lies** (see quirks). Confirm its held value with `browser_evaluate` on the live `<select>`.
- **Parse the form once.**
- **Several properties: one subagent/session each**, so cost stays flat instead of growing with every earlier transcript.

## Run shape

- **Login:** assume logged in; stop only on an actual wall.
- **One checkpoint:** run the wizard autonomously, **stop once before Publish** (or "Publish without premium"), summarize, wait for approval.
- **Screening criteria, lease-terms wording and tour availability are per-property.** Pull them from the form. Don't reuse a previous property's answers as a template.
- **House-wide values come from the profile:** pet types/deposit/pet rent (`profile.defaults.pets`), renters insurance, utilities included, listing contact (`profile.listing_contact`). Apply them without asking and note them in the summary.

## Step 0 — Parse the intake form

Extract: address, property type (apartment/house/condo/townhome — not "room for rent" unless the form says so), beds, baths, sqft, monthly rent, security deposit, furnished, what's included in rent, pet policy, interior amenities (laundry, cooling, heating, appliances, flooring), lease term, renters insurance, listing contact, date available.

### When to stop and ask

Never guess `profile.always_ask` fields, property type, or whether pets are allowed (if the profile is also silent). Also ask about Zillow-specific fields general intake forms often skip: **income-to-rent ratio, minimum credit score, lease-terms wording, tour length/availability.** Note everything that used a default.

## Description

Write it yourself (no external tool). For `furnished-monthly`:

- Lead with **flexible, fully furnished, month-to-month** — the main selling point.
- Name the target renters (`profile.business.target_renters`).
- A paragraph on the space (bed/bath, kitchen, laundry, Wi-Fi, climate control) for someone staying weeks to months.
- A location/commute paragraph: nearby highways, drive time downtown, airport distance — **confirm with a web lookup for the real address; never invent distances.**
- Mention a military discount only if `profile.business.military_discount` is true or the form says so.
- No emoji. Warm, factual, short paragraphs.

For `short-term`, lean on location and amenities instead and drop the month-to-month framing.

## The wizard

Progress bar: **Property info → Rent & costs → Media → Screening criteria → Description & amenities → Final details → Review & publish.** Start at Zillow → **Manage rentals** → **List your property for rent**. If a draft for the address exists, Zillow resumes it — check for this before starting fresh.

1. **Landing form** — street address, property type. Leave the "room for rent with shared living space" toggle off unless it really is one.
2. **Property info** — type, sqft, beds, baths.
3. **Rent & costs** — monthly rent, security deposit. "Promote a special offer?" — **leave blank and flag it for the owner** (a discount is a business decision, not a data field). Additional monthly fees: leave Utilities blank if `profile.defaults.utilities_included`; Parking/Administrative only if the form lists a fee. "Show the total price" — off unless fees were added.
4. **Media** — upload photos; lead with an inviting exterior/hero shot. Not documented beyond that; ask if anything looks unexpected.
5. **Screening criteria and policies**
   - Financial criteria: income-to-rent ratio, "negotiable" checkbox, minimum credit score — from the form. Minimum monthly income auto-calculates from rent × ratio; **don't type it.**
   - Pets: per form, else profile. Pet type counters (Cats / Small dogs < 40 lb / Large dogs > 40 lb) per `profile.defaults.pets.types`. "Pet policy is negotiable" — unchecked unless told. Pet fees: deposit = `profile.defaults.pets.deposit` (check "one-time, non-refundable" if `deposit_refundable` is false), pet rent = `profile.defaults.pets.pet_rent`.
6. **Description & amenities** — laundry (radio, pick one), cooling, heating, appliances, flooring, other (e.g. Furnished). Check only what the form confirms. Paste the drafted description.
7. **Final details** — a sub-wizard. It starts at "Step 1 of 5" and grows to 7 when a tour option is chosen at step 4. Expected.
   1. **Lease terms** — a short paragraph on duration/flexibility, furnished status, and any policy not captured elsewhere. Example: *"This is a flexible month-to-month lease. The rental is fully furnished, and utilities are included in monthly rent."* Say utilities are included if the profile says so. Renters insurance → `profile.defaults.renters_insurance_required`.
   2. **Who is listing this property?** — role, name, email from `profile.listing_contact`. **Overwrite whatever the account pre-filled if it differs.**
   3. **Verify your phone number** — usually pre-verified. Confirm and continue; change only if the form gives a different number.
   4. **Use Zillow to book tours?** — per `profile.zillow.tour_booking`: "Review and confirm" (renter requests, you confirm), "Book tours instantly", or neither (skips to step 7).
   5. **Tour settings** — tour length and the property's **real** time zone.
   6. **Tour availability** — specific calendar dates, one "Add a day" at a time, ~1 month out max. From the form; **if silent, ask for a pattern. It's the owner's real calendar.**
   7. **Confirm final details** — hide address, date available, lease duration, accept online applications. **Always re-check Lease duration** — Zillow has silently set "1 year" on a month-to-month listing. Open the modal and set it to match the form and lease terms.
8. **Review & publish.** Decline the premium upsell if `profile.zillow.decline_premium`.
9. **Stop before Publish.** Summarize address, type, bed/bath/sqft, rent, deposit, screening criteria, pet policy, lease terms text, renters insurance, listing contact, tour settings, lease duration, and every defaulted or ambiguous field. Wait for approval.
10. On approval → publish. Zillow confirms and says the listing posts to Zillow, HotPads and Trulia within the hour, and offers "Add more availability" — mention it, don't click it.

## Final summary (always)

- Published, or sitting ready at Review & publish.
- Fields from the form vs. defaults, and everything flagged for the owner (special offer, fees, tour pattern).
- Zillow syndicates to HotPads and Trulia automatically.
- Tour availability runs out after ~a month and needs topping up; this skill doesn't do that.

## Known Zillow quirks

- **"Minimum credit score" can silently revert to "Not set"** after you navigate away, even though the income-ratio dropdown next to it holds. A programmatic select shows it selected, then it reverts. Fix: focus the field and type the digits with real keystrokes (e.g. "6","6","0", Enter). Then navigate forward and back once to confirm it held, and re-check on Review & publish ("Credit score: NNN or above").
- Lease duration can default to "1 year". Always verify at Confirm final details.
- Final-details step count changes (5 vs. 7). Not a bug.
- Minimum monthly income auto-calculates. Don't type it.
- An in-progress draft for the same address gets resumed.
- A "Press & Hold to confirm you are human" bot check can appear. **Never try to get past it programmatically**; ask the user to clear it.
