# Device info and hardware tests

Everything here is under `sdk.device` and needs no `baseUrl`.

## Naming

| Form | Meaning |
|---|---|
| `get*` | Retrieval (Future) |
| `check*()` | Factory returning a controller |
| `controller.getX` | State getter |
| `controller.checkX()` | Action / notifier |
| `on*` | Stream |

## Device information

```dart
final info = await sdk.device.getDeviceInfo();
final stable = await sdk.device.getDeviceInfo(includeStableId: true);
```

`CheckDeviceInfo`: `brand`, `model`, `osName`, `osVersion`, `sdkInt`,
`isPhysicalDevice`, `deviceId` (random UUID per install), `stableId`
(`androidId` / `identifierForVendor`, only when requested).

`getDeviceInfo` never throws; plugin failures fall back to `Platform.*`.

```dart
final battery = await sdk.device.getBatteryInfo(); // null if unsupported
final memory = await sdk.device.getMemoryInfo();
final storage = await sdk.device.getStorageInfo();
final summary = await sdk.device.getHardwareSummary();
final ids = sdk.device.getSupportedTestIds();
```

- `CheckBatteryInfo`: `percentage`, `status`, `health`, `temperatureC`,
  `plugged`, `voltage`, `technology`, `powerSaveMode`.
- `CheckMemoryInfo`: `totalRam`, `availableRam`, `usedRam`, `usedPercent`.
- `CheckStorageInfo`: `totalStorage`, `freeStorage`, `physicalStorage`,
  `usedStorage`, `usedPercent`.
- Real battery/memory/storage data is Android-only; elsewhere `null`.
- `getHardwareSummary()` returns `device`, `battery`, `memory`, `storage`,
  `tests`, `platform`.
- `getSupportedTestIds()` is the static capability list, not hardware detection.

## Status contract

```dart
enum DeviceTestStatus { passed, failed, skipped, notSupported, requiresUserAction }
```

Wire: `PASS`, `FAILED`, `SKIPPED`, `NOT_SUPPORTED`, `REQUIRES_USER_ACTION`.
`status.isTerminal == false` only for `requiresUserAction`.

```dart
class DeviceTestResult {
  final String test;
  final DeviceTestStatus status;
  final String? message;
  final DeviceTestAction? action; // only for requiresUserAction
  final Map<String, dynamic>? data; // null when empty
}
```

## Canonical recipe

```dart
final controller = sdk.device.checkGps();

final sub = controller.getResultStream.listen((result) {
  if (!result.status.isTerminal) {
    setState(() => _intermediate = result);
    return;
  }
  setState(() => _result = result);
});

await controller.checkStart();
// dispose: await sub.cancel(); controller.checkDispose();
```

Never `await` a result from `checkStart`. Results only arrive on the stream.

## Test matrix

| Test | Factory options (defaults) | Timeout | Getters | Notifiers |
|---|---|---|---|---|
| Touchscreen | `columns: 15, rows: 26` | 60 s | `getCoverage`, `getProgressListenable`, `getSecondsLeftListenable` | `checkMark`, `checkFinishPassed`, `checkFinishTimeout` |
| Accelerometer | `sampleStream?` | 15 s | `getMovedAxes`, `getLastSample` | `checkNotifySample`, `checkMarkNotSupported` |
| Gyroscope | `sampleStream?` | 15 s | `getMovedAxes`, `getLastSample` | `checkNotifySample`, `checkMarkNotSupported` |
| Biometric | `mode`, `canCheck?`, `isSupported?`, `authenticate?` | 30 s | `getModeName` | — |
| Bluetooth | — | 20 s | `getLastState` | `checkNotifyAdapterState`, `checkRequestEnable` |
| WiFi | — | 20 s | `getNetworks` | `checkNotifyScan`, `checkRequestEnable` |
| GPS | `requestPosition?` | 30 s | — | `checkNotifyPosition` |
| Camera | `checkIsFront`, `initialize?`, `captureFileSize?` | init 15 s | — | `checkNotifyCaptured`, `checkNotifyFlashAnswer` |
| Speaker | `random?` | 20 s | `getTargetCount` | `checkVerifyGuess` |
| Vibration | `random?` | 20 s | `getTargetCount` | `checkVerifyGuess` |
| Speech | `random?`, `startListening?`, `stopListening?` | 30 s | `getTargetNumber`, `getTranscript` | `checkNotifyTranscript`, `checkFinishFromTranscript` |
| Microphone | `listenDuration`, `onCheckTranscript?`, `startListening?`, `stopListening?` | 30 s | `getTargetNumber`, `getTranscript` | `checkNotifyTranscript`, `checkFinishFromTranscript` |
| Flashlight | `isAvailable?`, `turnOn?`, `turnOff?` | 30 s | `getIsTorchOn` | `checkVerifySeen` |
| Navigation button | `backTimeout`, `stepTimeout` | 15 s / 30 s | `getStep`, `getPassedSteps` | `checkNotifyBack`, `checkNotifyHome`, `checkNotifyRecent` |
| Power | `screenStreamOverride?` | 30 s | — | `checkNotifyScreenOff`, `checkNotifyScreenOn`, `checkNotifyScreenEvent` |
| Volume button | `checkKeyCode: 24`, `buttonStreamOverride?` | 30 s | — | `checkNotifyKey`, `checkNotifyButtonEvent` |

All controllers expose `getResultStream`, `getIsFinished`, `checkSkip()`,
`checkDispose()`. `checkStart()` returns `Future<void>`, `Future<int>` (target),
or `void` depending on the test (see notes below).

## Per-test notes

