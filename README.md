# Comix.to Extension (Mihon/Tachiyomi)

A [Mihon](https://mihon.app)/Tachiyomi extension for reading manga, manhwa, and manhua from **[comix.to](https://comix.to)**.

Built on the [keiyoushi/extensions-source](https://github.com/keiyoushi/extensions-source) build infrastructure, stripped down to a single extension with full API reverse-engineering.

---

## Features

- **Browse** — Popular (most followed), Latest (recently updated)
- **Search** — By title with advanced filters
- **Filters** — Type (Manga/Manhwa/Manhua/Other), Status, Demographic, 31 Genres, Year range, Min chapters, 11 sort options
- **Manga details** — Author, artist, genres, tags, status, score, alternative names
- **Chapter list** — Full pagination (handles 700+ chapter manga), deduplication (official > votes > recency), scanlator filter
- **Reader** — Full image loading with proper headers
- **Settings** — 10 configurable options (content rating defaults, blocked genres, deduplication, score display, etc.)

### Settings

| Setting | Description |
|---------|-------------|
| Default content rating | Safe, Suggestive, Erotica, Pornographic |
| Default type filter | Manga, Manhwa, Manhua, Other |
| Default demographic filter | Shounen, Seinen, Shoujo, Josei |
| Blocked genres | Hide specific genres from chips |
| Deduplicate chapters | Keep best version per chapter (official > votes) |
| Scanlator filter | Show only chapters from specific scanlators |
| Show alternative names | Display alt titles in description |
| Show extra info | Type, status, year, rating, latest chapter |
| Show tags in genre chips | Include format tags (Long Strip, Full Color, etc.) |
| Score display position | Don't show / Top of description / End of description |

---

## Installation

### Add the repo to Mihon

1. Open Mihon
2. Go to **More → Settings → Browse → Extension repos → Add**
3. Paste:
   ```
   https://raw.githubusercontent.com/marbou92/MHRepo/main/index.min.json
   ```
4. Restart Mihon
5. Go to **Browse → Extensions** and install **Comix**

### Manual install (APK sideload)

Download the latest APK from the [Actions tab](https://github.com/marbou92/MHExtensions/actions) → click a successful CI run → Artifacts.

---

## How It Works

Comix.to uses a custom API protection layer that was reverse-engineered:

1. **Request signing** — Every API request needs a `_` query parameter: a base64url signature computed via a 3-stage chained S-box substitution over the request path + sorted query string.

2. **Response encryption** — Chapter list and chapter page responses are encrypted (`x-enc: 1` header). The body is `{"e": "<base64url>"}` and decryption reverses the S-box using inverse lookup tables.

3. **API envelope** — All responses are wrapped in `{"status":"ok","result":...}` which is unwrapped by an OkHttp interceptor.

Both the sign and decrypt functions are implemented natively in Kotlin — no WebView or JS engine needed.

### API Endpoints

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/manga?params` | Manga list (popular / latest / search) |
| `GET /api/v1/manga/{hid}` | Manga details |
| `GET /api/v1/manga/{hid}/chapters` | Chapter list (paginated) |
| `GET /api/v1/chapters/{chapterId}` | Chapter pages (image URLs) |

---

## Repository Structure

This repo (`MHExtensions`) contains the **source code**. The **published extension repo** is at [marbou92/MHRepo](https://github.com/marbou92/MHRepo).

```
MHExtensions/
├── src/all/comix/                    # The Comix extension
│   ├── build.gradle.kts
│   ├── res/mipmap-*/ic_launcher.png  # Extension icon (5 densities)
│   └── src/eu/kanade/tachiyomi/extension/all/comix/
│       ├── Comix.kt                  # Main HttpSource + sign/decrypt + settings
│       └── ComixDto.kt               # kotlinx.serialization DTOs
├── core/                             # Gradle plugin / DSL
├── common/                           # Shared runtime utilities
├── compiler/                         # @Source annotation processor
├── lib/                              # Reusable helpers
├── lib-multisrc/                     # 61 multisrc themes
├── .github/
│   ├── workflows/
│   │   ├── build_push.yml            # Auto debug builds on push
│   │   ├── build_pull_request.yml    # PR checks
│   │   └── release_publish.yml       # Manual release + publish to MHRepo
│   └── scripts/
│       └── publish-repo.py           # Generates index.min.json
├── setup-signing.sh                  # Signing key generator
├── ext-bootstrap.py                  # Extension scaffolder
└── SIGNING.md                        # Signing setup guide
```

---

## Building

### Prerequisites

- JDK 17
- Android SDK (with build-tools)
- Internet access (Gradle downloads dependencies on first build)

### Build debug APK

```bash
./gradlew :src:all:comix:assembleDebug
```

Output: `src/all/comix/build/outputs/apk/debug/*.apk`

### Build release APK (signed)

```bash
./gradlew :src:all:comix:assembleRelease \
  -PALIAS=<alias> \
  -PKEY_STORE_PASSWORD=<password> \
  -PKEY_PASSWORD=<password>
```

Output: `src/all/comix/build/outputs/apk/release/*.apk`

---

## CI Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| **CI** (`build_push.yml`) | Auto on push to `main` | Builds debug APK for testing |
| **PR check** (`build_pull_request.yml`) | Auto on PR | Builds debug to verify PRs |
| **Release & Publish** (`release_publish.yml`) | Manual only | Builds signed release APKs + publishes to [MHRepo](https://github.com/marbou92/MHRepo) |

### Release process

1. Test with debug build (auto-built on push)
2. Go to **Actions → Release & Publish → Run workflow**
3. CI builds signed release APKs
4. CI publishes to `MHRepo` (generates `index.min.json`, uploads APKs + icons)
5. Mihon auto-updates the extension

---

## License

Apache License 2.0

## Disclaimer

This project does not have any affiliation with comix.to or the content providers available.
This project is not affiliated with Mihon/Tachiyomi. All credits to the codebase go to the original keiyoushi contributors.
