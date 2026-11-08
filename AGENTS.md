# Assistant playbook

Read this whole file before you run `adb` against a treadmill. README.md is the human explanation. This file is the procedure.

The owner should be able to hand you this folder, a computer, and a USB-C data cable and have you reproduce the Watch home screen and the metrics bar. Stop and ask when a check below fails. Do not improvise past a failed check.

## Goal

On an original Peloton Tread running Android 10:

- Install `com.treadcover` (source in `app/`).
- Make `com.treadcover/.HomeActivity` the Home screen. It shows a "Watch" grid of launchable apps, with a faint Peloton mark in the upper right behind the tiles, and a small sleep button in that corner. After a minute with no touch, or a tap on that button, a screensaver keeps the display on: the mark spins once every two seconds and bounces the full screen. The metrics bar and the bottom controls hide while it runs. A corner hit bursts it into confetti for five seconds, then it starts again from the center on a new heading. Holding the sleep button plays that burst immediately. A tap brings the grid back.
- Run `CoverService`, a bottom bar with INCLINE, PACE, TIME, MILES, and MPH at the same text size and the same label color, plus pause, stop, and a minimize chevron. Minimized, the pill shows miles, not speed. Pause freezes time and distance and does not stop the belt. Stop asks first. I'm sure sets the belt to 0 mph, then shows a summary. Close on that summary clears the numbers and does not change the belt.
- Leave the belt controller, person-present stop, and hardware safety key alone.
- Keep the speed and incline knobs working.
- Be able to undo it with `app/revert.sh`.

## Hard limits

Never do any of these, even if the owner asks for "the same thing the Bike people do" or a Google account starts to look possible:

- Do not root the tablet, unlock the bootloader, flash a ROM, or replace the operating system.
- Do not disable, uninstall, or replace `com.onepeloton.affernetservice`. That process is the belt controller. Person-present auto-stop and the hardware safety key live there.
- Do not change `ENABLE_PERSON_PRESENT`. The app calls it with `1` (on). Do not pass `0`.
- Do not add new binder transaction codes, and do not probe the controller by guessing codes. If the existing calls fail, stop and tell the owner. The belt is not a device to fuzz.
- Do not call `SET_SPEED` or `SET_INCLINE` except through the knob handler or `stopBelt()` in `TreadController`. `stopBelt()` may only write speed 0. Do not raise speed from software.
- Do not factory-reset the tablet.
- Do not commit `app/debug.keystore`, `app/build/`, `app/device-state/`, any APK, or a PIN. The PIN is entered on the tablet and stored only in the app's private prefs (or, if the owner sets it that way, the `treadcover_pin` global setting). Never copy a PIN into the repo, a commit, or a screenshot you add here.
- Do not put the tablet serial number in the repo.
- Do not download paid or subscription apps from random mirror sites. Install only an APK the owner already has the right to install.
- Do not disable packages other than the three named in `app/apply.sh`.

`app/apply.sh` already refuses to run when `adb` sees no device, and when `com.onepeloton.affernetservice` is missing. Do not delete those checks.

## Before you change the tablet

1. Confirm one device:

   ```bash
   adb devices
   ```

   You want `device`. `unauthorized` means the owner must tap Allow on the tablet. `no devices` means the cable is charge-only, debugging is off, or the tablet is asleep. Stop. Do not look for a way to force debugging on.

2. Confirm it is the Tread this project was exercised on, or close enough to ask:

   ```bash
   adb shell getprop ro.product.model
   adb shell getprop ro.product.name
   adb shell getprop ro.build.version.release
   adb shell getprop ro.build.version.sdk
   adb shell getprop ro.product.cpu.abi
   ```

   Expected on the reference tablet: model `PLTN-TTR01` or similar Tread name, Android `10`, SDK `29`, ABI `arm64-v8a`. If it is a Bike, a Tread+, Android other than 10, or you are unsure, stop and tell the owner. Do not run `apply.sh`.

3. Confirm the controller is present and that you will not touch it:

   ```bash
   adb shell pm path com.onepeloton.affernetservice
   ```

   Empty output means stop.

4. Check free space. The data partition is about 4 GB. Keep at least 500 MB free.

   ```bash
   adb shell df /data
   ```

5. Tell the owner, in one sentence, what you are about to change: Home screen, three Peloton UI packages disabled, a metrics bar. Tell them `app/revert.sh` plus a reboot puts the Peloton screens back. Wait for a yes.

## Do the setup

