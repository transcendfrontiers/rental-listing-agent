---
name: listing-update-summary
description: Draft a concise, plain-text message to the property owner/client summarizing the listing fixes and changes made in this session (Airbnb, Vrbo, Zillow, TurboTenant, channel-manager settings), save it to a file, and open it for copy-paste. Use when the user asks to draft/write a client update about what was fixed or changed.
---

# listing-update-summary

Writes a short client-facing update about what changed in this conversation, as plain text that pastes cleanly into email, SMS, Slack, or a platform inbox.

## Content

- Only include changes that were actually made **and verified** in this session (e.g. confirmed after a page reload). Anything attempted but unconfirmed, or left for the client to decide, goes under a short "Still needs your input:" line — never present it as done.
- For each change: what was wrong (one clause, in the client's terms) → what was changed → the practical effect ("guests will now get X"). One to three sentences.
- Use the property names the client uses, not internal IDs, URLs, or automation details.
- If guest-facing wording changed, quote the new wording, replacing shortcodes like `%smartlock_code%` with a bracketed description like `[guest's code]`.
- No preamble, no filler, no sign-off name (the user adds their own). Open with "Hi," unless the user names the recipient. End with "Let me know if you have any questions."

## Formatting (copy-paste safe)

- Plain text: no Markdown, tables, or emoji.
- Numbered list `1.`, `2.`, … with a blank line between items; no nested bullets.
- Straight quotes, no em dashes. ASCII only.
- No hard-wrapping: each paragraph/list item is one line.

## Steps

1. Write the draft to `<client_updates_dir>/YYYY-MM-DD-<short-topic>.txt`, where `client_updates_dir` comes from `host-profile.yaml` (default `./client-updates`). If the file exists, add `-2`, `-3` rather than overwriting.
2. Check it's ASCII-only: `LC_ALL=C grep -n '[^ -~]' <file>` should print nothing; fix any hits.
3. Open it: the user's `$VISUAL`/`$EDITOR` if set, else `open -t <file>` on macOS. Otherwise just give the path.
4. Reply with the path, a one-line summary of what it covers, and anything deliberately left out (e.g. unverified changes). Don't paste the full draft unless asked.
