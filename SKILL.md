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

1. Copy `references/taste-profile.md` and customize with your artists, venues, and promoters
2. Replace `YOUR_PHONE_NUMBER` in the messaging section with your iMessage number (E.164 format)
3. Create the state file: `~/clawd/memory/nyc-house-scanner-state.json` with `{"alerted_events": [], "last_scan": null}`
4. Add cron jobs (see bottom of this file)

## Sources (check in this order)

1. **Shotgun** — `https://shotgun.live/en/cities/new-york/house`
   - Raw Cuts and underground promoters post here first
   - Also check: `https://shotgun.live/en/cities/new-york/techno` (crossover acts)
2. **Resident Advisor** — `https://ra.co/events/us/newyorkcity?genres=house&genres=techhouse`
3. **DICE** — `https://dice.fm/browse/new-york/music/electronic`
   - Brooklyn Mirage / Avant Gardner tickets are DICE-exclusive
   - Good for popup events
4. **Edmtrain** — `https://edmtrain.com/new-york-city-ny`
5. **doNYC** — `https://www.donyc.com/genres/house`
   - NYC-specific aggregator, good catch-all
6. **Bandsintown** — `https://www.bandsintown.com/c/new-york-ny?genre=electronic`
   - Best for tracking specific artists' tour dates hitting NYC
7. **Songkick** — `https://www.songkick.com/metro-areas/7644-us-new-york-nyc/genre/electronic`

For each source, use WebFetch with a prompt like:
```
List all upcoming events. For each event return: artist/lineup, venue, date, time, ticket link, price if shown, and promoter/collective if listed. Format as JSON array.
```

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

[N] new events found:

1. [Artist] — [Venue] — [Date]
2. [Artist] — [Venue] — [Date]

[Note about top promoters if nothing new from them]"
```

### Nothing new — Don't text. Silent run.

## Pre-Show Prep

When a Tier 1 event is found that the user is likely attending (named artist + good venue), send a cig reminder:

```bash
imsg send --to "YOUR_PHONE_NUMBER" --text "🚬 Don't forget to order cigs on DoorDash before [ARTIST] tonight"
```

Send this as a separate text ~4 hours before the event start time. If the event is tonight and it's already within 4 hours, send immediately after the Tier 1 alert.

## Edge Cases

- Brooklyn Mirage 2026 summer season lineup → send full breakdown
- Named artist announces US tour including NYC → alert
- Ticket price >$80 for a club night → flag as "pricey" in the message
- Free RSVP + limited capacity = highest urgency, Tier 1

## Run Procedure

1. Read `~/clawd/memory/nyc-house-scanner-state.json` (create with empty schema if missing)
2. Read `references/taste-profile.md` for current filtering criteria
3. Fetch each source via WebFetch, extract events
4. Filter events against taste profile
5. Dedup against state file
6. Send Tier 1 alerts immediately via iMessage
7. If 9am run: compile Tier 2 into daily summary, send if non-empty
8. If 6pm run: only send Tier 1 alerts (no summary)
9. Update state file with newly alerted events
10. Prune entries older than 7 days past event date

## Cron Setup

Add two cron jobs via OpenClaw CLI:

```bash
# Morning scan — Tier 1 alerts + daily summary
openclaw cron add \
  --name "nyc-house-scanner:morning" \
  --cron "0 9 * * *" \
  --tz "America/New_York" \
  --session isolated \
  --model codex \
  --message "Run the nyc-house-scanner skill. This is the 9am morning scan — send both Tier 1 immediate alerts AND the Tier 2 daily summary." \
  --announce \
  --channel imessage \
  --to "YOUR_PHONE_NUMBER" \
  --timeout-seconds 300

# Evening scan — Tier 1 alerts only
openclaw cron add \
  --name "nyc-house-scanner:evening" \
  --cron "0 18 * * *" \
  --tz "America/New_York" \
  --session isolated \
  --model codex \
  --message "Run the nyc-house-scanner skill. This is the 6pm evening scan — send Tier 1 immediate alerts ONLY (no daily summary)." \
  --announce \
  --channel imessage \
  --to "YOUR_PHONE_NUMBER" \
  --timeout-seconds 300
```