From this folder, with `adb` on `PATH` and `ANDROID_HOME` pointing at an SDK that has `platforms/android-29` and `build-tools/34.0.0`:

```bash
app/setup.sh
```

That builds `app/build/treadcover.apk`, installs it, and runs `app/apply.sh`.

`apply.sh` records the previous Home component in `app/device-state/home-activity.txt` the first time it runs. That folder is gitignored. If the owner later uses a different tablet, delete `app/device-state` first so the next apply records that tablet's own Home screen.

If the build tools are missing, install the Android command-line SDK pieces named above. Do not switch the app to a higher target SDK. The manifest is `minSdkVersion` 29 / `targetSdkVersion` 29 because the tablet is Android 10.

## Check your work

- `adb shell dumpsys activity activities` shows `com.treadcover/.HomeActivity` resumed after you send Home (`adb shell input keyevent KEYCODE_HOME`).
- A screenshot shows the Watch grid and the five metrics at the same size.
- `adb shell pm path com.onepeloton.affernetservice` still returns a path.
- These three packages are disabled for user 0: `com.onepeloton.systempluginui`, `com.onepeloton.sensorstateindicator`, `com.peloton.activity`.
- `com.treadcover/.CoverService` is running.
- The owner can still turn the speed and incline knobs, and the numbers follow.
- Ask the owner to pull the safety key with the belt moving slowly and confirm the belt stops. Do not declare success before that. If the belt does not stop, run `app/revert.sh` immediately and tell the owner.

The first launch may show "Enter Tread Lock PIN". The owner types the PIN they set on the Tread. The app stores it on the tablet and reuses it, including after reboot. If they have no PIN, say so and stop rather than guessing one.

## What the three disabled packages are

- `com.onepeloton.systempluginui` — kiosk-style UI that was force-closing Netflix, Prime, and similar apps, and drew Peloton's top status bar.
- `com.onepeloton.sensorstateindicator` — the Tread Lock passcode keypad. Disabling it is UI only. It does not remove the safety key.
- `com.peloton.activity` — the member app. It draws "Activate your Tread" and the subscription nag, which is a secure system overlay an app-op cannot block.

`settings put global lock_service 1` lets the controller honor a registered pin-lock owner. The app registers as that owner (`com.onepeloton.lockservice` cookie inside `TreadController`) so the belt does not re-lock a few seconds after a successful unlock. That is the existing mechanism in this code. Do not invent a second one.

## Apps, YouTube, and Google

The Watch grid is every launcher app that is not hidden by the prefixes in `HomeActivity.HIDE_PREFIXES`. Apps already on the tablet show up after install. New apps show up on the next resume of Home.

On the reference tablet, Netflix, Prime Video, Disney+, HBO Max, Peacock, and Apple TV were already installed and launched under the bar. Paramount+ and Firefox were added later (see Paramount+ below). Prefer those. Apple TV, Netflix, and Prime Video do not need Google Play services. Disney+, HBO Max, and Peacock hang at launch if Play services is installed but its main process is dead (HBO Max ANR, Disney+ spinner, Peacock load screen; log: `Timeout waiting for provider com.google.android.gms.chimera`).

If Play services, Services Framework, and the Play Store are already installed, this is the state that made all of those apps open on the reference tablet:

```bash
adb shell pm grant com.google.android.gms android.permission.INTERACT_ACROSS_USERS
adb shell pm disable-user --user 0 com.google.android.gsf
adb shell pm disable-user --user 0 com.android.vending
```

`INTERACT_ACROSS_USERS` is a development permission, so `pm grant` works, and it fixes the `isSingleton` crash below. Services Framework and the Play Store then crash-loop on `MANAGE_USERS` (signature|privileged, not grantable), so they stay disabled. Keep Play services enabled: YouTube exits at launch and Peacock opens Play services' App info page when it is disabled. Play services' location module still crashes on `LOCATION_HARDWARE` every few seconds, more often while Apple TV is open. That is expected. Shell cannot disable components inside a non-system app on this build, so do not try `pm disable pkg/component`. Undo with `pm enable` on the two packages and `pm revoke` on the grant.

Do not install Google Play services, the Play Store, or YouTube TV unless the owner explicitly asks, and tell them the result first:

