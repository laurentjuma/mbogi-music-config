# mbogi-music-config

Runtime configuration for the Mbogi Music Android app, so that things which change often — where ads
appear, what they show, whether they run at all — can change without shipping a release.

Two files:

- **`config.json`** — the app's own settings. Small, always read.
- **`ads.json`** — the ad slots. Only fetched when `config.json` says ads are on.

And the same pair again under **`debug/`**, which is what debug builds of the app read. Release
builds read the ones at the root. They are separate files at separate URLs, so experimenting in
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
  },
  "explore": {
    "on": true,
    "podcasts": true,
    "radio": true,
    "tv": true,
    "blockedRadioFeed": []
  }
}
```

`version` describes the file format, not the app. Bump it only if an existing section changes shape
in a way older builds cannot read. Each file carries its own.

Everything else is a **section**, keyed by what it configures: `ads`, `update`, `explore`, `games`,
`message`.
A new section is a new key: adding `"player": { }` alongside them does not disturb anything, because
each consumer reads only the key it knows and ignores the rest. Older app builds that have never
heard of a section simply skip it, so the file can run ahead of what is installed — builds from
before `explore` existed show the screen as they always did, whatever this file says.

`ads.on` is the universal switch. Setting it to `false` stops every slot at once, whatever `ads.json`
says — and the app does not even fetch `ads.json`. This is the one to reach for to pull all
advertising immediately.

## The `update` section

The app is not distributed through Play, so nothing tells anyone a new build exists. This does:

```json
"update": {
  "on": true,
  "versionCode": 3050143,
  "name": "3.5.42 (build 6)",
  "url": "https://github.com/laurentjuma/mbogi-music-releases/releases/latest"
}
```

An app whose own `versionCode` is lower than the one here shows a dismissible dialog offering the
download. `on: false`, or leaving the section out, means nobody is told anything.

| Field | Required | Meaning |
| --- | --- | --- |
| `on` | no, defaults `true` | `false` stops the prompt without losing the rest |
| `versionCode` | yes | The `versionCode` of the build being advertised. Must be **higher** than the installed one or nothing happens |
| `name` | no | What the dialog calls it, e.g. `3.5.42 (build 6)`. Falls back to generic wording |
| `url` | yes | Opened in the browser. Point it at the release **page**, not an APK — there is one file per architecture and the reader has to choose |
| `force` | no, defaults `false` | Replaces the whole app with a full-screen download prompt. Read the warning below before using it |

### `force`

`force: true` stops the app being usable at all: a full-screen page with a Download button, no way
past it, and back leaves the app rather than returning to it. It is for a build that must not keep
running — one corrupting data, or talking to something that no longer exists — and not for
encouraging upgrades.

Two things to be sure of before setting it. **The `versionCode` must be one that people can actually
install**, or everyone is locked out with nowhere to go. And **you are locking out everyone below
that number**, including anyone whose device cannot run the new build.

The way back is this file: remove `force`, and each app frees itself on its next launch. The config
is still fetched while the screen is up, and the app does not use a cached copy of it without asking
the server first, so the fix is not held up behind a cache. It still takes a relaunch, and it still
needs the device to be online — someone offline stays locked out until they are not.

`versionCode` is the release workflow's run number added to a base, so it climbs with every release
even when `versionName` does not. Take it from the build you are advertising rather than guessing:
publishing a number nothing has reached leaves everyone permanently prompted for a download that
does not exist.

Publishing a release and announcing it are separate steps on purpose. A build that turns out bad is
simply never named here, and nobody is ever sent to it.

`debug/config.json` deliberately has **no** `update` section. A debug build is a different
application id from the release it would be told to download, so the new APK cannot install over it
and the prompt is one nobody can act on — and a debug build's `versionCode` is the bare base, below
every release, so it would be prompted forever. Add the section there while you are working on the
prompt itself, and take it out again afterwards.

## The `explore` section

What the Explore screen offers, and whether it is there at all:

```json
"explore": {
  "on": true,
  "podcasts": true,
  "radio": true,
  "tv": true,
  "blockedRadioFeed": []
}
```

| Field | Required | Meaning |
| --- | --- | --- |
| `on` | no, defaults `true` | `false` takes the whole screen away |
| `podcasts` | no, defaults `true` | `false` drops the Podcasts tab |
| `radio` | no, defaults `true` | `false` drops the Radio tab |
| `tv` | no, defaults `true` | `false` drops the TV tab |
| `blockedRadioFeed` | no | Radio stations to withdraw, by title or playlist URL |

Everything here fails **open**: a missing section, a missing field, a value of the wrong type all
mean visible. A config that cannot be read never hides the app.

`on: false` removes Explore from the bottom navigation and its overflow, from the nav drawer, from
the drawer customization dialog so a user cannot put it back, and from the hybrid side nav. Anything
still pointing at it — a saved last screen, a default page — lands on **Add Feed** instead, which is
the one screen offering what Explore did: the directory searches, add by RSS or m3u, OPML import.

The three tab switches drop one tab each. The tab strip disappears when a single tab is left, so the
page fills the screen. Turning all three off is the same as `on: false`.

### Withdrawing a tab takes its feeds with it

**This deletes subscriptions.** Hiding `radio` deletes every radio playlist the user has subscribed
to, hiding `tv` every TV one, and `on: false` deletes both — the feeds, their episodes, downloaded
media, queue entries and download log. It happens on the next start after the edit lands. Turning
the tab back on restores the tab, not the subscriptions; the stations return to the grid and can be
subscribed again.

That is deliberate: a station kept after its tab is gone cannot be found, refreshed or replaced, and
it goes on playing from a source we no longer offer.

**Podcasts are the exception.** Hiding `podcasts` hides the tab and deletes nothing, because a
podcast subscribed from Explore is an ordinary RSS feed, indistinguishable from one the user added
by search, by URL, or by OPML import. Deleting "all podcasts" would mean deleting a library built
elsewhere.

### `blockedRadioFeed`

For withdrawing one station rather than the whole tab:

```json
"blockedRadioFeed": [
  "Some Station",
  "https://raw.githubusercontent.com/…/stations/somestation.m3u"
]
```

An entry matches a station's **title or its playlist URL**, trimmed and ignoring case, so a station
can be pulled by name without looking up its stream address. It leaves the Explore grid, and if it
was subscribed the feed and everything in it is deleted, exactly as above.

Only playlists the app subscribed from Explore are eligible, so a podcast that happens to share a
station's name is never touched. Taking an entry back out returns the tile and stops the deleting;
what was already deleted stays deleted.

## The `games` section

Which games the app offers, and whether the Games screen is there at all:

```json
"games": {
  "on": true,
  "list": [
    { "on": true, "name": "cover-puzzle" },
    { "on": true, "name": "cover-reveal" },
    { "on": true, "name": "lyrical-genius" },
    { "on": true, "name": "reverse-musicology" },
    { "on": true, "name": "wave-jump" }
  ]
}
```

| Field | Required | Meaning |
| --- | --- | --- |
| `on` | no, defaults `true` | `false` takes the whole Games screen away |
| `list` | no | The games to offer, **in the order to offer them**. Leaving it out means every game the build has, in the order it registered them |
| `list[].name` | yes | The game's id — one of the five above |
| `list[].on` | no, defaults `true` | `false` withdraws that one game, same as leaving it out |

**A present `list` is the whole truth.** It decides both which games appear and what order they
appear in, so reordering the entries reorders the hub without a release. A game the build has and
the list does not name does **not** appear.

The consequence worth holding on to: **a game shipped after this file was last edited is invisible
until you add it here.** Releasing a new game is therefore two steps, and this is the second one.

Failing open still covers the case that matters, though. A missing `games` section, a section with no
`list`, or a file that cannot be parsed at all means *unspecified*, which is every game the build
has — so a broken config leaves the hub alone rather than emptying it. Only a `list` that is present
and readable takes control.

`on: false` removes Games from the bottom navigation and its More menu, from the nav drawer, and from
the drawer customization dialog so a user cannot put it back. Anything still pointing at it — a saved
last screen, a default page — lands on the home screen instead. The onboarding screen drops its games
page too, so a first run does not introduce a screen the app does not have; that leaves two pages
rather than three.

A game named here that the installed build does not have is ignored, so this may run ahead of what is
released: naming a game before it ships is harmless, and it appears the moment a build has it.

To withdraw a game, either set its `on` to `false` or delete its entry — both leave it out, and the
first says why. Prefer `on: false` when you mean to bring it back.

Switching every game off, or an empty `list`, is the same as `on: false`: a hub with nothing in it is
no hub, so the screen goes with them.

The config cannot *add* a game. A game is an activity, its resources and its own Gradle module, all
compiled into the build; this file chooses among what is already there and orders it. That is why
`name` has to match a `GameEntry` id exactly — the app is matching your string against what it
registered at startup, and a typo silently drops the game.

Unlike `explore`, withdrawing a game **deletes nothing**. A game keeps no library of its own; it
draws from the user's music each time it is played, and their scores are their own.

## The `message` section

For telling people something — an outage, a migration, a station that is not coming back:

```json
"message": {
  "id": "radio-move-2026-08",
  "title": "Scheduled maintenance",
  "body": "Radio stations move to a new provider tonight. Some may be briefly unavailable.",
  "action": {
    "label": "Read more",
    "url": "https://www.mbogimusic.com/status"
  },
  "repeat": false
}
```

| Field | Required | Meaning |
| --- | --- | --- |
| `on` | no, defaults `true` | `false` stops the dialog without losing the wording |
| `id` | yes | Identifies **this** message. Shown once per id — change it to say something new |
| `title` | no | Dialog title. Left out, the dialog is body-only |
| `body` | yes | What it says. No message without one |
| `action.url` | no | Opened when the button is tapped: a web link, or a deep link the app handles |
| `action.label` | no | What the button says. Falls back to `Ok` |
| `repeat` | no, defaults `false` | `true` shows it at every launch until the id changes |

The dialog appears on the first launch **after** the edit has landed, like everything else here, and
is recorded as seen only once the user answers it — a message is not lost to an app that restarted
while it was up. Editing the wording without changing the `id` reaches nobody who has already seen
it; a new `id` reaches everyone again.

An available update takes precedence: the two dialogs never stack, and the message waits for the
launch after the update prompt is out of the way. Leave the section out when there is nothing to say
— an empty `body`, or `on: false`, means no dialog.

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
| `games` | The Games list |
| `cover_puzzle` | Cover Puzzle, while it is being played |
| `cover_reveal` | Cover Reveal, while it is being played |
| `lyrical_genius` | Lyrical Genius, while it is being played |
| `reverse_musicology` | Reverse Musicology, while it is being played |

A slot id that the installed build does not recognise is ignored, so naming a screen that does not
exist yet is harmless.

The five games screens are not lists, so they report no Mbogi content and need
`"requiresMbogiContent": false` to show anything at all — see [below](#requiresmbogicontent). Leaving
it out defaults it to `true` and the slot stays blank, which looks exactly like a slot switched off.

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

A screen that holds no list at all — the games — always reports no Mbogi content, so `true` there
means the slot never draws. Those five entries have to say `false` explicitly.

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

Both pairs go live on the same push and are validated by the same run, so a malformed debug file
turns the run red without having held anything back — the release config is already served either
way. The two no longer share a fate, which is the good half of the trade: a broken `debug/` file
cannot delay a release config fix. It also means the red run is the only thing telling you, so read
it.

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

This repository is public, and the app reads these files from it directly:

```
https://raw.githubusercontent.com/laurentjuma/mbogi-music-config/main/
```

so `config.json` is at `…/main/config.json` and the debug pair at `…/main/debug/config.json`.
There is no build and no deploy: **a commit on `main` is the publish**. Nothing stands between an
edit and the app, which cuts out the step that used to be forgotten, and takes away the one that
used to catch mistakes — see below.

The whole repository is readable, not just the four JSON files. This README, the history, and
anything else added here are public. The app fetched with no credentials even when this was private,
so the config itself was always world-readable by anyone who unzipped the app and followed the URL;
what changed is that everything around it is too. Put nothing here that is not meant to be read.

`raw.githubusercontent.com` answers with `cache-control: max-age=300` and an ETag, the same terms
Netlify gave. An edit takes up to five minutes to reach a device that asks for it, on top of the app
only asking at startup.

### Invalid JSON now goes live, and it switches ads off

This is the one thing the move away from Netlify made worse, and it is worth knowing exactly.

`.github/workflows/publish.yml` still validates all four files, but it now runs **after** the push
and publishes nothing — the push already did that. A red run is a fire alarm, not a gate.

A commit that will not parse does not merely go stale. The app stores what it fetched **before** it
tries to parse it, so the bad file replaces the last good copy in internal storage, and reading it
back on the next launch gives the same unparseable text. The bundled asset is not a fallback here
either; it is only reached when the stored file cannot be *read*, not when it cannot be *parsed*.
What every device then runs is the empty config:

- **ads off**, everywhere, until a valid file is fetched
- no update prompt and no message, however they were set
- Explore and Games fully visible, because those fail open

Nothing rolls back on its own, and there is no cached copy left to roll back to. The way out is the
same as `force`: push valid JSON, and each app fixes itself on its next launch.

So validate before you push, not after:

```sh
for f in config.json ads.json debug/config.json debug/ads.json; do python3 -m json.tool "$f" > /dev/null || echo "BAD $f"; done && echo ok
```
