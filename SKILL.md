---
name: nyc-house-scanner
description: |
  Scan NYC event sources for upcoming house, tech house, and UK-influenced electronic music events.
  Filter by taste profile (artists, venues, promoters) and text a summary via iMessage.
  Triggered by cron at 9am and 6pm ET. Also use when the user asks about upcoming house music events,
  NYC nightlife, or anything related to the event sources and artists listed in this skill.
---

# NYC House Music Event Scanner

Scan event listing sites for house/tech house events in NYC. Filter against your taste profile. Text results via iMessage.

## Setup

1. Edit `references/taste-profile.md` with your artists, venues, and promoters
2. Replace `YOUR_PHONE_NUMBER` in the messaging section with your iMessage number (E.164 format)
3. Create the state file: `~/clawd/memory/nyc-house-scanner-state.json` with `{"alerted_events": [], "last_scan": null}`
4. Add cron jobs (see bottom of this file)

## Important: Browser Required

Most event sites (Shotgun, RA, DICE) block curl/WebFetch with bot protection (Vercel checkpoint, Cloudflare CAPTCHA) or render events client-side via JavaScript. **You must use browser automation to scrape them.**

If your OpenClaw instance has browser control (CDP), use it:

```bash
# 1. Start browser via control endpoint
curl -s -X POST -H "Authorization: Bearer $GATEWAY_TOKEN" http://127.0.0.1:18791/start

# 2. Connect to CDP at http://127.0.0.1:18800
#    Navigate to each source, wait 6-8s for JS to render,
#    extract text via Runtime.evaluate({ expression: "document.body.innerText" })

# 3. Stop browser when done
curl -s -X POST -H "Authorization: Bearer $GATEWAY_TOKEN" http://127.0.0.1:18791/stop
```

### Cron session requirements

**Isolated cron sessions (`--session isolated`) cannot run this skill** — they have no web access and a read-only filesystem. Use `--session main` for cron jobs. Main session jobs fire as system events that your agent processes when awake.

## Sources (check in this order)

1. **Shotgun** — `https://shotgun.live/en/cities/new-york/house`
   - Underground promoters post here first
   - Also check: `https://shotgun.live/en/cities/new-york/techno` (crossover acts)
   - Requires browser (Vercel bot protection)
2. **Resident Advisor** — `https://ra.co/events/us/newyorkcity`
   - Don't add genre params to URL — filter results after fetching
   - Requires browser (Cloudflare CAPTCHA)
3. **DICE** — `https://dice.fm/browse/new-york`
   - Brooklyn Mirage / Avant Gardner tickets are DICE-exclusive
   - Requires browser (JS SPA, `/music/electronic` subpath may 404 — use base browse URL)
4. **Edmtrain** — `https://edmtrain.com/new-york-city-ny`
   - "Recently Added" section at bottom has newest listings
   - Requires browser (events loaded via JS)
5. **Bandsintown** — `https://www.bandsintown.com/c/new-york-ny?genre=electronic`
   - Best for tracking specific artists' tour dates hitting NYC
6. **Songkick** — `https://www.songkick.com/metro-areas/7644-us-new-york-nyc/genre/electronic`

For each source, navigate via browser, wait for page load, extract `document.body.innerText`, then parse for event details.

If a source fails or returns nothing useful, skip it and move on. Don't retry.

## Filtering

After fetching events, filter against the taste profile in `references/taste-profile.md`. Read it every run.

### Priority tiers

**Tier 1 — Text immediately:**
- Any event by a top-priority promoter (regardless of lineup)
- Any named artist from the "must-see" list at a listed venue
- Free RSVP events with limited capacity (sell out in hours)

**Tier 2 — Include in daily summary:**
- Events at listed venues with house/tech house genres
- Events by listed promoters/collectives
- Any UK house/tech house DJ doing a rare NYC appearance

**Tier 3 — Skip:**
- Genres: dubstep, drum & bass, hardstyle, trance, big room EDM, psytrance
- Venues outside Manhattan, Brooklyn, Queens
- Events already alerted (check state file)

