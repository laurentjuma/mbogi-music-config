# mbogi-music-config

Runtime configuration for the Mbogi Music Android app. One file, `config.json`, read by the app so
that things which change often — where ads appear, what they show — can change without shipping a
release.

The app currently ships a copy of this file as a bundled asset (`app/src/main/assets/config.json`)
and reads it at startup. Fetching it from this repository at runtime is the next step; the bundled
copy stays as the fallback for a first run, a failed fetch, or no network.

## Shape

```json
{
  "version": 1,
  "ads": { }
}
```

`version` describes the file format, not the app. Bump it only if an existing section changes shape
in a way older builds cannot read.

Everything else is a **section**, keyed by what it configures. `ads` is the only one so far. A new
section is a new key: adding `"player": { }` alongside `ads` does not disturb anything, because each
consumer reads only the key it knows and ignores the rest. Older app builds that have never heard of
a section simply skip it, so the file can run ahead of what is installed.

## The `ads` section

```json
{
  "ads": {
    "slots": {
      "queue": {
        "type": "admob",
        "adUnitId": "ca-app-pub-…/1039341195",
        "requiresMbogiContent": true
      }
    }
  }
}
```

### Which screens can show an ad

A **slot** is one place in the app that can hold an ad. A slot named under `slots` shows what its
entry says; **a slot left out of the file shows nothing**. Omission is how a screen is turned off,
so this list is the list of places ads appear.

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
| `type` | all | yes | `admob`, `image` or `video` |
| `adUnitId` | `admob` | yes | Google AdMob banner unit id |
| `media` | `image`, `video` | yes | URL of the image or video to display |
| `click` | `image`, `video` | no | URL opened in the browser when tapped; omit to make it non-tappable |
| `requiresMbogiContent` | all | no, defaults `true` | See below |

`image` is drawn at full width, keeping its aspect ratio. `video` plays muted and looping.

An entry missing a field it needs is dropped and that slot shows nothing — one bad entry does not
take the others down with it. A malformed file means no ads at all rather than a crash.

### `requiresMbogiContent`

Most screens are lists that can hold a mix of Mbogi Music items and other content. When this is
`true`, the ad only appears if the list actually holds a Mbogi item; on a screen showing nothing but
imported podcasts, or on an empty list, the slot stays blank. Set it to `false` to show the ad
regardless of what the list holds.

It defaults to `true`, which is the behaviour every slot had before the setting existed.

## Two things that also have to be true

An ad only appears if, on top of a slot entry:

1. The user has **Show ads** on — Settings → User interface → Ads. It is off by default, and no
   config here can override it.
2. For AdMob slots, the Google Mobile Ads SDK initialised at app start, which it only does when that
   setting was already on.

## Editing

`config.json` is read by a program, so it has to stay valid JSON — no trailing commas, no comments.
Validate before pushing:

```sh
python3 -m json.tool config.json > /dev/null && echo ok
```

To move a screen from AdMob to your own creative, change its entry in place:

```json
"queue": {
  "type": "image",
  "media": "https://cdn.mbogimusic.com/ads/promo.png",
  "click": "https://www.mbogimusic.com/promo",
  "requiresMbogiContent": false
}
```

To switch a screen off, delete its entry.
