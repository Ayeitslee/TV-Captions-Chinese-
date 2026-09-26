# TV Caption Translate

An Android TV app for your Chromecast with Google TV that runs independently
on the device: it reads YouTube's own on-screen English captions via OCR,
translates them to Chinese, and shows Jyutping + Mandarin Pinyin underneath
in a floating overlay -- all without touching the YouTube app itself.

Everything runs on-device after first launch. No API key, no gateway, no
ongoing cost.

## How it works (three stages, all included here)

1. **Screen capture + overlay permission.** Uses Android's official
   `MediaProjection` screen-capture API (the same permission any screen
   recorder uses) and the "display over other apps" overlay permission.
2. **OCR.** Google's on-device ML Kit Text Recognition reads just the
   caption region of each captured frame -- free, offline, no account needed.
3. **Translation + Jyutping/Pinyin + overlay.** Google's on-device ML Kit
   Translation converts the English text to Chinese (downloads a small
   translation model once, over Wi-Fi, then works fully offline). Jyutping
   and Pinyin come from the same curated dictionaries already used by your
   Chrome extension (`dict-data.js`, `pinyin-data.js`, `dict-overrides.js`,
   etc., extracted into `app/src/main/assets/*.json`), so results are
   consistent with the extension.

## Build

You'll need Android Studio (or command-line Gradle + Android SDK), same as
building CaptureTest before.

```bash
cd tv-caption-app
./gradlew assembleDebug
```

(If `gradlew` isn't present, open the project folder in Android Studio once
and let it generate the wrapper, or run `gradle wrapper` first.)

The output APK will be at:
```
app/build/outputs/apk/debug/app-debug.apk
```

## Install onto your Chromecast with Google TV

Same `adb` flow as CaptureTest:

```bash
adb connect <your-chromecast-ip>:5555      # if not already connected
adb install app/build/outputs/apk/debug/app-debug.apk
adb shell am start -n com.aiviewer.tvcaption/.MainActivity
```

## Using it

1. Open the app (it appears on the Google TV home screen as "TV Caption
   Translate").
2. Tap **Overlay settings** and allow the app to display over other apps.
3. Tap **Start** and accept the screen-capture consent prompt -- Android
   requires this confirmation every session; it isn't something the app can
   skip.
4. Open YouTube and play a video with English captions turned on. Within a
   couple seconds, a translation box should appear near the bottom of the
   screen.

## Settings (toggle screen, like your extension's controls)

Tap **Caption settings** on the main screen for:
- Show/hide original English caption, Chinese translation, Pinyin, Jyutping independently
- Turn translation off entirely (keeps just the original caption line)
- Adjust the caption crop region (top/bottom/left/right, as fractions of the screen) if YouTube's captions aren't where the default expects
- Toggle the debug OCR log on/off

All settings save instantly and take effect on the very next processed frame — no need to stop and restart capture.

## Known limitations, honestly

- **Only reads captions that are already on screen.** If a video has no
  captions (auto-generated or otherwise), there's nothing for the OCR to
  read. Turn on YouTube's own captions (the CC button) first.
- **Caption position assumption.** The OCR crop region is currently set to
  the bottom-center ~20% of the screen (`CROP_TOP_FRACTION` etc. in
  `CaptionOverlayService.java`). If YouTube's captions sit somewhere else on
  your setup, adjust those four constants.
- **Chromecast dongle hardware is modest.** OCR + translation running
  continuously may be slow or laggy on this hardware. If it struggles, the
  code already throttles processing to once every 1.2 seconds
  (`PROCESS_INTERVAL_MS`) -- raising that further trades responsiveness for
  smoothness.
- **Translation quality.** ML Kit's on-device translation is decent but not
  as good as a large language model. It's what makes this fully free and
  offline, which was the priority here.
- **First run needs internet once** to download the ~30MB translation
  model. After that, no network is required.
- **Debug OCR log.** `DEBUG_LOG_OCR = true` in `CaptionOverlayService.java`
  writes every newly recognized caption line to
  `Android/data/com.aiviewer.tvcaption/files/ocr_log.txt` on the device, so
  you can pull it with `adb pull` and check OCR accuracy/tune the crop
  region. Set it to `false` once you're happy with it.
- **This reads pixels, not internal video streams**, so it works the same
  regardless of which app is showing captions -- but only if that app's
  captions are visible as burned-in/rendered on-screen text, which YouTube's
  are.
