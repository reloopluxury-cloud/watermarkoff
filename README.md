# WatermarkOff

A fully offline video watermark remover. Import a video (max 10MB), draw a
box over the watermark, and the app removes it on-device using FFmpeg —
no internet connection, no upload, ever.

## What this is (and isn't)

This is a **real, working Flutter app skeleton** — not a mockup. It
implements the actual pipeline: import → size check → mask selection →
FFmpeg processing → save/share, matching the approved Stitch design.

What it is **not**: an AI-inpainting app. See "Upgrade path" below for how
this differs from the earlier AI-inpainting concept and what it would take
to get there.

## How watermark removal works right now (v1)

It uses FFmpeg's `delogo` filter, which reconstructs the masked region using
neighboring pixels across frames (temporal + spatial interpolation):

```
delogo=x=<X>:y=<Y>:w=<W>:h=<H>:show=0
```

- The rest of the video is re-encoded at `-crf 18` (visually near-lossless).
- Audio is stream-copied untouched (`-c:a copy`) — zero audio quality loss.
- This works well for logos/timestamps sitting on a static or simple
  background. It works less well for a watermark sitting over a busy,
  fast-moving background — the fill will be a plausible reconstruction,
  not the "real" pixels that were covered (since those are actually gone).

This is a completely legitimate, widely-used technique (it's literally
built into FFmpeg) and is honest about its limits — it is not magic AI
reconstruction, just very good interpolation.

## Project structure

```
lib/
  main.dart                     — app entry point
  theme/app_theme.dart          — colors/typography ported from the Stitch DESIGN.md
  models/mask_rect.dart         — fractional mask rectangle + pixel conversion
  services/video_service.dart   — file size check + the actual FFmpeg command
  screens/
    home_screen.dart            — import + size check
    file_size_warning_screen.dart
    mask_selection_screen.dart  — draggable/resizable box over video preview
    processing_screen.dart      — runs VideoService, shows progress
    preview_result_screen.dart  — before/after toggle, save to gallery
    settings_screen.dart
  widgets/watermark_off_bottom_nav.dart
```

## Fastest free path: get an installable APK without installing anything

You do **not** need the Play Store or any paid account to test this on your
Android phone. Android lets you install an APK file directly ("sideloading").
The only thing you need is a way to *build* that APK — and GitHub will do
that for you, for free, in the cloud. You don't need Flutter, Android
Studio, or a powerful computer for this path.

1. **Create a free GitHub account** at https://github.com (skip if you have one).

2. **Create a new repository:**
   - Click the "+" in the top right → "New repository"
   - Name it anything, e.g. `watermark-off-app`
   - Keep it Public or Private, either works — click "Create repository"

3. **Upload this project's files:**
   - On the new repo's page, click "uploading an existing file"
   - Unzip `watermark_off_app.zip` on your computer first
   - Drag in the `lib` folder, `android` folder, `assets` folder,
     `pubspec.yaml`, and `README.md` — then click "Commit changes"

   ⚠️ **Important:** the `.github` folder (which contains the build
   instructions) starts with a dot, so Windows Explorer / Mac Finder often
   **hide it by default** — it may not even show up to drag. Don't worry
   if it's missing from your upload. Instead, add it directly on GitHub's
   website:
   - On your repo page, click "Add file" → "Create new file"
   - In the filename box, type exactly: `.github/workflows/build-apk.yml`
     (typing the slashes like this makes GitHub create those folders for you)
   - Paste in this content, then click "Commit new file":
     ```yaml
     name: Build APK

     on:
       push:
         branches: [ main, master ]
       workflow_dispatch:

     jobs:
       build:
         runs-on: ubuntu-latest
         steps:
           - name: Checkout code
             uses: actions/checkout@v4

           - name: Set up Flutter
             uses: subosito/flutter-action@v2
             with:
               channel: 'stable'

           - name: Fill in missing platform folders (android/ios build scaffolding)
             run: flutter create --project-name watermark_off --org com.watermarkoff.app .

           - name: Install dependencies
             run: flutter pub get

           - name: Build release APK
             run: flutter build apk --release

           - name: Upload APK so you can download it
             uses: actions/upload-artifact@v4
             with:
               name: watermarkoff-apk
               path: build/app/outputs/flutter-apk/app-release.apk
     ```

