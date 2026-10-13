# CCloud PRO 2.0.0 — Build & Distribution Guide

This guide takes the rebuilt **Fluent 2 / Acrylic Glass** UI from source to a
one-click `Setup.exe` on a Windows machine. Compilation of Windows desktop
targets **must happen on Windows** (Flutter uses the Visual Studio toolchain);
everything else — analyze, formatting, asset work — can run anywhere.

---

## 1. Prerequisites (Windows 10 or 11, x64)

| Tool | Version | Notes |
|------|---------|-------|
| Flutter SDK | **3.35.7** (Dart 3.9.2) | matches `pubspec.yaml` → `sdk: ^3.9.2` |
| Visual Studio 2022 | Community is fine | workload: **Desktop development with C++** |
| Inno Setup 6 | 6.3+ | only needed for the installer step |
| Git | any | for cloning / versioning |

Verify the toolchain:

```powershell
flutter doctor -v
```

All three desktop rows (Flutter · Windows toolchain · VS) must be green.

---

## 2. Restore dependencies

From the project root:

```powershell
flutter pub get
```

Key packages resolved: `provider`, `go_router`, `media_kit` (+`media_kit_video`,
`media_kit_libs_video`), `window_manager`, `flutter_acrylic`, `file_picker`,
`http`, `shared_preferences`, `path_provider`, `url_launcher`, `share_plus`.

Fonts are **bundled locally** (`assets/fonts` — Plus Jakarta Sans + Vazirmatn),
so there is no Google Fonts network dependency at runtime.

---

## 3. Quality gate (must be clean)

```powershell
flutter analyze
```

Expected output: `No issues found!` — the tree ships zero errors, zero
warnings, zero infos (curated lint rules live in `analysis_options.yaml`).

---

## 4. Compile the release build

```powershell
flutter build windows --release
```

Verify the complete payload in
`build\windows\x64\runner\Release\`:

```
ccloud_gui.exe          <- app executable
data\                   <- icudtl.dat, flutter_windows.dll, app.so, fonts, shaders
flutter_windows.dll
media_kit_libs_windows_video.dll   (mpv engine — bundled by media_kit_libs_video)
*.dll                   <- plugin DLLs (window_manager, file_picker, …)
```

> Smoke test: double-click `ccloud_gui.exe` — the frameless acrylic window
> should open with the CCLOUD PRO titlebar. Theme toggle, search, catalog,
> player, subtitle flyout and downloads are all live against the real API.

---

## 5. Build the standalone installer

```powershell
iscc installer\CCloudPro.iss
```

Output: `dist\CCloudPro-2.0.0-Setup.exe`

The script:

* bundles **everything** from the Release folder (exe, `data\`, all DLLs),
* registers Start-Menu + optional desktop shortcuts,
* ships the bundled app icon (`windows\runner\resources\app_icon.ico`),
* installs per-machine into `Program Files\CCloud PRO`,
* cleans runtime metadata on uninstall but **keeps user downloads**.

Distribute the single `Setup.exe` — no runtime prerequisites for end users on
Windows 10/11 x64.

---

## 6. Architecture map (what lives where)

```
lib/
├─ main.dart                     bootstrap: MediaKit, window_manager, acrylic, providers
└─ src/
   ├─ core/
   │  ├─ theme/                  AppPalette · AppTheme · ThemeManager (dark/light/system)
   │  └─ router/                 go_router graph + PlaybackRequest payload
   ├─ models/                    PRESERVED — MediaItem, Source, Season/Episode, Poster…
   ├─ repositories/              PRESERVED — API client + helper-server failover
   ├─ providers/                 PRESERVED — movie/series/genre/seasons/search/countries/tv
   ├─ platform/                  PRESERVED — VLCLauncher (hybrid player escape hatch)
   ├─ utils/                     PRESERVED — StorageUtils (favorites), IconGenerator
   ├─ services/                  NEW — Settings, Toasts, FavoritesService,
   │                                   ContinueWatchingService, DownloadManager (real engine)
   ├─ player/                    NEW — SRT/VTT/ASS subtitle parser
   └─ ui/
      ├─ shell/                  frameless titlebar · smart sidebar · glass shell
      ├─ widgets/                glass kit: panels, buttons, cards, hero, mirrors modal…
      ├─ screens/                home · catalog · detail · favorites · search ·
      │                          live TV · countries · player · settings
      └─ player/                 subtitle flyout drawer
```

**Preserved logic:** every repository/provider/model untouched (only
`print → debugPrint` lint hygiene). The new UI binds onto the same
`ChangeNotifier` contracts, so scrapping services, caching and HTTP failover
behaviour are identical to upstream CCloud-GUI.

**New capabilities:**

* real in-app download engine (parallel byte-range chunks, pause/resume,
  live MB/s speed, queue persistence),
* subtitle engine (embedded tracks + local `.srt/.vtt/.ass` upload,
  ±0.5 s sync calibration, S/M/L live sizing),
* hybrid player (media_kit in-app + `Open in VLC`),
* continue-watching persistence, favorites with live sidebar counter,
* glass design system with Dark Acrylic / Light Frosted themes.

---

## 7. Troubleshooting

| Symptom | Fix |
|---------|-----|
| `Unable to find suitable Visual Studio toolchain` | Install VS 2022 with the *Desktop development with C++* workload |
| `media_kit` DLLs missing after build | Run `flutter pub get` once more, then rebuild (libs are fetched by `media_kit_libs_video`) |
| Blank window on Windows 10 older builds | The acrylic effect auto-falls back to the painted gradient; update GPU drivers or disable *GPU hardware acceleration* in Settings → Playback |
| SmartScreen warning on Setup.exe | Expected for unsigned builds — click *More info → Run anyway*, or sign with your certificate via `signtool` |
| Downloads land in Documents | Settings → Download Manager → *Browse…* to relocate |
