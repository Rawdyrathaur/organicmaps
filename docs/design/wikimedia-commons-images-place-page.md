# Design Specification

## Wikimedia Commons Images in Place Page 

**Related Issue:** [organicmaps/organicmaps#3252](https://github.com/organicmaps/organicmaps/issues/3252)
**Data Source:** Existing `FMD_WIKIMEDIA_COMMONS` metadata (already in .mwm files)

---

## 1. Overview

This proposal adds optional visual enrichment to place pages by displaying images from Wikimedia Commons when available.

The feature uses existing `wikimedia_commons=*` metadata already stored in `.mwm` files, fetches images on-demand, and respects all existing network and storage constraints.

---

## 2. Constraints & Requirements

### 2.1 Architectural Constraints

* Offline map data (`.mwm`) must not increase in size
* No changes to map generation pipeline
* No background network activity
* No bulk or predictive downloads
* Existing `NetworkPolicy` must be respected
* Feature must fail silently

### 2.2 UX Constraints

* No new global settings
* No mandatory user actions
* UI must gracefully handle absence of images


---

## 3. Data Sources

### 3.1 Metadata Source

* Uses existing `wikimedia_commons=*` metadata
* Metadata is already stored in `.mwm` files
* Only `File:*` values are supported
* `Category:*` values are ignored

No Wikidata runtime queries are used.

### 3.2 Runtime Data Access

**Layer:** Place page presentation layer (Android UI fragments).
**Object:** `MapObject` — the runtime container for POI data.
**Flow:**
1. User opens place page
2. Fragment reads `FMD_WIKIMEDIA_COMMONS` field from `MapObject`
3. Returns Commons identifier (e.g., `"File:Example.jpg"`)
4. Identifier is sent to Wikimedia Commons API to fetch actual image URL and attribution

**Rationale for runtime API call:** CDN URLs for Commons images change over time and cannot be stored offline. Only the stable identifier is stored in `.mwm` files.

---

## 4. Proposed Solution

### 4.1 High-Level Description

When a user opens a place page:

* If an image for that place is already cached, it is displayed
* Otherwise, and only if network access is permitted, a single image is fetched from Wikimedia Commons and cached locally

Images are fetched **strictly on demand**, one per place. No image downloads are triggered by map movement, search results, viewport changes, or background processes.

### 4.2 API Integration

Send Commons identifier to:
```
https://commons.wikimedia.org/w/api.php?action=query&titles=File:X.jpg&prop=imageinfo&iiprop=url|user|extmetadata&iiurlwidth=800
```

Response contains 800px thumbnail URL and attribution metadata (author, license). CDN URLs change over time, requiring runtime API call rather than storing URLs offline.

---

## 5. Runtime Flow

1. User opens a place page
2. Application reads `FMD_WIKIMEDIA_COMMONS` metadata from `MapObject`
3. Local cache is queried
4. If cache hit → image is loaded
5. If cache miss:
   * Network policy is checked
   * If permitted → single Wikimedia API request is made
   * Image is downloaded on background worker thread
   * Image is cached
6. Image is displayed with attribution overlay

---

## 6. Network Behavior

### 6.1 Network Integration

Uses existing network policy system before every image fetch. **No new dialogs are added.**

| Connection Type | Behavior |
|-----------------|----------|
| **WiFi** | Fetch automatically |
| **Mobile data (ALWAYS)** | Fetch automatically |
| **Mobile data (ASK)** | Use existing system prompt, persist user's choice |
| **Mobile data (NEVER)** | Image section hidden, no fetch attempted |
| **Roaming (disabled)** | Image section hidden, no fetch attempted |

If network access is denied, the image section is removed silently. No error messages, no retries, no background queuing. This reuses the same permission flow already used by map downloads.

### 6.2 Precedent

This matches existing pattern used by map downloads.

---

## 7. Local Storage

### 7.1 Image Identifier Storage

Image identifiers (like `"File:Example.jpg"`) are **already embedded** in `.mwm` files as `FMD_WIKIMEDIA_COMMONS` metadata. This metadata already exists—no changes to offline map data or generation pipeline. Actual images are NOT stored offline. They are fetched on-demand at runtime and cached locally after download.

### 7.2 Cache Implementation

**Where stored:** Application-managed cache directory automatically cleaned when device storage is low.
**Size limits:** 50 MB limit (approximately 200 cached images at ~250 KB each).
**Eviction strategy:** LRU (Least Recently Used). When cache reaches the limit, oldest unused images are deleted to make room for new ones.
**Key format:** SHA-256 hash of the Commons identifier ensures collision-free filenames.
**Persistence:** Cache is intended to survive app restarts but remains expendable—Android OS may clear it under storage pressure without user intervention.


---

## 8. UI Design

A single optional image section inside the place page. If no image is available or loading fails, the section is removed entirely—no empty space, no error messages, no placeholders. The UI provides visual context for landmarks without disrupting the existing layout when images are absent.

### 8.1 Image Display

* **Image size**: 800px width thumbnail, height varies by aspect ratio
* **Placement**: Dedicated section between "Bookmark" and "Wikipedia" sections
* **Container**: Uses the same place page section container pattern as other sections
* **Attribution**: Semi-transparent white bar at bottom showing:
  * Author name
  * License (CC BY-SA)
  * "View on Commons" link
* **Text size**: 11sp for attribution

### 8.2 Full-Screen Viewing

**No full-screen image viewer is included.** Tapping the image opens the device browser to the full Commons page. This matches the existing behavior of Wikipedia links in place pages. Adding an in-app image viewer would require additional UI complexity (zoom, pan, close button) and increase maintenance burden without significant user benefit.

### 8.3 Multiple Images

Only the first image is shown. No carousel (single image only), no preloading (additional images are not fetched), and no precaching (only the displayed image is cached). Multiple images would require UI controls (navigation dots, swipe gestures) and additional network requests, increasing complexity. A single representative image provides sufficient visual context.

### 8.4 Loading & Failure

* Loading indicator shown briefly (small spinner)
* Timeout: short, fixed duration (initially ~5 seconds)
* On failure:
  * Section is removed
  * No error messages shown
  * No placeholder
  * No "tap to load" button

No automatic retries are scheduled; the image is re-attempted only if the user reopens the place page while online.

### 8.5 User Interaction

None. Image appears automatically when `NetworkPolicy` allows.

---

## 9. Resource Usage

All numeric values are indicative defaults and subject to adjustment during implementation and review.

### 9.1 Storage

| Resource | Size | Notes |
|----------|------|-------|
| Offline storage | 0 bytes | Metadata already in .mwm files |
| Code size | ~15 KB | Fragment class, API wrapper, cache manager |
| Runtime memory | ~2-4 MB | One decoded 800px bitmap (RGB_565 format) |
| Disk cache | 50 MB limit | ~200 cached images |

### 9.2 Memory Management

* Format: RGB_565 (2 bytes/pixel) not ARGB_8888 (4 bytes/pixel)
* Lifecycle: Released when view is destroyed
* Pressure: Responds to low memory callbacks

---

## 10. Offline Behavior

| Condition | Result |
|-----------|--------|
| Cached image | Displayed |
| Not cached + offline | No image shown |
| Offline maps | Fully functional |
| No network | Image section not shown |
| NetworkPolicy denies | Image section not shown |

Images are treated as optional enrichment.

---

## 11. Privacy

### 11.1 Data Sent

* Commons identifier (e.g., `"File:X.jpg"`)
* Thumbnail width (800px)
* No user ID, no cookies, no analytics

### 11.2 Recipient

Wikimedia Foundation API servers. Same domain already used for Wikipedia article fetching. Wikimedia is non-profit with public privacy policy (non-tracking, no ads).

### 11.3 IP Visibility

Standard for HTTP requests. User opts in via `NetworkPolicy` dialog on mobile data.

### 11.4 Telemetry

None. No tracking of which images are fetched.

---

## 12. Explicit Non-Goals

The following are **out of scope**:

* **Bulk image downloads**: No regional or area-based downloads
* **Background sync**: All downloads happen when user opens place page
* **Carousel or galleries**: Only first image shown even if API returns multiple
* **Full-screen image viewer**: Tapping image opens browser to Commons page (same as current wikimedia_commons link)
* **Preloading**: Images fetched only when user opens place page, not preemptively
* **Generator changes**: No modifications to map data generation pipeline
* **Wikidata runtime integration**: No API calls to Wikidata. Only Commons identifiers already in .mwm files
* **Category: support**: Only `File:X.jpg` processed. `Category:Y` values ignored (would require selecting one image from hundreds)

---

## 13. Implementation Outline

The following steps are subject to review:

1. Create `PlacePageImagesFragment` following pattern from `PlacePageWikipediaFragment`
2. Add fragment container to place page layout between bookmark and wikipedia sections
3. Read `FMD_WIKIMEDIA_COMMONS` metadata from `MapObject` when place page opens
4. Check network policy before HTTP request
5. Execute API call and image decode on background worker thread
6. Use existing HTTP client for network request
7. Marshal decoded bitmap to main thread
8. Release bitmap when view is destroyed

---

## 14. Failure Handling

All failures are silent to the user (no error toasts or dialogs):

| Scenario | Behavior |
|----------|----------|
| No network connection | Image section not shown |
| NetworkPolicy denies | Image section not shown |
| API returns HTTP error | Spinner disappears after 5s timeout |
| Invalid JSON response | Silent failure, no image shown |
| Image download fails | Silent failure, no image shown |
| Decode fails (corrupt image) | Silent failure, no image shown |
| Cache full | LRU eviction, new image cached |
| Low memory event | In-memory bitmap released, re-fetched if place page reopened |

**Rationale:** Image display is optional enrichment. Failure should not disrupt core map functionality or clutter UI with error messages.

---

## 15. Trade-offs

* Does not improve first-time offline experience (only visited POIs have cached images)
* Image coverage depends on OSM tagging quality (`wikimedia_commons=*` tag completeness)
* Cache content is user-specific and incomplete
* Limited visual coverage compared to bulk preloading

These trade-offs are consistent with existing Wikipedia integration, which also provides content only for visited places when online.

---

## 16. Rationale

This design:

* **Preserves offline-first principles**: Core map functionality unchanged when no network
* **Scales naturally with user behavior**: Cache grows based on actual usage patterns
* **Avoids infrastructure costs**: No hosting, no moderation, relies on Wikimedia Commons
* **Minimizes maintenance burden**: Reuses existing networking, threading, and place page architecture
* **Respects user control**: Network access governed by existing user preferences
* **Fails gracefully**: Silent failure preserves UX consistency

---

## 17. Precedent in Codebase

This follows established patterns:

* **Wikipedia content**: Pre-generated in .mwm files, displayed on-demand in place page fragment
* **Map downloads**: User-initiated, foreground service with progress notifications
* **NetworkPolicy usage**: Applied consistently across all network features
* **Fragment architecture**: Modular place page sections using fragment containers

---

## 18. Summary

This specification defines an on-demand, user-initiated image enrichment mechanism for place pages, using existing Wikimedia Commons metadata and respecting all current architectural and policy constraints of Organic Maps.

Images are treated as optional visual enrichment that enhances place recognition without compromising the offline-first experience.
