# Experiment / Vary Site Content with GTM

Randomly bucket visitors into a persistent variation, swap page content per bucket with **zero flicker**, and log the exposure to GA4 — using nothing but a first-party cookie, a Custom Template tag, and a Custom HTML tag. No Optimizely. No Adobe Target. Free.

Blog write-up: https://drewspen.blogspot.com/2011/01/experiment-vary-site-content-with-gtm.html

Container export: [`experiment-vary-site-content-with-gtm.json`](https://drive.google.com/file/d/15ug-HSPF9xveBxdH9Km57vw-1qn3b9aF/view?usp=drivesdk)

## What this is

An image-swap A/B test built entirely in Google Tag Manager:

1. **Bucket assignment** — on page load, if the visitor has no bucket cookie yet, a Custom Template tag generates a random integer (0–3), writes it to a persistent first-party cookie, and pushes a dataLayer event.
2. **Content swap** — on DOM Ready, a Custom HTML tag finds the page's default **transparent 1×1 pixel** image and rewrites its base URL to one of four real images, based on the cookie value — while preserving the original size-suffix query string.
3. **Exposure logging** — a Custom Event trigger catches the bucketing tag's dataLayer push and fires a GA4 event (`experiment`) with the chosen bucket as both an event parameter and a user property.

The transparent-pixel default is the key anti-flicker technique: because the page's *default* image state is invisible rather than "the control image," there is nothing to visibly flash before the swap happens.

## Repository contents

| File | Description |
|---|---|
| `experiment-vary-site-content-with-gtm.json` | The GTM container export (Export Format Version 2). Import via **Admin → Import Container → Merge**. |
| `experiment-vary-site-content-with-gtm.html` | The full Blogger post HTML for this write-up, including the required illustrative image `<div>` block and inline `<style>`. |
| `experiment-vary-site-content-with-gtm.rtf` | RTF version of the same write-up, for offline reading or import into a word processor. |

## Container structure

### Folders

| Folder | Contains |
|---|---|
| `Analytics` | GA4 configuration tag, shared configuration/event settings variables, environment- and stream-routing variables, client ID and session ID utilities. |
| `Personalization and Experiments` | The random-bucket cookie tag, the image-swap Custom HTML tag, the GA4 exposure event tag, and every trigger/variable specific to this experiment. |

### Tags

| Tag | Type | Fires On | Purpose |
|---|---|---|---|
| `Google Analytics Configuration` | Google Tag (`googtag`) | Initialization, once per load | Standard GA4 config tag wired to environment-aware Stream IDs. |
| `Random Integer Cookie and dataLayer Push Tag` | Custom Template (`Random Integer Cookie and dataLayer Push`) | Initialization, filtered to fire only when no bucket cookie exists | Generates the random bucket (0–3), writes the persistent cookie, pushes the dataLayer event. |
| `Randomize Face Image` | Custom HTML | DOM Ready, filtered to this post's page path | Rewrites the transparent-pixel `<img>`/`<a>` to the bucketed image, preserving the size suffix. |
| `Randomize Face Send Event` | GA4 Event (`gaawe`) | Custom Event, filtered to the bucket cookie's dataLayer push | Sends the `experiment` GA4 event with `chosenFace` as an event parameter and user property. |

### Triggers

| Trigger | Type | Filter |
|---|---|---|
| `Random Integer Cookie and dataLayer Push Initialize` | Initialization | `{{Random Face Cookie Value}}` matches `^(undefined\|null\|false\|NaN\|)$` |
| `Randomize Face DOM Ready` | DOM Ready | `{{Page Path}}` starts with `/2011/01/experiment-vary-site-content-with-gtm.html` **(update this for your own site — see below)** |
| `Random Integer Cookie and dataLayer Push Event` | Custom Event | `{{_event}}` matches `.*`, further filtered so `{{is Random Face Cookie Name}}` equals `true` |

### Custom Templates

| Template | Kind | Source | Role |
|---|---|---|---|
| **Random Integer Cookie and dataLayer Push** | Tag | Built for this recipe | Core bucketing logic — idempotent random-integer cookie write + dataLayer push, with unit tests included in the template. |
| If Else If — Advanced Lookup Table | Variable | [github.com/sublimetrix/gtm-template-ifelseif](https://github.com/sublimetrix/gtm-template-ifelseif) | General multi-condition lookup used for BigQuery-safe client ID formatting and timestamp padding. |
| Timestamp | Variable | [github.com/luratic/Timestamp](https://github.com/luratic/Timestamp) | Wraps the sandboxed `getTimestampMillis()` API. |
| Get Root Domain | Variable | [github.com/mbaersch/get-root-domain](https://github.com/mbaersch/get-root-domain) | eTLD+1 extraction, kept for portability to non-Blogger platforms. |

### Key variables (Personalization and Experiments folder)

| Variable | Type | Notes |
|---|---|---|
| `Random Face Cookie Name` | Constant | `_i_rnd_face` — cookie name and dataLayer event name. |
| `Random Face Cookie Value` | 1st-Party Cookie | Reads the cookie directly. |
| `is Random Face Cookie Name` | Lookup Table | Gates the Custom Event trigger to only this experiment's dataLayer push. |
| `Randomize Face Image Cookie Lookup` | Lookup Table | Maps bucket value (0–3) to a real image base URL; all fallbacks resolve to the transparent-pixel URL. |
| `dataLayer randomInteger Value` | Data Layer Variable | Debug/Preview inspection of the pushed `randomInteger`. |

## The image-swap script (Custom HTML tag)

```html
<script>
var oldBase = "https://blogger.googleusercontent.com/img/a/AVvXsEierEu8lbNxsfUf...K9LM540AYfh";
var newBase = {{Randomize Face Image Cookie Lookup}};
try {
  var faceSrc = document.querySelector('img[src^="' + oldBase + '"]');
  if (faceSrc) {
    var oldSrc = faceSrc.getAttribute("src");
    var suffix = oldSrc.substring(oldBase.length); // e.g. "=w320-h320"
    faceSrc.setAttribute("src", newBase + suffix);
  }
} catch (err) { }
try {
  var faceHref = document.querySelector('a[href^="' + oldBase + '"]');
  if (faceHref) {
    var oldHref = faceHref.getAttribute("href");
    var suffix = oldHref.substring(oldBase.length); // e.g. "=s1"
    faceHref.setAttribute("href", newBase + suffix);
  }
} catch (err) { }
</script>
```

> **Two things you must change for your own implementation:**
> 1. `oldBase` is hard-coded to this blog's transparent-pixel image URL — replace it with the base URL of your own 1×1 placeholder.
> 2. The `Randomize Face DOM Ready` trigger's page-path filter (`/2011/01/experiment-vary-site-content-with-gtm.html`) is hard-coded to this post — update it to your own page's path.

## Extending the pattern to text content

The same flicker-free technique applies to swapping text instead of an image. Target a stable `id` on an empty (or safe-default-copy) `<div>` instead of an image URL:

```html
<!-- On the page: a container that starts with safe default copy -->
<div id="experiment-headline">Welcome to our site</div>

<script>
// {{Randomize Headline Text Cookie Lookup}} is a Lookup Table variable,
// keyed on the same bucket cookie value (0-3), returning the variant copy.
var headlineText = {{Randomize Headline Text Cookie Lookup}};
try {
  var headlineEl = document.getElementById('experiment-headline');
  if (headlineEl && headlineText) {
    headlineEl.textContent = headlineText;
  }
} catch (err) { }
</script>
```

Wire this as a second Custom HTML tag on the same DOM Ready trigger (or a copy of it scoped to your text-experiment page), and build a companion Lookup Table variable the same way as `Randomize Face Image Cookie Lookup`, but returning text strings instead of image URLs. Set its default/fallback to your safe on-page copy — that default plays the same role the transparent pixel plays for images.

## Why this is a free alternative to Optimizely / Adobe Target

| Capability | This GTM pattern | Paid platform |
|---|---|---|
| Bucketing | First-party cookie via sandboxed GTM APIs | Vendor script/cookie |
| Content delivery | DOM Ready Custom HTML tag, page-scoped | Vendor snippet, often synchronous/edge |
| Exposure logging & analysis | Standard GA4 event + user property — works with your existing reports and BigQuery export | Vendor's own analytics/reporting suite |
| Flicker mitigation | Transparent-pixel / safe-default-copy starting state | Vendor anti-flicker snippet |
| Statistical significance, bandits, edge rendering | ❌ Not included — build/borrow your own analysis | ✅ Built in |
| Cost | Free | Paid SaaS |

For a single-element experiment on a blog or marketing site, this trade is usually well worth it.

## How to import

1. Download `experiment-vary-site-content-with-gtm.json` from the [Google Drive link](https://drive.google.com/file/d/15ug-HSPF9xveBxdH9Km57vw-1qn3b9aF/view?usp=drivesdk).
2. In GTM: **Admin → Import Container** → upload the file → choose **Merge** (not Overwrite).
3. Update `Measurement Stream ID Production` / `Measurement Stream ID Development` constants to your real GA4 Measurement IDs.
4. Update the `Randomize Face DOM Ready` trigger's page-path filter to your own page.
5. Install your own transparent-pixel placeholder image on that page, and update `oldBase` in the `Randomize Face Image` tag plus every entry (and fallback) in the `Randomize Face Image Cookie Lookup` variable.
6. In GA4, register `chosenFace` as a custom dimension/user property under **Admin → Custom definitions** if you want it available for segmenting.
7. Test in GTM Preview mode with cookies cleared before publishing.

## License / attribution

Recipe and container built by Kent Spencer ([drewspen.blogspot.com](https://drewspen.blogspot.com/)). Gallery templates retain their original authors' licenses — see the links in the Custom Templates table above.
