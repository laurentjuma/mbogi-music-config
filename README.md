# mbogi-music-config

Runtime configuration for the Mbogi Music Android app, so that things which change often — where ads
appear, what they show, whether they run at all — can change without shipping a release.

Two files:

- **`config.json`** — the app's own settings. Small, always read.
- **`ads.json`** — the ad slots. Only fetched when `config.json` says ads are on.

And the same pair again under **`debug/`**, which is what debug builds of the app read. Release
builds read the ones at the root. They are separate files on the same site, so experimenting in
`debug/` cannot reach anybody's phone — see [Debug config](#debug-config).

The app fetches both at startup and stores them in internal storage, preferring that copy on later
runs. It also ships a copy of each as a bundled asset, which is what a fresh install shows before the
first fetch lands and what any install falls back to when the fetch fails. There is never no config,
only a stale one.

Because the fetch happens after the first screens have drawn, an edit here reaches a running app on
its **next start**, not immediately.

## `config.json`

```json
{
  "version": 1,
  "ads": {
    "on": true
  }
}
```

`version` describes the file format, not the app. Bump it only if an existing section changes shape
in a way older builds cannot read. Each file carries its own.

Everything else is a **section**, keyed by what it configures. `ads` is the only one so far. A new
section is a new key: adding `"player": { }` alongside `ads` does not disturb anything, because each
consumer reads only the key it knows and ignores the rest. Older app builds that have never heard of
a section simply skip it, so the file can run ahead of what is installed.

`ads.on` is the universal switch. Setting it to `false` stops every slot at once, whatever `ads.json`
says — and the app does not even fetch `ads.json`. This is the one to reach for to pull all
advertising immediately.

## `ads.json`

```json
{
  "version": 1,
  "slots": {
    "queue": {
      "on": true,
      "type": "image",
      "media": "https://cdn.mbogimusic.com/ads/queue-banner.png",
      "click": "https://www.mbogimusic.com",
      "requiresMbogiContent": true
    }
  }
}
```

### Switching one screen off

`slots.<id>.on` stops one screen while leaving its entry intact, so the creative, the click URL and
everything else survive to be switched back on later. It defaults to `true` when left out.

Deleting a slot entry does the same, but loses the settings with it. Use `on` to pause a screen,
deletion to retire it.

### Which screens can show an ad

A **slot** is one place in the app that can hold an ad. A slot named under `slots` shows what its
entry says; **a slot left out of the file shows nothing**. So this list, minus anything switched
off, is the list of places ads appear.

| Slot id | Screen |
| --- | --- |
| `feed_item_list` | A podcast's episode list |
| `playlist` | A playlist's contents |
| `album_details` | An album's tracks |
| `episodes_list` | Base episode list (any screen that has not named its own slot) |
| `all_episodes` | Episodes |
| `all_songs` | Local Media → Songs |
| `all_videos` | Local Media → Videos |
| `inbox` | Inbox |
| `playback_history` | Playback history |
| `favorites` | Favorites |
| `favourite_episodes` | Favourite episodes |
| `add_to_playlist` | Add to playlist |
| `queue` | Queue |

A slot id that the installed build does not recognise is ignored, so naming a screen that does not
exist yet is harmless.

### What a slot shows

| Field | Applies to | Required | Meaning |
| --- | --- | --- | --- |
| `on` | all | no, defaults `true` | `false` switches this slot off without losing its settings |
| `type` | all | yes | `admob`, `image` or `video` |
| `adUnitId` | `admob` | yes | Google AdMob banner unit id |
| `media` | `image`, `video` | yes | URL of the image or video to display |
| `click` | `image`, `video` | no | URL opened in the browser when tapped; omit to make it non-tappable |
| `requiresMbogiContent` | all | no, defaults `true` | See below |

### Creative size

Every ad is drawn as a banner: the full width of the screen, 60dp tall, the same strip an AdMob
banner occupies. `image` and `video` are scaled to fill that strip and cropped where they do not
fit, so **supply banner-shaped creatives** — roughly 8:1. Anything squarer arrives with its top and
bottom cut off.

Good sizes are 1200×150, or the standard banner units 468×60 and 320×50. `video` plays muted and on
a loop.

An entry missing a field it needs is dropped and that slot shows nothing — one bad entry does not
take the others down with it. A malformed file means no ads at all rather than a crash.

### `requiresMbogiContent`

Most screens are lists that can hold a mix of Mbogi Music items and other content. When this is
`true`, the ad only appears if the list actually holds a Mbogi item; on a screen showing nothing but
imported podcasts, or on an empty list, the slot stays blank. Set it to `false` to show the ad
regardless of what the list holds.

It defaults to `true`, which is the behaviour every slot had before the setting existed.

## What this file cannot decide

An ad only appears if, on top of a slot entry that is switched on:

1. The user has not removed ads — Settings → User interface → Ads. Ads are on by default and the
   button there turns them off; no config here can override that choice. It also stops `ads.json`
   being fetched at all, so someone who has removed ads never asks this repository for anything.
2. For AdMob slots, the Google Mobile Ads SDK initialised at app start, which it only does when ads
   were already on for that user.

## Debug config

`debug/config.json` and `debug/ads.json` are the same two documents, in the same shape, read only by
debug builds. Which one a build reads is decided at compile time by `CONFIG_BASE_URL` in the app's
`build.gradle`, so it is not something a running app can be talked into changing.

Edit them as freely as you like: a broken slot, a switch left off, a creative that is not ready.
Nothing there reaches an installed release build. The debug creatives are deliberately orange and
labelled `DEBUG`, so a screenshot says which config produced it without anyone having to check.

Both pairs are validated and deployed by the same push, so a malformed debug file fails the workflow
and stops the release config deploying with it. If that ever becomes annoying, split the validation
step rather than dropping it.

To iterate faster than a push, serve the files from your own machine and point a debug build at them:

```sh
python3 -m http.server 8765           # in a checkout of this repository
```

```sh
./gradlew :app:installPlayDebug -PconfigBaseUrl=http://10.0.2.2:8765/    # 10.0.2.2 is the host, from an emulator
```

Restart the app to pick up an edit. `-PconfigBaseUrl` only affects the debug build type, and is not
committed anywhere; leave it out and you are back on `debug/`.

## Editing

These are read by a program, so they have to stay valid JSON — no trailing commas, no comments.
Validate before pushing:

```sh
python3 -m json.tool config.json > /dev/null && python3 -m json.tool ads.json > /dev/null && echo ok
```

To move a screen from your own creative to an AdMob banner, change its entry in place:

```json
"queue": {
  "on": true,
  "type": "admob",
  "adUnitId": "ca-app-pub-…/1039341195",
  "requiresMbogiContent": false
}
```

To switch a screen off, set its `on` to `false` in `ads.json`. To stop all advertising at once, set
`ads.on` to `false` in `config.json`.

## Serving

This repository is private. The app reads the config from

```
https://mbogi-music-config.netlify.app/
```

Pushing to `main` runs `.github/workflows/publish.yml`, which validates both files and deploys them
there. Only `config.json` and `ads.json` are published — `netlify.toml` copies those two into the
publish directory, so everything else here, this README included, stays off the public site.

Keeping the repository private hides the history and anything else in it, not the config itself. The
app fetches with no credentials, so whatever is published is world-readable by anyone who unzips the
app and follows the URL. Put nothing here that is not meant to be read.

Responses are cached for five minutes, so an edit takes up to that long to reach a device that asks
for it — on top of the app only asking at startup.
