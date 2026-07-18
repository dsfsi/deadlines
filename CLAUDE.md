# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Jekyll-based static site (deployed to GitHub Pages) that shows countdown timers to AI/ML/DS conference deadlines relevant to the African continent. It is a fork of [aideadlin.es](http://aideadlin.es/).

## Build & Development

```bash
# Install dependencies
bundle install

# Build the site
bundle exec jekyll build --future

# Serve locally with live reload
bundle exec jekyll serve --future

# Validate HTML (as CI does)
bundle exec htmlproofer ./_site --only-4xx --check-favicon --check-html
```

The site is served from the `gh-pages` branch. CI runs on Travis CI and builds + validates HTML on every push.

**Do not attempt a local `bundle exec jekyll build`/`serve` in this environment** — the local Ruby/gem setup here is incomplete (missing native extensions, gem version mismatches) and builds fail for reasons unrelated to any code change. When editing `_data/conferences.yml`, validate with a plain YAML parse (e.g. `python3 -c "import yaml; yaml.safe_load(open('_data/conferences.yml'))"`) and check for duplicate `id`s instead. Rely on CI (Travis) to catch real build/HTML issues after push.

## Adding or Updating Conferences

All conference data lives in `_data/conferences.yml`. Each entry requires:

```yaml
- title: Conference Full Name
  year: 2026
  id: UNIQUEID2026         # used as HTML element ID — must be unique
  link: https://...
  deadline: "2026-MM-DD 23:59:59"
  timezone: GMT+2          # use momentjs timezone strings; UTC±N also supported
  date: Month D - Month D, YYYY
  place: City, Country
  sub: CON                 # see _data/types.yml for valid values
```

Optional fields: `note` (displayed below date/place).

Valid `sub` values (defined in `_data/types.yml`): `CON`, `JRN`, `WP`, `IND`, `CV`, `DM`, `ML`, `NLP`, `RO`, `SP`.

Use `deadline: "TBA"` when the deadline is not yet announced.

### Data-entry conventions

- When a conference has multiple submission rounds (e.g. an "extended deadline"), use the final/latest one — that's the existing convention throughout the file (e.g. ACVSS entries use the extended deadline, not the original).
- Pick an accurate `timezone` for the event's actual location (e.g. `GMT+3` for East Africa, `GMT+1` for West Africa, `UTC-12` to represent an "Anywhere on Earth" deadline) rather than defaulting to `GMT+2`. A large fraction of older entries default to `GMT+2` regardless of actual location and are known-inaccurate as a result — don't propagate that habit into new entries.
- Treat every existing `id` as permanent. Ids are embedded in `/conference?id=` URLs that may be bookmarked or indexed externally, so never rename one for cosmetic consistency — not for casing (`dl2019` vs `DLI2022`), not to fix a stale acronym spelling (`PAISS2021`/`PAISS2022` vs the later `PAAISS2025`), even when it visibly clashes with a newer year's naming convention. New entries should follow the current convention (uppercase acronym + year); old ones stay untouched.
- Keep an entry's `sub` type consistent with the same conference series' other years unless the event's actual format genuinely changed (checked at the time of writing: Deep Learning Indaba is `CON` across all years, RAIL is `WP` across all years).

### Known data-quality issues (left unfixed)

These were found while auditing the file but weren't corrected — each needs either external research or a judgment call this repo's maintainer should make, not a mechanical fix:

- `africa2020` entry: `title` is a bare `-Africa 2020` and `link` is the malformed `www.-africa.org` — looks like a dropped prefix word from the original aideadlin.es fork. True title/link unknown.
- `SAICSIT2020`: `date` reads "September 14-September 9, 2020" — the day range runs backwards, likely a typo, but the correct dates aren't confirmed.
- Date-string formatting is inconsistent across years (hyphen spacing, en dash vs hyphen, "to" vs "-", ordinal suffixes like "19th") — purely cosmetic, not worth a blanket rewrite without maintainer sign-off on a target style.

### Historical coverage status

As of 2026-07, every recurring series in the file has been checked for editions back to 2015 (see git log around July 2026 for the additions). Findings, so a future session doesn't redo this research:

- **Confirmed no edition exists before the file's current earliest entry**: SACAIR, RAIL, NASSMA, PAAISS/PAISS, EE-RDS, COSAA, SICSS-JIAS.
- **2015–2018 editions added**: Deep Learning Indaba (2017, 2018), AfriCHI (2016, 2018), DHASA (2017), AFRICON (2015, 2017), icABCD (2018), ISCMI (2017, 2018 — Africa-based editions only; 2015/2016 editions existed but were in Hong Kong/Dubai, excluded), ICDHT (2018 — this is actually the series' founding year, correcting an earlier assumption that 2019 was first).
- **2015–2018 editions found but deliberately NOT added**, pending primary-source verification: Data Science Africa (official site and Wikipedia disagree on exact dates in 3 of 4 years), SAICSIT (dates sourced via DBLP but almost no deadlines found), IST-Africa (ist-africa.org was unreachable during research; everything came from search snippets/mirrors), AI Expo Africa 2018 (key sources had gone 404, only indirectly corroborated). The Wayback Machine was unreachable during this research — re-attempting these four series through archive.org first is the natural next step if someone wants to close this gap.
- Nothing before 2015 has been investigated yet.

## Sorting the Data

After editing `_data/conferences.yml`, you can use the utility script to sort entries by deadline:

```bash
cd utils
pip install pyyaml pytz future
python process.py
# Review sorted_data.yml, then copy back to _data/conferences.yml
```

Note: `process.py`'s sort crashes with `pytz.exceptions.UnknownTimeZoneError` on any entry whose `timezone` is `GMT±N` (only `UTC±N` is handled) — and most of the file uses `GMT±N`. In practice this script currently cannot run to completion against `_data/conferences.yml` as it stands; when it fails, sort/insert new entries by deadline manually instead.

## Architecture

- `_data/conferences.yml` — the single source of truth for all conference entries
- `_data/types.yml` — defines conference categories/sub-types (name, `sub` code, color) used for filtering and badge colors
- `_layouts/home.html` — main page template. Liquid loops over `site.data.conferences` at build time to emit one hidden div per conference (with per-`sub` CSS classes) plus a matching inline `<script>` block that wires up jQuery Countdown / Moment-timezone for that conference's deadline. Filtering by `sub` is done client-side by toggling those CSS classes, not by rebuilding the page; the active filter set is persisted to `localStorage` via `store.js` and mirrored in the URL as `?sub=`
- `_pages/conference.html` — a single static page at `/conference/`. It is NOT generated per-conference: Liquid emits an `if (conf == "{{conf.id}}") {...}` block for every conference into one big client-side script, and on load JS reads the `?id=` query param to pick which block's data to display. `_plugins/data_page_generator.rb` exists in the repo (a generic Jekyll data-to-pages generator) but is currently unused — there's no `page_gen` entry in `_config.yml` wiring it up
- `_layouts/calendar.ics` — iCal feed template rendered as `ai-deadlines.ics`
- `_config.yml` — Jekyll site config; sets `domain`, `github_username`, `github_repo` used throughout templates
- `utils/process.py` — standalone Python 3 script to sort and deduplicate `conferences.yml`
- `static/js/` — vendored JS deps: jQuery Countdown, Moment + Moment-timezone (with full tz data), Bootstrap multiselect, `store.js` (localStorage wrapper), `ouical` (ICS export). No package manager/bundler — these are checked-in minified files loaded directly by the layouts