4. **Let it build automatically:**
   - Click the "Actions" tab at the top of your repo
   - You should see a "Build APK" run start automatically (if it says
     Actions need enabling, click the button to enable them — it's free)
   - Wait 3-5 minutes for the green checkmark

5. **Download the APK:**
   - Click on the completed run
   - Scroll down to "Artifacts" → click `watermarkoff-apk` to download it
   - Unzip it — inside is `app-release.apk`

6. **Get it onto your phone and install it:**
   - Email it to yourself, or upload to Google Drive and download on your
     phone, or plug your phone into your computer via USB and copy it over
   - On your phone, tap the `.apk` file in your Files app
   - Android will ask permission to "install unknown apps" for that one
     source (e.g. Files or Gmail) — allow it, then tap Install
   - Open the app and test it — fully offline, exactly like the real thing

**Note for iPhone users:** this free sideloading path is Android-only.
Apple doesn't allow installing an app on iPhone without either a paid Apple
Developer account ($99/year) or a Mac + Xcode + a free Apple ID (which
limits the install to 7 days before it needs re-signing). There's no
truly free, permanent way to install a custom app on iPhone without one of
those — that's an Apple platform restriction, not something specific to
this app.

## Setup — getting this running on your machine

I wrote the Dart source and pubspec here, but I don't have the Flutter SDK
in this sandboxed environment to actually run `flutter create`/`flutter
build`, so you'll do that part locally (it's quick):

1. **Install Flutter** if you haven't: https://docs.flutter.dev/get-started/install

2. **Unzip this project**, then inside the folder run:
   ```bash
   flutter create --project-name watermark_off --org com.yourcompany .
   ```
   This generates the missing `android/`, `ios/`, etc. platform folders.
   It will NOT overwrite the `lib/`, `pubspec.yaml`, or `android/app/src/main/AndroidManifest.xml`
   I've already written, but if it complains about the manifest, keep the
   version in this repo (it already has the correct permissions and no
   `INTERNET` permission — intentional, to keep this honestly offline).

3. **Get packages:**
   ```bash
   flutter pub get
   ```

4. **Run it:**
   ```bash
   flutter run
   ```
   (Needs a connected Android device/emulator or iOS simulator with Xcode.)

### iOS note
Add this to `ios/Runner/Info.plist` for gallery saving to work:
```xml
<key>NSPhotoLibraryAddUsageDescription</key>
<string>Save your watermark-free video to your photo library.</string>
```

## Known things to finish before shipping

- **Mask box UX**: current drag/resize is basic (single corner handle).
  Consider adding all-4-corner handles and aspect-ratio-locked resize for
  polish.
- **Video trimming**: the "file too large" screen has a placeholder
  "Choose Another Video" button — wiring an actual in-app trimmer (e.g.
  `video_editor` or similar package) instead of forcing the user to trim
  externally is a nice v1.5 addition.
- **Share button**: stubbed — add `share_plus` for a native share sheet.
- **iOS FFmpeg binary size**: `ffmpeg_kit_flutter_new`'s full-GPL build adds
  meaningful app size. If binary size matters, look at their "min" or
  "audio" flavored packages if you don't need every codec.

## Upgrade path: swapping in AI inpainting later

If `delogo` quality isn't good enough on busy backgrounds, the place to
upgrade is entirely inside `VideoService.removeWatermark()` in
`lib/services/video_service.dart`. The rest of the app (mask drawing UI,
progress screen, before/after preview, save/share) doesn't need to change.

The general shape of that upgrade:
1. Convert an inpainting model (e.g. LaMa) to TFLite (Android) / Core ML (iOS).
2. Extract frames with FFmpeg instead of running `delogo` directly.
3. Run each frame's masked region through the model.
4. Cache/reuse the inpainted patch across frames where the watermark is
   static, instead of recomputing every frame (huge speed win).
5. Re-mux frames back into a video with FFmpeg, same as now.

This is a real project on its own — budget real time for model conversion,
performance tuning, and testing on actual devices.
