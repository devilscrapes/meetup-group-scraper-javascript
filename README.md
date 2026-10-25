<p align="center"><a href="https://apify.com/DevilScrapes/meetup-group-scraper"><img src="assets/banner.svg" alt="Meetup Group Scraper — JavaScript example" width="100%"></a></p>

# Meetup Group Scraper for JavaScript and Node.js: follow community calendars from your backend

<p align="center">
<a href="https://apify.com/DevilScrapes/meetup-group-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="JavaScript" src="https://img.shields.io/badge/Node.js-20%2B-339933?logo=nodedotjs&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/meetup-group-scraper-javascript/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/meetup-group-scraper-javascript/actions/workflows/ci.yml/badge.svg"></a>
</p>

This repository shows how to call the **[Meetup Group Scraper](https://apify.com/DevilScrapes/meetup-group-scraper)** from Node.js with the official `apify-client` package. You keep a list of Meetup groups, by link or by slug, and your service gets back plain objects describing each event: title, start and end, timezone, venue and coordinates, RSVP total, capacity, price and host. The Actor runs on Apify's infrastructure, so a Lambda, a Cloudflare Worker cron or a small Express app only starts a run and reads the rows. Run as shipped, it saves two files next to the script; lift the five-line core into your own handler and the events stay in memory.

> **Actor:** [Meetup Group Scraper](https://apify.com/DevilScrapes/meetup-group-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in Python:** [meetup-group-scraper-python](https://github.com/devilscrapes/meetup-group-scraper-python)

## 🔥 What you get

- **Groups you name, not search luck**: each entry in `groupUrls` can be a full `meetup.com/<group>/` link, a link with `/events/` on the end, or just the bare group slug; the Actor reduces all of them to the slug before fetching.
- **Pick the side of today you care about**: `eventWindow` keeps events still to come, events already held, or both, judged against the moment the run started.
- **Gaps filled from the event page**: when a listing lacks an end time, venue coordinates or a host, the Actor opens that event's own page to complete the row, and skips the extra request when nothing is missing.
- **Counts, not people**: `rsvp_count` is a single total per event; attendee names and profiles never enter the dataset.
- **Optional discovery mode**: leave the group list empty and give `location` and/or `topics` to search Meetup's find page instead; `matched_via` tells you which route produced each row.
- **One row per event across the run**: an event that turns up twice, say from a group link and from a search, is written once, and `maxEvents` caps the run total.
- **Tolerant of hiccups**: throttled or erroring pages are retried up to five times with growing pauses on a fresh proxy session, and a group slug Meetup does not recognise is logged and skipped.

## 🎯 Use cases

### Publish a combined community calendar

Agencies, coworking spaces and university clubs often want one calendar that merges several local groups. Turn the rows into an ICS file, like the recipe below does, serve it from a static bucket and anyone can subscribe from Google Calendar or Outlook.

### Ping Slack when a watched group schedules something

Store the `event_id` values you have already announced in KV or Redis. On each scheduled run, post only the unseen ones to a Slack incoming webhook with the start time and the `event_url`, and your team hears about new sessions the day they are posted.

### Power an events widget on a partner page

A serverless function can refresh a JSON file of upcoming events for the communities you sponsor every few hours. Your front end reads that file, so page loads never wait on an Apify run.

### Enrich CRM accounts with community activity

If your accounts map to local user groups, attach the next event date and its RSVP total to each record. Sales and DevRel see at a glance where a talk or a pizza budget would land in front of a full room.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/meetup-group-scraper-javascript.git
cd meetup-group-scraper-javascript
npm install
export APIFY_TOKEN=your_token_here
node index.js                       # uses input.json
node index.js my-input.json --out data --max-charge 2
```

The script starts the [Meetup Group Scraper](https://apify.com/DevilScrapes/meetup-group-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Everything else in `index.js` is plumbing; the real work is these few statements. `call()` resolves once the Actor run has stopped, and `listItems()` reads what it saved.

```javascript
import { ApifyClient } from 'apify-client';

const client = new ApifyClient({ token: 'YOUR_APIFY_TOKEN' });
const run = await client.actor('DevilScrapes/meetup-group-scraper').call({"groupUrls": ["chapter-community"], "eventWindow": "upcoming", "maxEvents": 10, "includeDescription": true, "proxyConfiguration": {"useApifyProxy": true}});
const { items } = await client.dataset(run.defaultDatasetId).listItems();
console.log(items);
```

## 📥 Input

`index.js` sends `input.json` as-is. The example follows one public organisation's group, Chapter Community, asks only for future events and caps the run at 10 rows. Add more slugs to `groupUrls` to watch several communities in one call.

```json
{
  "groupUrls": [
    "chapter-community"
  ],
  "eventWindow": "upcoming",
  "maxEvents": 10,
  "includeDescription": true,
  "proxyConfiguration": {
    "useApifyProxy": true
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `groupUrls` | array | no | `["chapter-community"]` | — |
| `location` | string | no | — | — |
| `topics` | array | no | — | — |
| `eventWindow` | string | no | `"upcoming"` | `upcoming`, `past`, `both` |
| `maxEvents` | integer | no | `50` | — |
| `includeDescription` | boolean | no | `true` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true}` | — |

## 📤 Output

Expect one object per event. Timestamps are ISO-8601 strings with an offset, so `new Date()` parses them directly; `rsvp_count` is always an integer, while capacity, fee and venue properties may be `null`.

| Field | Type | Always present |
|---|---|---|
| `event_id` | string | yes |
| `event_url` | string | yes |
| `event_name` | string | yes |
| `group_name` | string | yes |
| `group_url` | string | yes |
| `organizer_name` | string or null | no |
| `date_time_start` | string | yes |
| `date_time_end` | string or null | no |
| `timezone` | string or null | no |
| `is_online` | boolean | yes |
| `venue_name` | string or null | no |
| `venue_address` | string or null | no |
| `venue_city` | string or null | no |
| `venue_state` | string or null | no |
| `venue_country` | string or null | no |
| `venue_lat` | number or null | no |
| `venue_lon` | number or null | no |
| `rsvp_count` | integer | yes |
| `max_tickets` | integer or null | no |
| `is_free` | boolean | yes |
| `fee_amount` | number or null | no |
| `fee_currency` | string or null | no |
| `description` | string or null | no |
| `matched_via` | string | yes |
| `scraped_at` | string | yes |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result row emitted | $0.002 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/meetup-group-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in JavaScript

The snippet below converts the saved rows into an iCalendar file using nothing beyond Node.js built-ins. Events without an end time get a two-hour placeholder, and the RSVP count goes into each entry's description.

```javascript
import { readFile, writeFile } from 'node:fs/promises';

const rows = JSON.parse(await readFile('results.json', 'utf8'));
const stamp = (iso) => new Date(iso).toISOString().replace(/[-:]/g, '').replace(/\.\d{3}/, '');
const escape = (text) => String(text ?? '').replace(/[\;,]/g, (c) => `\\${c}`).replace(/\n/g, '\\n');

const lines = ['BEGIN:VCALENDAR', 'VERSION:2.0', 'PRODID:-//meetup-group-feed//EN'];
for (const row of rows) {
    const end = row.date_time_end ?? new Date(Date.parse(row.date_time_start) + 2 * 3600e3).toISOString();
    const where = row.is_online ? 'Online' : [row.venue_name, row.venue_address, row.venue_city].filter(Boolean).join(', ');
    lines.push(
        'BEGIN:VEVENT',
        `UID:${row.event_id}@meetup.com`,
        `DTSTAMP:${stamp(row.scraped_at)}`,
        `DTSTART:${stamp(row.date_time_start)}`,
        `DTEND:${stamp(end)}`,
        `SUMMARY:${escape(`${row.event_name} (${row.group_name})`)}`,
        `LOCATION:${escape(where)}`,
        `URL:${row.event_url}`,
        `DESCRIPTION:${escape(`${row.rsvp_count} going${row.is_free ? '' : `, ${row.fee_amount} ${row.fee_currency}`}`)}`,
        'END:VEVENT',
    );
}
lines.push('END:VCALENDAR');
await writeFile('community.ics', lines.join('\r\n') + '\r\n');
console.log(`Wrote ${rows.length} events to community.ics`);
```

## ❓ FAQ

### Can I start from a city instead of group links?

Yes. Leave `groupUrls` empty and set `location`, `topics` or both; the Actor runs one Meetup find search per topic. Rows from that path carry `matched_via: "location_topic_search"` so you can treat them differently downstream.

### How do I run it on a schedule without a server?

Create an Apify schedule for the Actor, or call this module from a GitHub Actions cron or a serverless timer. Either way, dedupe on `event_id` so repeated runs update existing events instead of adding copies.

### Which time zone are the dates in?

`date_time_start` and `date_time_end` keep the offset of the event's local time, and `timezone` names the group's zone, such as `America/New_York`. Convert with `Intl.DateTimeFormat` when you display them.

### Does it work with TypeScript?

Yes. `apify-client` ships its own types; declare an interface for the event row and cast the `items` array to it.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/meetup-group-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Meetup Group Scraper on Apify Store](https://apify.com/DevilScrapes/meetup-group-scraper)
- [Python version of this example](https://github.com/devilscrapes/meetup-group-scraper-python)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [Meetup Events Scraper](https://apify.com/DevilScrapes/meetup-events-scraper) — keyword and city searches across all of Meetup when you do not yet know which groups matter.
- [Lu.ma Event Discovery Scraper](https://apify.com/DevilScrapes/luma-event-discovery) — the Luma calendars where many startup and AI communities now publish.
- [Eventbrite Events Scraper](https://apify.com/DevilScrapes/eventbrite-events-scraper) — ticketed workshops and conferences with organizer profiles.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/meetup-group-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Meetup or Meetup.com.
