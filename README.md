# Comix.to Extension (Mihon/Tachiyomi)

A fork of [keiyoushi/extensions-source](https://github.com/keiyoushi/extensions-source) stripped down to a **single extension**: [comix.to](https://comix.to) — a manga/manhwa/manhua reading site.

All other 1,356 extensions have been removed. Only the build infrastructure (`core`, `common`, `compiler`, `lib`, `lib-multisrc`, Gradle plugin) and the Comix extension remain.

---

## What's Included

```
extensions-source/
├── core/                    # Gradle plugin / DSL ("keiyoushi {}" block)
├── common/                  # Shared runtime utilities
├── compiler/                # Annotation processor for @Source
├── lib/                     # Single-source reusable helpers
├── lib-multisrc/            # 61 reusable themes (madara, mangathemesia, keyoapp, ...)
├── gradle/                  # Gradle wrapper + version catalogs + build-logic
├── ext-bootstrap.py         # Scaffolding script for new extensions
├── CONTRIBUTING.md          # The extension-writing guide (from keiyoushi)
├── settings.gradle.kts      # Configured to load ONLY src:all:comix
└── src/
    └── all/
        └── comix/           # ⭐ The Comix.to extension (the only one)
            ├── build.gradle.kts
            ├── res/
            │   ├── mipmap-xxxhdpi/ic_webview.png
            │   └── values/strings.xml
            └── src/eu/kanade/tachiyomi/extension/all/comix/
                ├── Comix.kt       # Main HttpSource + sign/decrypt
                └── ComixDto.kt    # kotlinx.serialization DTOs
```

---

## The Comix Extension

Reads manga, manhwa, and manhua from **comix.to** directly inside Mihon/Tachiyomi.

### Features

- **Popular** — most-followed series
- **Latest** — recently updated series
- **Search** — by title, with filters (type, status, content rating, 8 sort options)
- **Manga details** — author, artist, genres, tags, description, status
- **Chapters** — chapter number, name, scanlation group, approximate dates
- **Page reader** — all images load directly in the app

### How It Works

Comix.to is a React SPA with a JSON API at `/api/v1/` that is protected by a custom anti-scraping layer:

1. **Request signing** — every GET to `/manga*` or `/chapters/*` needs a `_` query param: a base64url-encoded signature computed via a **3-stage chained S-box substitution** over the request path + sorted query string.

2. **Response encryption** — chapter list/page responses carry `x-enc: 1`. The body is `{"e": "<base64url-encrypted>"}`. Decryption reverses the S-box using **inverse lookup tables**.

Both were extracted from the site's minified JS and reimplemented natively in Kotlin (no WebView/JS engine needed). See `src/all/comix/src/.../Comix.kt` for the full implementation.

| Endpoint | Purpose | Encrypted? |
|----------|---------|------------|
| `GET /api/v1/manga?params` | Manga list (popular / latest / search) | No |
| `GET /api/v1/manga/{hid}` | Manga details | No |
| `GET /api/v1/manga/{hid}/chapters` | Chapter list | Yes (`x-enc`) |
| `GET /api/v1/chapters/{chapterId}` | Chapter pages (image URLs) | Yes (`x-enc`) |

---

## Building

### Prerequisites

- **JDK 17**
- **Android SDK** (with `build-tools` for `aapt` — needed by the publish script; not needed for a plain debug build)
- Internet access (Gradle will download dependencies on first build)

### Build the APK

```bash
cd extensions-source

# Make the wrapper executable (if needed)
chmod +x gradlew

# Build the Comix extension (debug — no signing key needed)
./gradlew :src:all:comix:assembleDebug
```

The APK will be at:

```
src/all/comix/build/outputs/apk/debug/*.apk
```

For a release build (needs a signing keystore):

```bash
# Generate a keystore (one-time)
keytool -genkeypair -v \
  -keystore signingkey.jks \
  -alias myrepo \
  -keyalg RSA -keysize 2048 \
  -validity 10000

# Build release
./gradlew :src:all:comix:assembleRelease \
  -PALIAS=myrepo \
  -PKEY_STORE_PASSWORD=<your_store_pw> \
  -PKEY_PASSWORD=<your_key_pw>
```

### Install on your phone

1. Transfer the built `.apk` to your Android device.
2. Open it and allow installation from unknown sources.
3. Open Mihon → **Browse** → find **Comix** → enable it.

---

## Publishing Your Own Repo (Optional)

If you want to serve the extension to Mihon via a repo URL (instead of side-loading the APK), you need a second repo (the "extensions" repo) and the GitHub Actions CI:

1. **Create a second GitHub repo** (e.g. `my-extensions`) with an orphan branch called `repo`.

2. **Edit `.github/scripts/publish-repo.py`** in this source repo:
   - Change the `APK_BASE_URL`, `JAR_BASE_URL`, `ICON_BASE_URL` to point to your `my-extensions` repo.
   - Change the `signingKey` value to your key's SHA-256 fingerprint.
   - Change `name` and `contact` to your details.

3. **Edit `.github/workflows/build_push.yml`** to push to your `my-extensions` repo instead of `keiyoushi/extensions`.

4. **Set GitHub Actions secrets** in this source repo:
   - `SIGNING_KEY` — base64 of your `.jks` keystore (`base64 -w0 signingkey.jks`)
   - `ALIAS` — your key alias
   - `KEY_STORE_PASSWORD` — keystore password
   - `KEY_PASSWORD` — key password

5. **Push** — CI builds, signs, and publishes to your `my-extensions` repo branch.

6. **Add to Mihon**: More → Settings → Browse → Extension repos → Add, then paste:
   ```
   https://raw.githubusercontent.com/YOU/my-extensions/repo/index.min.json
   ```

---

## Adding More Extensions

This fork only ships Comix, but the full keiyoushi infrastructure is intact. To add another extension:

1. **Scaffold a new extension** using the bootstrap script:
   ```bash
   python ext-bootstrap.py -n "Some Site" -l en -u "https://somesite.com" -c SAFE
   ```

2. **Edit `settings.gradle.kts`** to include it:
   ```kotlin
   loadIndividualExtension("all", "comix")
   loadIndividualExtension("en", "somesite")   // add this line
   ```
   Or switch back to auto-discovery:
   ```kotlin
   loadAllIndividualExtensions()
   ```

3. **Implement the scraping logic** in the generated `.kt` file. See `CONTRIBUTING.md` for the full guide, and look at the [keiyoushi/extensions-source](https://github.com/keiyoushi/extensions-source) upstream for 1,356 working examples.

---

## License

    Copyright 2015 Javier Tomás

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.

## Disclaimer

This project does not have any affiliation with comix.to or the content providers available.

This project is not affiliated with Mihon/Tachiyomi. Don't ask for help about these extensions at the
official support means of Mihon/Tachiyomi. All credits to the codebase goes to the original keiyoushi contributors.
