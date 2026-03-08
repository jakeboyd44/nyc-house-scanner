# nyc-house-scanner

An [OpenClaw](https://openclaw.com) skill that scans NYC event sources for house, tech house, and UK-influenced electronic music events — then texts you what's worth going to.

Runs on a cron schedule (9am + 6pm ET), scrapes 7 event sources, filters against your personal taste profile, and sends iMessage alerts for matches. Also reminds you to order cigs on DoorDash before the show.

## What it does

- Scrapes **Shotgun**, **Resident Advisor**, **DICE**, **Edmtrain**, **doNYC**, **Bandsintown**, and **Songkick** for upcoming events
- Filters by your artists, venues, promoters, and genre preferences
- Sends **immediate alerts** for high-priority matches (your favorite artist at a great venue)
- Sends a **daily summary** at 9am with everything else worth knowing
- **Deduplicates** so you never get the same event twice
- Sends a pre-show cig reminder via DoorDash because you're going to forget

## Setup

1. Install the skill in your OpenClaw instance
2. Edit `references/taste-profile.md` with your artists, venues, and promoters (or use mine as a starting point — I have good taste)
3. Replace `YOUR_PHONE_NUMBER` in `SKILL.md` with your iMessage number
4. Create the state file:
   ```bash
   echo '{"alerted_events": [], "last_scan": null}' > ~/clawd/memory/nyc-house-scanner-state.json
   ```
5. Add the cron jobs (see bottom of `SKILL.md` for the commands)

## Taste Profile

The included example profile is tuned for UK house and tech house — Raw Cuts NYC vibes, warehouse popups, intimate events. Edit `references/taste-profile.md` to match your own preferences.

## Sources

| Source | Why |
|--------|-----|
| Shotgun | Underground promoters post here first (Raw Cuts, MAD Radio) |
| Resident Advisor | Industry standard, good genre filtering |
| DICE | Brooklyn Mirage/Avant Gardner exclusive, popup events |
| Edmtrain | Tour dates hitting NYC |
| doNYC | NYC-specific aggregator, catches what others miss |
| Bandsintown | Best for tracking specific artists |
| Songkick | Broad electronic coverage |

## License

MIT
