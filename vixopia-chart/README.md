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

The constants at the top of the `<script>` block configure this:

```js
const FILE_ID       = '...';  // Google Drive file id of monitor.txt
const WORKER_URL    = '...';  // Cloudflare Worker proxy endpoint
const DRIVE_API_KEY = '';     // optional, for the direct-fetch backup
```

The worker is managed from the Cloudflare dashboard at
<https://workers.cloudflare.com/> — **Compute** -> **Workers & Pages**.

Data is fetched once automatically on page load (`window.onload`), and
thereafter only when **REFRESH** is pressed. There is no polling timer.

### The direct-fetch backup

If the worker fails, the app retries against the Google Drive API directly:

    https://www.googleapis.com/drive/v3/files/FILE_ID?alt=media&key=API_KEY

Unlike the usual `drive.google.com/uc?export=download` link, this endpoint
sends `Access-Control-Allow-Origin`, so a browser is allowed to read it
cross-origin. It needs two things:

1. `monitor.txt` shared as **Anyone with the link** — an API key alone can
   only reach public files.
2. A Google Cloud project with the Drive API enabled, and an API key.

If no key is configured the backup is skipped and says so; the worker path is
unaffected. Prefer passing the key by URL flag over committing it, and if you
do set `DRIVE_API_KEY` in the source, restrict the key to the Drive API and to
an HTTP referrer — this repository is public.

### Choosing a source

Append these flags to the page URL:

| Flag | Effect |
| ---- | ------ |
| *(none)* | Worker first, direct Drive fetch as backup |
| `?source=drive` | Force the direct Drive fetch, skipping the worker |
| `?direct=1` | Alias for `?source=drive` |
| `?source=worker` | Force the worker only, with no backup |
| `?apiKey=KEY` | Supply or override the Drive API key for this session |

Flags combine, so a direct-fetch test looks like:

    .../vixopia-chart/?direct=1&apiKey=YOUR_KEY

The status line under the buttons names whichever source answered, marking it
`(backup)` when the worker was tried first and failed. **Debug Info** lists
every source that failed and why, surfacing the Drive API's own error message
(for example `The request is missing a valid API key.`) rather than a bare
status code.

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

- `FILE_ID` and `WORKER_URL` are committed in plain text. Anyone who can read
  this repository can call that worker endpoint and read the monitor data it
  proxies. Move them out of the source, or put access control on the worker,
  if that data should not be public.