- Play services 26.40.62 (`com.google.android.gms`, versionCode 264062013, arm64-v8a + armeabi-v7a, Android 9+, nodpi) installs and then the process dies. Log: `am_crash` / `SecurityException` at `ActivityManagerService.isSingleton` while attaching the app. Cause: Play services is not a privileged system app, and this is a locked user build with no root. A reboot does not fix it. microG needs signature spoofing this build does not have.
- Play Store 53.5.48 (`com.android.vending`, universal, minSdk 24) installs and cannot sign in, because there is no working Play services and the Settings app has no Accounts section.
- YouTube TV 10.42.0 installs and stays on a loading spinner (`Timeout waiting for provider com.google.android.gms`). It needs an account. Uninstall it if it is present: `adb uninstall com.google.android.apps.youtube.unplugged`.
- YouTube 21.42.243 can play a public video while signed out, and only if `com.google.android.gsf` is installed first and YouTube is installed after it. Otherwise YouTube crashes with `READ_GSERVICES` permission denial. An update install can also fail with `INSTALL_FAILED_INSUFFICIENT_STORAGE`; uninstalling YouTube and installing it fresh granted the permission and fit. YouTube logs `Requires 264210000 but found 264062013` and still plays signed out. It cannot add a Google account. The first open after boot can sit on a white screen for about half a minute while it waits on the dead Play services process.
- Services Framework that matched this tablet: `com.google.android.gsf` 10-6494331, versionCode 29, Android 10+.

If the owner does not need signed-out YouTube, leave Google's packages off. They consume the small data partition and crash in the background.

Sideload a single APK with `adb install -r`. A bundle (`.apkm` / `.xapk`, which are zip files) needs `adb install-multiple -r` of the base APK plus the `arm64-v8a` split. Do not also install unused ABI splits if space is tight; the reference tablet is `arm64-v8a`.

### Paramount+

Installed on the reference tablet on Nov 8, 2026, after the owner asked for it and had the agent fetch the bundle from APKMirror, the same source as the other streaming bundles. Before installing, check that every split is signed by CBS with `apksigner verify --print-certs`. `app/build.sh` points `JAVA_HOME` at `/usr/local/opt/openjdk`; `apksigner` needs the same.

- Package `com.cbs.app` (phone/tablet build), version 17.3.0, versionCode 420000982, minSdk 24, targetSdk 36.
- Bundle: APKMirror "Paramount+ 17.3.0 (universal) (160-640dpi) (Android 7.0+)", `com.cbs.app_17.3.0-420000982_4arch_3dpi_..._apkmirror.com.apkm`, 49,861,190 bytes, SHA-256 `c9a673dce6aafa80f2ea4ddc11b1aa26df8e72c388aec06f0686fb4a40e4cd57`.
- Signer: `CN=CBS Interactive Mobile, O=CBS Interactive`. Certificate SHA-256 `a7520085336e50053de6b8b92e5ab89b6f62b5c22653e5684870586d02fb014e`, SHA-1 `f1c2d55f1759221a516f09766c59bc56b0b1e6e3`. Do not install a Paramount+ APK signed by anything else.
- Splits installed: `base.apk`, `split_config.arm64_v8a.apk`, `split_config.mdpi.apk`, `split_config.xxhdpi.apk`. The tablet is 240 dpi and the bundle has no hdpi split; mdpi plus xxhdpi lets Android pick. Skip the armeabi-v7a, x86, x86_64, and xxxhdpi splits.

```bash
adb install-multiple -r base.apk split_config.arm64_v8a.apk split_config.mdpi.apk split_config.xxhdpi.apk
```

- Do not use `com.cbs.ott`. That is the Android TV build and requires `android.software.leanback`, which the Tread does not report. The arm64-only `com.cbs.app` variant is 480–640 dpi; the universal one above covers 160 dpi too. Skip variants marked Android 12L+.
- Result: launches in about a second to the upsell screen with Get started and Sign in. Sign in shows email/password and Connect provider. Play services binds (clearcut, scheduler, trustedtime) and Paramount+ does not hang on it. No `com.cbs.app` crash. Playback is not confirmed until the owner signs in.
- On-disk cost was about 90 MB of `/data` (661 MB free before, 574 MB after).
- APKMirror's Cloudflare check blocks `curl`, and Cursor's browser does not save downloads to disk. `open "<apkmirror download/?key=... URL>"` in the owner's default browser saved it to `~/Downloads`.

### Making room on `/data`

Lanebreak (`com.onepeloton.odyssey` 2.7.2926, versionCode 2072926, installed by `com.onepeloton.dm.android`) is Peloton's Unity running game. It is a non-system app, not the controller. It took about 219 MB (100 MB APK plus 119 MB native libraries), and uninstalling it took `/data` from 407 MB to 661 MB free. The owner had it removed on Nov 8, 2026, after a copy was pulled to the owner's Mac (outside this repo; APKs are never committed):

