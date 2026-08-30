<p align="center">
  <img src="https://raw.githubusercontent.com/kiwibrowser/src.next/kiwi/kiwi_logo_circle.svg" alt="KiwiBrowser" title="KiwiBrowser" width="200" height="200"/>
</p>

<h1 align="center">Kiwi Reborn Ultimate</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Maintained-Yes-brightgreen" alt="Maintained">
  <img src="https://img.shields.io/badge/Engine-Chromium_148-blue" alt="Engine">
  <img src="https://img.shields.io/badge/Platform-Android-orange" alt="Platform">
  <img src="https://img.shields.io/badge/Adblock-Native_C%2B%2B-EA4AAA" alt="Adblock">
</p>

---

### 💡 Project Overview

**Kiwi Reborn Ultimate** is a feature fork of [raihanbhaii/kiwi-reborn](https://github.com/raihanbhaii/kiwi-reborn) (itself a maintained fork of the original Kiwi Browser, based on Chromium **148.0.7778.96**). This branch (`relixor-ultimate`) adds a Brave-inspired **native adblock engine**, **universal background playback**, and a cleaned-up, honest privacy and sync story — while preserving every existing Kiwi Reborn capability, including the full **Chrome extension support** that made Kiwi famous.

> [!IMPORTANT]
> Nothing from the existing Kiwi Reborn feature set was removed or replaced. All additions are additive and are individually toggleable in Settings.

---

### ✨ What This Fork Adds

#### 🛡 Native AdBlock (Brave-inspired, C++ engine)

A real, network-level filter engine — not a URL-blocker toy, and not a page-script hack:

* **Engine core** (`components/adblock`, self-contained C++, no content/ deps):
  * Parses **ABP / EasyList syntax**: `||host^` anchors, `^` separators, `*` wildcards, `@@` exceptions, `$important`, `$third-party`/`$first-party`, `$domain=`, resource-type options (`$script`, `$image`, `$stylesheet`, `$subdocument`, `$xmlhttprequest`, `$media`, `$websocket`, `$popup`, `$font`, `$ping`, …), and `redirect` rules approximated as blocks.
  * **Hash-indexed host-anchored matching** — candidate lookup is O(host-length) hash probes instead of a linear scan over 100k+ rules; matching runs on immutable `EngineData` snapshots so list updates never stall requests and requests never block updates.
  * **Cosmetic filtering**: generic + per-domain element-hiding selectors with `#@#` exceptions, plus `$generichide`/`$elemhide` suppression support.
* **Network blocking** (`AdblockURLLoaderThrottle`): every request that flows through Chromium's network service is checked *before any connection is made* and cancelled with `ERR_BLOCKED_BY_CLIENT` — the same architectural position Brave uses for its native blocker.
* **Cosmetic injection** (`AdblockTabHelper`): page CSS is injected per frame in an isolated world the page cannot see or fight.
* **Filter lists**: EasyList + EasyPrivacy (default on), Fanboy's Annoyance, EasyList Cookies, and a **custom list URL**. Lists are cached, atomically written, refreshed daily in the background, and parsed off-thread.
* **Control & transparency**: Settings → Adblock — master switch, per-list toggles, "update now", custom filters (one per line), a per-site **allowlist**, and live statistics (blocked requests / network rules / cosmetic rules). The pre-existing app-menu per-site ads toggle (Chromium `ContentSettingsType.ADS`) continues to work alongside the engine.

#### ▶ Universal Background Playback

Generic web media background play — **not** a YouTube hardcode:

* `WebMediaPlayerImpl::OnFrameHidden` keeps playback alive when the app loses focus or the screen turns off (skips the gesture-to-resume rule and the idle-pause timer).
* `Document::IsPageVisible` reports hidden pages as visible under the feature, so *site scripts* (e.g. YouTube's `visibilitychange` handler) cannot pause media via the Page Visibility API. Unload semantics and Chromium's timer/memory throttling are untouched.
* The flag is a persisted user toggle (Settings → **Background playback**, default on), propagated from the browser process into each renderer via `AppendExtraCommandLineSwitches`. It takes effect for newly spawned renderer processes (fully applied after a relaunch).
* Upstream Chromium's MediaSession integration keeps providing lock-screen / notification media controls, audio focus and Bluetooth headset buttons wherever the site exposes a MediaSession.

#### 🌙 Dark / AMOLED Themes (preserved & verified)

Kiwi Reborn's full theme engine is intact: **default, AMOLED, AMOLED grayscale, gray, gray grayscale, high-contrast** web-contents modes plus system-following dark UI, configurable in Settings → Night mode. This fork changes nothing here — it is documented so it isn't lost.

#### 🧩 Chrome Extension Support (preserved)

Kiwi's flagship desktop-extension support on Android (WebStore install flow, `webstorePrivate` wiring, `chrome://extensions`) is preserved untouched by this branch's changes.

#### 🔒 Privacy

* Audit result: the overlay adds **no new telemetry** to upstream Chromium. UMA histogram calls are the standard local instrumentation that only uploads with the user's metrics consent (Settings → Google services → usage & diagnostics).
* The adblock engine runs **fully local**: filter lists are fetched directly from their publishers (easylist.to or your custom URL) — no proxy, no MITM, no telemetry, no account.
* No analytics, crash-report ingestion change, or data-collection code was introduced by this branch.

#### 🔑 Google Account Sync (honest enablement)

Chromium's native Google services require per-developer API keys (Google ToS). The build now accepts optional keys without code changes:

1. Add `GOOGLE_API_KEY`, `GOOGLE_DEFAULT_CLIENT_ID`, `GOOGLE_DEFAULT_CLIENT_SECRET` as GitHub Actions secrets in your fork.
2. The workflow passes them into `gn gen`; builds without them stay key-free exactly as before.

Everything else about sign-in/sync plumbing that ships in the Kiwi Reborn base is preserved.

---

### 📬 What Came From Where (transparency)

| Area | Source |
|---|---|
| Chromium 148 engine, extension support, AMOLED/night modes, per-site ads toggle | **raihanbhaii/kiwi-reborn** (preserved) |
| Native adblock engine, network throttle, cosmetic injection, settings UI | **new in this branch** (Brave architecture as inspiration, own implementation) |
| Universal background playback (blink + browser plumbing + UI) | **new in this branch** |
| Optional Google API key plumbing | **new in this branch** (config only) |

> [!NOTE]
> [Relixor/Relixor](https://github.com/Relixor/Relixor) was inspected in depth as requested. Its `kiwi` branch tree is byte-identical to the Kiwi Reborn base apart from README/logo assets, and its other branches are the upstream Kiwi Chromium-bump history — i.e. its advertised features are the *inherited Kiwi feature set* (extension support etc.), not new implementations to port. This branch therefore implements the requested feature gaps directly.

---

### 🏗 Building

**Via GitHub Actions (recommended):** run the *Build Kiwi Reborn APK* workflow (`workflow_dispatch`) — it syncs Chromium at the pinned version in `CHROMIUM_VERSION`, overlays this tree, runs `gn gen` + `autoninja chrome_public_apk` for arm64 and uploads `ChromePublic.apk`. A full build needs the full 6h timeout; keep `symbol_level = 0`.

**Locally (Linux):**

```bash
git clone https://github.com/<you>/kiwi-reborn-ultimate.git kiwi-patches
mkdir chromium && cd chromium
gclient config --name=src https://chromium.googlesource.com/chromium/src.git --unmanaged
echo "target_os = ['android']" >> .gclient
gclient sync --no-history --nohooks --revision=src@$(grep CHROMIUM_MAJOR ../kiwi-patches/CHROMIUM_VERSION | cut -d= -f2 | tr -d ' ' | sed 's/^/148.0.7778./' >/dev/null; echo 148.0.7778.96)
cd src
for d in base chrome components content extensions net remoting services third_party ui; do
  cp -rf ../kiwi-patches/$dir/* ./$d/ 2>/dev/null || cp -rf ../kiwi-patches/$d/* ./$d/ 2>/dev/null || true
done
gclient runhooks
./build/install-build-deps.sh --android --no-prompt
gn gen out/arm64 --args='target_os="android" target_cpu="arm64" is_debug=false is_component_build=false is_official_build=false ffmpeg_branding="Chrome" proprietary_codecs=true android_channel="stable" chrome_pgo_phase=0 symbol_level=0 blink_symbol_level=0'
autoninja -C out/arm64 chrome_public_apk
# → out/arm64/apks/ChromePublic.apk
```

**Supported Android versions:** same as the Kiwi Reborn base (Android 8.0+ arm64; the build matrix in CI targets arm64 — extend `target_cpu` as needed for arm32/x86_64).

---

### 🧭 Development Notes

* This repository is an **overlay tree**: it contains the files that differ from (or are added to) upstream Chromium. The CI workflow copies it over a pinned Chromium checkout. Modified upstream files (e.g. `chrome/browser/BUILD.gn`, `chrome/browser/chrome_content_browser_client.cc`, `third_party/blink/renderer/core/dom/document.cc`) are full replacements of the pinned tag, so every edit must be based on the exact `CHROMIUM_VERSION`.
* New Java sources are registered in `chrome/android/chrome_java_sources.gni` (`kiwi_java_sources`), resources in `chrome_android/java_resources` (`chrome_java_resources.gni`) and JNI classes in `generate_jni("chrome_jni_headers")` inside `chrome/android/BUILD.gn`.
* New C++ lives in `components/adblock` (engine, unit-testable, no `content/` dependency) and `chrome/browser/adblock` / `chrome/browser/kiwi` (integration + JNI), registered in `chrome/browser/BUILD.gn` (`static_library("browser")`).
* Adblock UI strings live in `chrome/android/java/res/values/strings.xml` (keys prefixed `preferences_adblock_`).

### ⚠️ Known Limitations (honest list)

1. **Filter syntax subset**: scriptlet injection (`#$#`) and procedural selectors (`#?#`) are ignored; `csp=` rules are dropped; `redirect=` rules block instead of substituting an empty resource.
2. **Background playback scope**: the visibility spoof is global while enabled (all pages report visible to site scripts) — some sites' battery-saver tricks may notice. Renderer flag changes apply to newly spawned renderers; do a full relaunch after toggling.
3. **YouTube**: works via the generic Page-Visibility approach; YouTube can change its scripts at any time, and ad-in-video blocking depends on the filter lists, not on special-casing YouTube.
4. **Sync**: requires user-provided Google API keys (Google's ToS applies); without keys, sign-in UI is present but sync will not function.
5. **Testing**: the CI build validates compilation; on-device test coverage depends on the maintainer (see PR description for the manual test matrix).

---

### 📬 Requests & Feedback
Open an issue — feature requests are welcome.

---

<p align="center">
  <i>Kiwi Reborn Ultimate — maintained with ❤️ on top of Kiwi Reborn (kiwibrowser lineage) and the Chromium project.</i>
</p>
