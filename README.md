# GPS Link Feed

Live link-share feed from the GPS Action Network! WhatsApp group, surfaced for
triage and review.

## What this shows

Every URL shared in the GPS Action Network! WhatsApp community group, in
reverse-chronological order. Backed by a WhatsApp → Supabase pipeline; this
page is read-only.

- Sender name where available (WhatsApp masks group members behind anonymous IDs by default)
- Page title (from WhatsApp's own link previews where available)
- Click-through to the original link
- Approximate share time

Polling refreshes every 5 seconds; new shares appear without reload.

## Hosted

GitHub Pages from this repo. The single `index.html` + `config.js` are the
whole app — no build step.

## Ownership

- Pipeline & data → Grant De Swardt, AI Fusion Automations
  (`grant@aifusionautomations.com`)
- Source group → GPS Network community