### Motion (accelerometer, gyroscope)

`checkStart()` is `void` and auto-subscribes `sensors_plus`. PASS when X, Y, and
Z all move beyond threshold (accel `2.0`, gyro `0.5`).

```dart
final c = sdk.device.checkAccelerometer();
c.getResultStream.listen(onResult);
c.checkStart();
final sample = c.getLastSample; // [x, y, z] or null
final moved = c.getMovedAxes;   // {'x', 'z'}
```

### Guess (speaker, vibration)

`checkStart()` returns `Future<int>` target (1–4). After the stimulus it emits
`REQUIRES_USER_ACTION` with `action.type` = `guess_speaker` /
`guess_vibration`. Finish with `checkVerifyGuess(n)`.

### Speech / microphone

`checkStart()` returns `Future<int>` (3-digit target). Live transcript via
`getTranscript` or `onCheckTranscript`. Evaluate early with
`checkFinishFromTranscript()`.

### Camera

`await checkStart()` initializes, captures, measures file size, deletes the temp
file. PASS if capture > 0 bytes. Rear camera can flag flash with
`checkNotifyFlashAnswer(bool)`.

### Flashlight

`await checkStart()` turns the torch on and emits `REQUIRES_USER_ACTION`
(`confirm_flashlight`). Finish with `checkVerifySeen(bool)`. The library always
turns the torch off.

### WiFi / Bluetooth

`await checkStart()`. WiFi PASS when enabled (SSID list in data / `getNetworks`,
max 8). Bluetooth PASS when the adapter is on (`getLastState`).
`await checkRequestEnable()` opens system settings.

### GPS

`await checkStart()`. Default provider uses `geolocator` (permission + service +
high-accuracy fix, 25 s internal limit). PASS when accuracy ≤ 50 m.
Custom provider via `requestPosition`; manual result via
`checkNotifyPosition(lat, lng, accuracy)`.

Note: `CheckGpsPosition` is not exported from the barrel. Prefer
`checkNotifyPosition` in app code, or define the callback inside the package
domain.

### Biometric

`await checkStart()`. PASS on successful system authentication; no hardware →
`NOT_SUPPORTED`. iOS requires `NSFaceIDUsageDescription`.

### Volume button

`checkStart()` is `void`. Native plugin intercepts the keycode on Android.
Simulate with `checkNotifyKey(24|25)` or
`checkNotifyButtonEvent(CheckButtonEvent.fromJson({...}))`.

`CheckButtonEvent` constants: `checkKeyVolumeUp = 24`,
`checkKeyVolumeDown = 25`, names `VOLUME_UP`, `VOLUME_DOWN`, `POWER`, `HOME`,
`BACK`, `CAMERA`.

### Power button

`checkStart()` is `void`. PASS on detected off → on. Simulate with
`checkNotifyScreenOff()` then `checkNotifyScreenOn()`.

### Navigation button

`checkStart()` is `void`. Sequence Back → Home → Recent. Host must wire:

- `PopScope(canPop: false)` to detect Back;
- `WidgetsBindingObserver` lifecycle to detect Home/Recent;
- then call `checkNotifyBack/Home/Recent`.

```dart
@override
void didChangeAppLifecycleState(AppLifecycleState state) {
  final c = _controller;
  if (c == null || c.getIsFinished) return;

  if (state == AppLifecycleState.paused) _sawPaused = true;
  if (state == AppLifecycleState.inactive) _sawInactive = true;

  if (state == AppLifecycleState.resumed) {
    if (c.getStep == 1 && _sawPaused) c.checkNotifyHome();
    if (c.getStep == 2 && (_sawInactive || _sawPaused)) c.checkNotifyRecent();
    _sawPaused = false;
    _sawInactive = false;
  }
}
```

### Touchscreen widget

```dart
final result = await Navigator.of(context).push<DeviceTestResult>(
  MaterialPageRoute(
    builder: (_) => CheckTouchscreenWidget(
      columns: 15,
      rows: 26,
      timeout: const Duration(seconds: 60),
      onCheckProgress: (coverage) {},
      onCheckFinished: (r) => Navigator.of(context).pop(r),
    ),
  ),
);
```

The widget owns its controller (starts in `initState`, disposes automatically).

## Simulation (emulator / desktop)

| Test | Simulation |
|---|---|
| Accelerometer | `checkNotifySample(0,0,0)` → `(3,0,0)` → `(3,3,0)` → `(3,3,3)` |
| Gyroscope | same with values > 0.5 |
| GPS | `checkNotifyPosition(-6.2088, 106.8456, 12)` |
| WiFi | `checkNotifyScan(['A','B'], enabled: true)` |
| Bluetooth | `checkNotifyAdapterState('on')` |
| Speaker / Vibration | `checkVerifyGuess(n)` |
| Speech / Mic | `checkNotifyTranscript('angka 456')` |
| Camera | `checkNotifyCaptured(1024)` |
| Flashlight | `checkVerifySeen(true)` |
| Power | `checkNotifyScreenOff()` + `checkNotifyScreenOn()` |
| Volume | `checkNotifyKey(24)` |
| Navigation | `checkNotifyBack/Home/Recent()` |
| Missing sensor | `checkMarkNotSupported('no sensor')` |

## Pitfalls

- Subscribing after `checkStart()` can miss the first result.
- Treating `REQUIRES_USER_ACTION` as final stops the flow early.
- Reusing a finished controller: create a new one or `checkDispose()` first.
- Forgetting `checkDispose()` leaves timers/streams alive.
- Listening twice to a broadcast stream is fine only with a stored
  subscription; avoid duplicate listeners per widget rebuild.
