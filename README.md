# eJP podcast summarizer

Watches a Slack channel in eJewishPhilanthropy's workspace. Paste a YouTube link,
podcast episode, tweet or article and the bot replies in-thread with the moments
an eJP editor would flag.

Third in the family, after
[`ji-podcast-summarizer`](https://github.com/LevelThreeLabsO/ji-podcast-summarizer)
and [`circuit-podcast-summarizer`](https://github.com/LevelThreeLabsO/circuit-podcast-summarizer).
Same plumbing; different beat.

## The beat

`BEAT_RUBRIC` at the top of `poll.py`. Derived from the relevance model in
[`eJP-Daily-Watcher`](https://github.com/LevelThreeLabsO/eJP-Daily-Watcher), which
measured eJP's own archive — 1,102 "What We're Watching" items and 435 "Major
Gifts" items, Jan 2021 – Sep 2026 — rather than guessing.

On-beat: philanthropy and funding; named donors, foundations and funders; Jewish
communal institutions and their leadership; the field's structural questions
(sustainability, wealth transfer, pipeline, burnout, governance); Israel–diaspora
relations as a philanthropic question; institutional response to antisemitism.

Two filters do the real work, and both exist to stop this duplicating the JI bot:

1. **Philanthropy means charitable money.** "Funding" is not enough. Government
   budgets, sovereign wealth, state capital projects, corporate capex and startup
   rounds are not philanthropy however large the sums. Somebody must be giving
   something away, and the recipient must be a nonprofit or communal institution.
2. **It must be this sector.** A Jewish or Israel topic alone doesn't qualify, and
   neither does philanthropy with no Jewish communal or donor connection. Israeli
   politics, a Knesset vote, war coverage, a candidate on Gaza — those are JI's.

Explicitly vetoed: antisemitism *incidents*, combat, crime, political horse-race.
These appear in **zero** of the 1,102 archive items. eJP covers the sector's
institutional response to such events, never the event itself. That veto is the
cleanest line between eJP and JI.

Verified both directions before first deploy:

| Input | Result |
| --- | --- |
| Qatari PM on Gaza, Iran and state budgets | `0 moments` — correctly rejected |
| eJP's own Met Council story | 3 points: the $25M campaign and Sen. Sutton's $1M matching grant, an 18% surge in food requests against 40% price inflation, 2M lbs of kosher food distributed |

The first version of the rubric **failed** that negative test — it surfaced Qatari
state budgets, having read "capital projects" and "funding" as philanthropy. Filter 1
exists because of that failure. If you edit the rubric, re-run both directions.

Returns **up to** five moments, not exactly five: an off-beat item costs the channel
more than a missing one.

## How it runs

Single-shot; one GitHub Actions tick per run. Cadence is the `*/5` schedule cron plus
a self-chain that re-dispatches at the end of each run via the built-in `GITHUB_TOKEN`.
State lives in `watcher_state.json`, committed back each run.

Failures are silent by design — no error noise in the channel. Transient problems
(network, quota, a sleeping Mac) raise `TransientError` and retry; only genuinely
permanent ones are marked done.

### Gemini quota

The free tier meters **per project per model**, 20/day each. This bot has its own API
key (separate project) and leads with `gemini-3.1-flash-lite`, while JI leads with
`gemini-3.6-flash` and Circuit with `gemini-3-flash-preview`, so no two drain the same
bucket first. Models are **pinned deliberately**: `gemini-flash-latest` silently
followed Google to a model with a much tighter limit, and the quota shrank with no code
change.

On quota exhaustion the helper moves to the next model rather than failing — a 429 is a
`ClientError`, and treating all `ClientError`s as fatal once made the whole fallback
chain dead code.

## Setup

Needs a Slack app in **eJP's own workspace** (`T01ANPY5Z5Y`) — the JI and Circuit
tokens will not work. Create at api.slack.com via "Sign in to another workspace"; that
site has its own login, separate from the Slack client. Bot scopes: `channels:history`,
`chat:write`, plus `channels:read` if you want the channel ID discoverable.

| Secret | Notes |
| --- | --- |
| `SLACK_BOT_TOKEN` | New — from the eJP-workspace app |
| `SLACK_CHANNEL_ID` | New — the channel to watch |
| `GEMINI_API_KEY` | eJP's own key, separate Google project |
| `GROQ_API_KEY` | Shared — Whisper fallback for uncaptioned video |
| `YT_TRANSCRIPT_IO_TOKEN` | Shared — primary YouTube path |
| `CLIPMAKER_URL` | Shared — Mac fallback; rotates, see below |
| `CLIPMAKER_AUTH_TOKEN` | Shared |

Then `/invite` the bot to the channel. Until the Slack secrets are set the workflow
exits 0 with "nothing to do yet" rather than failing every few minutes.

`~/bin/clipmaker-tunnel.sh` on the Mac fans the rotating tunnel URL out to every
consumer repo — **add this repo to its `REPOS` array** or its `CLIPMAKER_URL` will go
stale on the next restart.

## Local use

```bash
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
export GEMINI_API_KEY=...
.venv/bin/python poll.py --url "<any URL>"   # one URL, no Slack post
.venv/bin/python poll.py --dry-run           # poll the channel, print only
```

`--url` is how you check a beat edit. Run it against something on-beat and something
off-beat, and confirm both answers.