```bash
adb pull "$(adb shell pm path com.onepeloton.odyssey | sed 's/^package://' | tr -d '\r')" com.onepeloton.odyssey-2.7.2926.apk
adb uninstall com.onepeloton.odyssey
```

Backup SHA-256: `ce8d512a66e6720d675ee15fac873c8f9c8dfc77e316bb3dbb6b2553bb12ccf3`. Restore with `adb install com.onepeloton.odyssey-2.7.2926.apk`.

Other large `/data` apps on the reference tablet, code only (APK plus native libraries): Firefox ~319 MB (the owner uses it to sign in to HBO Max through AT&T; keep it), YouTube ~270 MB, Play services ~268 MB (Max, Disney+, Peacock, and YouTube need it), Prime Video ~197 MB, Netflix ~122 MB. Peloton's updated system apps (`com.onepeloton.tts` ~121 MB, `com.android.webview` ~128 MB, the disabled `com.peloton.activity` ~105 MB) also sit in `/data`. Do not uninstall those updates; they are system packages and not part of this project.

## Bluetooth

The reference tablet reports `android.hardware.bluetooth` and `android.hardware.bluetooth_le`. It was off until someone opened Settings. Pairing is Settings → Connected devices → Bluetooth. AirPods are not the scale named `YUNMAI-IMX-WS`. AirPods 4 enter pairing with the lid open and a double-tap on the front of the case; AirPods Pro use the flush button on the back of the case. The iPhone will grab them first if its Bluetooth is on. This project does not automate pairing. Do not turn Bluetooth on or off unless the owner asks.

## Known behavior

- TIME keeps counting after the belt stops, until the workout summary is closed, unless Pause is on. Once `firstMoveMs` is set, elapsed is `now - firstMoveMs` even at 0 mph. Do not change that unless the owner asks. Pause and stop stay inactive until that clock is running. While a workout is paused, those buttons stay active: play resumes the clock, and stop still ends the workout. Pause freezes elapsed time and miles and does not change belt speed. That same press sends a media-pause key to a show that is playing. Play resumes the clock only, not the show. I'm sure calls `stopBelt()`, which sets the speed target to 0 and nothing else. Close on the summary (`resetSession`) clears miles and the timer only.
- Labels are one color. Do not turn MPH green or make it larger. The numbers are all 30sp and the columns have equal weight. Order is INCLINE, PACE, TIME, MILES, MPH. The minimized pill shows miles.
- The bar is not focusable, so it does not take key events away from the app underneath. Keep it that way.
- The Watch screensaver starts after 60 seconds without a touch, or when the sleep button is tapped. Holding that button plays the corner burst. While it is up, the metrics bar is hidden and the Watch window is marked with Peloton's own window flag so the tablet hides the bottom row. The mark can travel that area. Clear the flag when the screensaver stops so the row comes back. It does not change the belt. A real corner is both edges in the same frame. The hold is the only shortcut.
- `app/build.sh` creates `app/debug.keystore` on first build (store password `android`, alias `androiddebugkey`). A later install onto the same tablet must be signed with that same key, or `adb install -r` fails with a signature mismatch. Uninstall first if the key is gone. Do not commit the keystore.

## Undo

```bash
app/revert.sh
adb reboot
```

`revert.sh` stops the overlay, sets `lock_service` back to 0, re-enables the three packages, and restores the Home component saved in `app/device-state/home-activity.txt`. If that file is missing, it falls back to `com.peloton.launcher/.LauncherActivity`.

Remove the app only if the owner asks:

```bash
adb uninstall com.treadcover
```

If Google packages were installed and the owner wants them gone:

```bash
adb uninstall com.google.android.gms
adb uninstall com.google.android.gsf
adb uninstall com.android.vending
adb uninstall com.google.android.youtube
adb uninstall com.google.android.apps.youtube.unplugged
```

## If you are about to edit the code

Keep the safety comments in `TreadController` true. The overlay stays read-only except pause, stop, and the minimize chevron. I'm sure may only set belt speed to 0. Close on the summary clears the session numbers and does not change the belt. Knob steps stay 0.1 mph and 0.5% incline. Do not add a software button that starts the belt or raises speed.

After a UI change, build, `adb install -r`, and confirm with a screenshot of Watch and of one streaming app that the bar is visible and the show still receives taps. Then ask the owner to confirm the safety key still stops the belt.
