# Place Page Images

**Issue:** [#3252](https://github.com/organicmaps/organicmaps/issues/3252)

Display photos in place pages from multiple OSM-tagged sources, shown in a swipeable carousel. Images are fetched on-demand, cached locally, and fail silently if unavailable.

---

## Image Sources

Five OSM tags are supported, in priority order:

| OSM tag | FMD field | Coverage | Auth required |
|---|---|---|---|
| `wikimedia_commons=File:X.jpg` | `FMD_WIKIMEDIA_COMMONS` (41, existing) | 350K nodes | No |
| `panoramax=<UUID>` | `FMD_PANORAMAX` (51, new) | 93K nodes | No |
| `wikipedia=en:Article` | (via wikiparser, no new FMD) | 2.2M nodes | No |
| `image=https://...` | `FMD_IMAGE` (52, new) | 403K nodes | No |
| `mapillary=<id>` | `FMD_MAPILLARY` (53, new) | 424K nodes | OAuth2 |

Mapillary is implemented last due to its OAuth2 requirement.

---

## Runtime Image Fetch

### Wikimedia Commons
```
GET commons.wikimedia.org/w/api.php
  ?action=query&titles=File:X.jpg
  &prop=imageinfo&iiprop=url|user|extmetadata
  &iiurlwidth=800
```
Returns 800px thumbnail URL + author + license. No auth required.

### Panoramax
```
GET api.panoramax.xyz/api/pictures/<UUID>/sd.jpg
```
Direct image URL. UUID validated in generator (36-char format). No auth required.

### Wikipedia Lead Image
Extracted by `descriptions_downloader.py` (wikiparser) during map generation.
`beautify_page()` currently strips all images — extend it to capture the lead image `<figure>` `src`. Store filename alongside description text. No new FMD field needed; piggybacks on existing Wikipedia pipeline.

### image= tag
Raw URL stored as `FMD_IMAGE`. Validator in `osm2meta.cpp` accepts only `https://` and rejects known redirector/shortener domains. Displayed with "Source: image= tag" attribution. No license guarantee — display as-is with a note.

### Mapillary
```
GET graph.mapillary.com/<id>?fields=thumb_1024_url&access_token=...
```
Requires app-level OAuth2 token. Implemented last. Token stored in build config, not bundled in open-source repo.

**Privacy note:** Mapillary is owned by Meta. Every image fetch sends the Mapillary ID and user IP to Meta's servers. This is disclosed to the user via the existing NetworkPolicy prompt on mobile data.

---

## Generator Changes

Follow the 16-file pattern from commit `1308b11d75` (original `wikimedia_commons` addition):

**`libs/indexer/feature_meta.hpp`** — add new FMD IDs:
```cpp
FMD_PANORAMAX = 51,
FMD_IMAGE = 52,
FMD_MAPILLARY = 53,
```

**`libs/indexer/feature_meta.cpp`** — add to `TypeFromString()` and `ToString()`:
```cpp
else if (k == "panoramax")   outType = Metadata::FMD_PANORAMAX;
else if (k == "image")       outType = Metadata::FMD_IMAGE;
else if (k == "mapillary")   outType = Metadata::FMD_MAPILLARY;
```

**`generator/osm2meta.hpp/.cpp`** — add validators:
```cpp
static std::string ValidateAndFormat_panoramax(std::string v);
// UUID format: 8-4-4-4-12 hex chars
static std::string ValidateAndFormat_image(std::string v);
// Accept https:// only, reject known shorteners
static std::string ValidateAndFormat_mapillary(std::string v);
// Accept numeric ID strings only
```

**`android/sdk/.../Metadata.java`** — mirror new FMD IDs, update `@IntRange(from=1, to=53)`.

**`android/sdk/.../MapObject.java`** — add getters for new fields.

Remaining files follow the same pattern: layout XML, `PlacePageView.java`, icon asset, `pygen.cpp`, `feature_list.cpp`, `metadata_parser_test.cpp`, Qt dialog.

---

## UI Design

### Carousel Section

A new `PlacePageImagesFragment` is inserted **above** the Wikipedia section in the place page. It follows the same fragment architecture as `PlacePageWikipediaFragment`.

<p align="center">
  <img src="https://github.com/user-attachments/assets/7c7812cb-45c5-422b-9010-5c8f09ddb361" width="416"/>
  <img src="https://github.com/user-attachments/assets/51bac7da-35b3-4396-a936-ffb296844b92" width="389"/>
</p>


- **Single image available:** show image, no arrows, no dots
- **Multiple sources provide images:** carousel with swipe gesture + dot indicator
- **No images available:** section removed entirely, no empty space
- **Loading:** small centered spinner, removed on success or failure
- **Failure:** silent — section disappears, no error message

### Image Sizing

- Width: match parent (full place page width)
- Height: fixed 200dp
- Scale: `centerCrop`
- Format: RGB_565 to halve memory usage vs ARGB_8888

### Attribution Bar

Semi-transparent overlay at image bottom (height 28dp, 11sp text):
- Author name
- License short name (e.g. CC BY-SA 4.0)
- Tappable "View source" link → opens browser

---

## Network Behavior

All fetches go through existing `NetworkPolicy.checkNetworkPolicy()`. No new network dialogs.

| State | Behavior |
|---|---|
| WiFi | Fetch automatically |
| Mobile data (ALWAYS) | Fetch automatically |
| Mobile data (ASK) | Existing prompt, user choice persisted |
| Mobile data (NEVER) | Section hidden |
| Offline | Section hidden |
| Any fetch failure | Section hidden silently |

Images fetch in order of priority. Once one image loads successfully, remaining are fetched lazily as user swipes.

---

## Caching

- **Location:** App cache directory (OS-managed, cleared under storage pressure)
- **Size limit:** 100 MB. Images vary widely in size — Wikimedia Commons thumbnails (800px) are typically 150–400 KB, Panoramax SD photos 300–800 KB. At 100 MB the cache holds roughly 150–300 real-world images depending on content type.
- **Eviction:** LRU
- **Key:** SHA-256 of source URL or identifier
- **Memory:** Single decoded bitmap held while place page is visible, released on view destroy

---

## Implementation Phases

1. **Phase 1 — Wikimedia Commons** (existing FMD, open API): Build `PlacePageImagesFragment`, carousel UI, caching, attribution bar. All infrastructure built here.
2. **Phase 2 — Panoramax** (new FMD_PANORAMAX): Add generator support + plug into Phase 1 UI.
3. **Phase 3 — Wikipedia lead image** (wikiparser): Modify `descriptions_downloader.py`, store image name alongside description, fetch via Wikimedia Commons API.
4. **Phase 4 — image= tag** (new FMD_IMAGE): Add generator validator + plug in.
5. **Phase 5 — Mapillary** (new FMD_MAPILLARY): OAuth2 token management + plug in.

Each phase is an independent PR. Phases 2-5 require only the generator additions and plugging a new URL resolver into the existing carousel.

---

## Failure Handling

All failures are silent — no error toasts, no empty placeholders, no retry buttons.

| Scenario | Behavior |
|---|---|
| No network / offline | Section not shown |
| NetworkPolicy denies | Section not shown |
| API returns error | Section removed silently |
| Invalid/corrupt image | Section removed silently |
| Image download timeout | Section removed silently |
| Cache full | LRU eviction, new image cached |
| Low memory | Bitmap released, re-fetched on next open |

---

## Non-Goals

- No bulk/preemptive image downloads
- No background sync
- No in-app full-screen image viewer (tap opens browser)
- No Wikidata runtime queries
- No `Category:*` support for `wikimedia_commons` (would require selecting one image from hundreds)
- No increase in `.mwm` file size
