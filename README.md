# APOD viewer

Wanted to open APOD and just see the picture of the day, big, centered, nothing else,
with left/right arrows for the other days. This is what I landed on.

## The idea

One static html file, no server, no browser extension, no api key. It pulls the day's
data from nasa's own json endpoint and draws a black page with one `<img>`.

While digging around I found that the new science.nasa.gov/apod is WordPress, and it
exposes the whole archive without auth and with CORS turned on:

    https://science.nasa.gov/wp-json/wp/v2/apod-basic/260929     # date as YYMMDD

that returns `{date, title, hdurl, url, media_type, explanation, credit, permalink}`.
Checked the headers by hand on 2026-09-30: it answers `access-control-allow-origin`
for whatever origin you send, including `null`, so a plain `file://` page can fetch it
straight from the browser. The list form
(`.../apod-basic?per_page=100&page=N`) reports 11422 entries, i.e. back to 1995.

## Why not the other obvious options

- `api.nasa.gov/planetary/apod` - the "official" api. Wants a signup for a real key and
  the DEMO_KEY is shared; I got 429s twice while testing it. The wp-json endpoint above
  is the same data with none of that.
- userscript that strips the real apod page down to the image. The redesigned page is a
  heavy JS build with churny class names, and its prev/next links are per-article slugs.
  Fragile, and pointless once the json endpoint exists.
- scraping / proxying the old `apod.nasa.gov/apod/apYYMMDD.html` pages. Those dated urls
  are dead now, they 301 to the new site (verified). A local proxy server just to work
  around a problem we don't have.

Side note: the apod images live on `assets.science.nasa.gov` and load in `<img>` fine
regardless of CORS, so only the json needed to be fetchable.

## How it works

- layout is flex-centered, img is `object-fit: contain` capped at the viewport. black
  background so letterboxing is invisible.
- prev/next just move one calendar day and refetch. nasa occasionally has no entry for a
  date (sep 31, early 90s gaps, whatever), so on a miss it keeps stepping in the same
  direction until something comes back.
- the date lives in the url hash (`#2026-09-29`), so any day is bookmarkable and
  back/forward buttons page through the archive for free.
- every day is cached in localStorage once fetched (past apods don't change) and the two
  neighbors are preloaded, so hitting the arrow feels instant after first visit.
- on a cold day change we slap a tiny blurred preview up first (blur-up). the full urls
  are cropped server side with w/h+fit=clip, so the preview scales w and h by the same
  factor or the aspect would change and the picture would visibly pop when the real one
  lands. if the file is already in the http cache - preloaded neighbors, revisits - we
  skip the preview entirely and it snaps in sharp. no scale/transform on the preview
  either, that was a bug that caused a resize flicker. one landmine found while
  testing: image.decode() silently never resolves on at least one engine, which would
  have left people stuck on a blurred picture forever, so the sharp swap races decode
  against a 1.5s timeout and happens no matter what.
- videos: the new api only gives a poster image for `media_type: video`, no youtube
  embed url like the old one did. so a video day shows the poster and a "watch on nasa"
  link. fine for my use.
- the explanation text comes back as html from nasa, so it goes through a template tag,
  scripts/on-attrs/javascript: hrefs get stripped, links forced to new tab.

## Controls

keyboard: `←` / `→` days, `i` explanation, `h` hide chrome, `f` fullscreen, `esc` reset.
clicking the picture also hides the chrome.

phone: swipe left/right for days, press and hold the picture for the explanation,
tap it to hide/show the overlay (first tap after holding the picture closes the panel).

there's no real fullscreen api on ios, so on an iphone the move is share -> add to
home screen, it launches without any browser bars anyway. android chrome hides its
own bar as soon as you swipe. sizing uses dvh so the picture always fits the part
of the screen you can actually see.

## Open it

live on github pages: https://mohumb.github.io/apod/ - that's the one I bookmarked.
the file also works straight off disk (`file:///home/barabadi-ai/apod/index.html`)
since nasa's endpoint lets any origin fetch it, pages is just more convenient.

It works, I use it. Poked at the weird cases too (a nonexistent date like sep 31, a
video day, going back to 1995) and they behave. If nasa ever moves that wp-json
endpoint, the rss feed `science.nasa.gov/feed/apod-basic/` has the same fields for the
last 30 days, so it'd be a small fix, not a rewrite.