## State Tracking

Read and update `~/clawd/memory/nyc-house-scanner-state.json` every run.

Schema:
```json
{
  "alerted_events": [
    {
      "id": "ben-sterling-elsewhere-2026-03-15",
      "artist": "Ben Sterling",
      "venue": "Elsewhere",
      "date": "2026-03-15",
      "alerted_at": "2026-03-07T14:00:00Z",
      "tier": 1
    }
  ],
  "last_scan": "2026-03-07T14:00:00Z"
}
```

- Dedup key: lowercase `"{artist}-{venue}-{date}"` — skip if already in `alerted_events`
- If a previously announced event adds new artists to the lineup, send a follow-up
- Prune entries older than 7 days past the event date

## Sending Messages

Use `imsg send --to "YOUR_PHONE_NUMBER"` to send alerts.

**Keep messages short.** iMessage/AppleScript times out on long texts. Split summaries into multiple messages if >10 events. Max ~15 lines per message.

### Tier 1 format (send immediately):
```bash
imsg send --to "YOUR_PHONE_NUMBER" --text "🔥 [ARTIST] at [VENUE]
[DATE] | [TIME]
Tickets: [LINK]
[One line on why this is a match]"
```

### Tier 2 format (daily summary, 9am run only):
```bash
imsg send --to "YOUR_PHONE_NUMBER" --text "📍 NYC House Radar — [DATE]

THIS WEEK:
1. [Artist] — [Venue] — [Date]
2. [Artist] — [Venue] — [Date]"
```

Send a second message for "COMING UP" events if the list is long.

### Nothing new — Don't text. Silent run.

## Pre-Show Prep

When a Tier 1 event is found that the user is likely attending (named artist + good venue), send a cig reminder:

```bash
imsg send --to "YOUR_PHONE_NUMBER" --text "🚬 Don't forget to order cigs on DoorDash before [ARTIST] tonight"
```

Send ~4 hours before event start. If already within 4 hours, send immediately after the Tier 1 alert.

## Edge Cases

- Brooklyn Mirage 2026 summer season lineup → send full breakdown
- Named artist announces US tour including NYC → alert
- Ticket price >$80 for a club night → flag as "pricey" in the message
- Free RSVP + limited capacity = highest urgency, Tier 1

## Run Procedure

1. Read state file (create with empty schema if missing)
2. Read `references/taste-profile.md` for current filtering criteria
3. Start browser via control endpoint (auth required)
4. Connect to CDP, navigate to each source, wait 6-8s, extract page text
5. Stop browser when done scraping
6. Parse and filter events against taste profile
7. Dedup against state file
8. Send Tier 1 alerts immediately via iMessage
9. If 9am run: compile Tier 2 into daily summary, send if non-empty (split long messages)
10. If 6pm run: only send Tier 1 alerts (no summary)
11. Update state file with newly alerted events
12. Prune entries older than 7 days past event date

## Cron Setup

**Must use `--session main`** — isolated sessions lack web tools. See "Important: Browser Required" above.

```bash
# Morning scan — Tier 1 alerts + daily summary
openclaw cron add \
  --name "nyc-house-scanner:morning" \
  --cron "0 9 * * *" \
  --tz "America/New_York" \
  --session main \
  --system-event "Run the nyc-house-scanner skill. This is the 9am morning scan — send both Tier 1 immediate alerts AND the Tier 2 daily summary. Use browser control (CDP) to scrape sources." \
  --timeout-seconds 600

# Evening scan — Tier 1 alerts only
openclaw cron add \
  --name "nyc-house-scanner:evening" \
  --cron "0 18 * * *" \
  --tz "America/New_York" \
  --session main \
  --system-event "Run the nyc-house-scanner skill. This is the 6pm evening scan — send Tier 1 immediate alerts ONLY (no daily summary). Use browser control (CDP) to scrape sources." \
  --timeout-seconds 600
```
