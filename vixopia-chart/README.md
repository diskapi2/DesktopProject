# Vixopia Chart

A single-file, browser-based dashboard that charts VIX and related market
monitor data over the trading day.

The whole app is `index.html` — no build step, no dependencies to install.
Open it in a browser and it loads.

## What it shows

Two time-synchronised line charts stacked vertically:

| Chart  | Series | Notes |
| ------ | ------ | ----- |
| Top    | `ValL`, `ValS` | Y axis is **inverted** (reversed scale) |
| Bottom | `Vix` and the components that follow it in the source data | Series are discovered dynamically, up to the `Rec` field |

Both charts share a hover cursor: hovering either one highlights the matching
timestamp in the other. Dashed vertical markers are drawn at the
**9:30 AM** open and the **4:00 PM** close, interpolated between the two
nearest samples.

## Controls

- **REFRESH** — re-fetch the source data.
- **RAW DATA** — modal showing the raw synced text, newest line first.
- **FULL/PARTIAL** — toggles the time filter. In *Partial View* (the default),
  once it is 11:00 or later in `America/New_York`, samples before `09:00:00`
  are dropped. *Full View* shows every sample.
- **Debug Info** — expands to show the first 300 characters of the last
  payload received.
- **HELP** — in-app notes on the data source and proxy.

## Where the data comes from

The dashboard reads a plain-text file (`monitor.txt`) stored in Google Drive.
Browsers cannot fetch it directly because of CORS, so requests go through a
Cloudflare Worker that acts as a read-only proxy:

    index.html  ->  Cloudflare Worker  ->  Google Drive (monitor.txt)

Two constants at the top of the `<script>` block configure this:

```js
const FILE_ID    = '...';  // Google Drive file id of monitor.txt
const WORKER_URL = '...';  // Cloudflare Worker proxy endpoint
```

The worker is managed from the Cloudflare dashboard at
<https://workers.cloudflare.com/> — **Compute** -> **Workers & Pages**.

Data is fetched once automatically on page load (`window.onload`), and
thereafter only when **REFRESH** is pressed. There is no polling timer.

## Expected data format

One record per line. Each line needs a `HH:MM:SS` timestamp and any number of
`Name: value` pairs; the parser picks them up with a regular expression, so
field order is flexible and unknown fields are tolerated. Lines shorter than
20 characters are skipped, and a record is kept only if it carries a `ValL` or
a `Vix` value. If a line has no timestamp, the record is labelled by its index
(`P0`, `P1`, ...) instead.

## Built with

- [Tailwind CSS](https://tailwindcss.com/) (CDN) for styling
- [Chart.js](https://www.chartjs.org/) (CDN) for the charts, plus a small
  custom `verticalLine` plugin for the session markers

Both load from a CDN, so an internet connection is required even when viewing
cached data.

## Notes

- The in-app help text refers to the worker as `flat-half-df36`, while
  `WORKER_URL` points at `flat-hall-df36`. The URL in the code is the one
  actually used.
- `FILE_ID` and `WORKER_URL` are committed in plain text. Anyone who can read
  this repository can call that worker endpoint and read the monitor data it
  proxies. Move them out of the source, or put access control on the worker,
  if that data should not be public.
