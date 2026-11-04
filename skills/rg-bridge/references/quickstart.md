# Quickstart

## Requirements

| Item | Value |
|---|---|
| Dart SDK | `^3.12.2` |
| Flutter | `>=1.17.0` (use current stable) |
| Android | Physical device strongly recommended |
| iOS | Volume/power supported via public APIs; Wi-Fi scan/navigation stubs → `NOT_SUPPORTED` |

## Dependency

`rg_bridge` is distributed from the private GitLab pub registry, not pub.dev.
Get the registry URL and secret token from the maintainer.

First, register the registry and paste the secret token when prompted:

```bash
dart pub token add <PRIVATE_PUB_HOSTED_URL>
# Paste the secret token when prompted
```

Then add the dependency:

```yaml
dependencies:
  rg_bridge:
    hosted: <PRIVATE_PUB_HOSTED_URL>
    version: ^0.1.5
```

```bash
flutter pub get
```

## Import

```dart
import 'package:rg_bridge/rg_bridge.dart';
```

The barrel exports `RGBridge`, enums, models, and the widgets
`CheckTouchscreenWidget`, `TakePhotoQrWidget`, and — when the installed version
includes it — `CheckCameraWidget`.
All device controller types are exported by the public barrel, so they can be
used for explicit typing without separate imports:

```dart
import 'package:rg_bridge/rg_bridge.dart';

late final CheckGpsController controller;
```

Controller types are also returned through `sdk.device.*`; do not import their
files directly.

Never import `src/`, with one documented exception: if the host's `rg_bridge`
version does not export `CheckCameraWidget` yet, import
`package:rg_bridge/src/addons/device/check_camera/check_camera_widget.dart`
temporarily and upgrade the package instead of copying the widget.

Every hardware test runs in its own host-built screen. `camera` and
`touchscreen` are the exception: they must use the library widgets above.

## Initialize

```dart
final sdk = RGBridge(
  baseUrl: 'https://api.example.com',
  clientId: 'rg_your_client_id',
);
```

`RGBridge({String baseUrl = '', required String clientId})`.

Recommended service wrapper:

```dart
class RGBridgeService {
  late final RGBridge sdk;

  RGBridgeService() {
    sdk = RGBridge(
      baseUrl: const String.fromEnvironment('RG_BASE_URL'),
      clientId: const String.fromEnvironment('RG_CLIENT_ID'),
    );
  }
}
```

One instance per app. Do not construct `RGBridge` inside every screen.

## Lifecycle

```text
Create RGBridge
  → use device / webview / form / desk / qr
  → handle result / event
  → cleanup listener + controller
  → screen/flow done
```

| Object | Cleanup |
|---|---|
| Device controller | `checkDispose()` |
| Device subscription | `await sub.cancel()` |
| WebView event | `offEvent(type)` / `offAllEvents()` |
| WebView controller | Follows host `WebViewWidget` lifecycle |
| QR photo | `stop()` ends session; `dispose()` closes streams |
| UI timer | Cancel on terminal result / dispose |

## Platform setup

### Android

Permissions are merged from the package manifest: Bluetooth, Bluetooth
scan/connect, location, WiFi state, camera, record audio, vibrate,
`USE_BIOMETRIC`, `USE_FINGERPRINT`. Do not copy them manually.

Biometric also requires the host `MainActivity` to extend
`FlutterFragmentActivity`. `FlutterActivity` makes `authenticate()` fail with
`NotFragmentActivity` even when permissions are present.

```kotlin
import io.flutter.embedding.android.FlutterFragmentActivity

class MainActivity : FlutterFragmentActivity()
```

### iOS — mandatory `Info.plist` keys

Add **all seven** keys to the **host app** `ios/Runner/Info.plist`. There is no
manifest merge on iOS: keys in a package/pod `Info.plist` are ignored, so the
SDK cannot inject them. Add every key even if the app does not expose the test
yet — native plugins initialize at startup and a missing description can crash
the app.

| Key | Feature | Notes |
|---|---|---|
| `NSCameraUsageDescription` | camera, QR take-photo, WebView camera | |
| `NSMicrophoneUsageDescription` | microphone, speech, WebView mic | |
| `NSSpeechRecognitionUsageDescription` | speech (`speech_to_text`) | |
| `NSFaceIDUsageDescription` | biometric (`local_auth`) | Face ID is denied without it |
| `NSLocationWhenInUseUsageDescription` | GPS (`geolocator`) | |
| `NSMotionUsageDescription` | accelerometer / gyroscope (`sensors_plus`) | **Crashes** on motion access if missing |
| `NSBluetoothAlwaysUsageDescription` | bluetooth (`flutter_blue_plus` + native `CBCentralManager`) | **Crashes** on iOS 13+ if missing |

```xml
<key>NSCameraUsageDescription</key>
<string>Camera is used for the camera test and QR photo capture.</string>
<key>NSMicrophoneUsageDescription</key>
<string>Microphone is used for the microphone and speech tests.</string>
<key>NSSpeechRecognitionUsageDescription</key>
<string>Speech recognition is used for the speech-to-text test.</string>
<key>NSFaceIDUsageDescription</key>
<string>Face ID is used for the biometric test.</string>
<key>NSLocationWhenInUseUsageDescription</key>
<string>Location is used for the GPS test.</string>
<key>NSMotionUsageDescription</key>
<string>Motion data is used for the accelerometer and gyroscope tests.</string>
<key>NSBluetoothAlwaysUsageDescription</key>
<string>Bluetooth is used for the Bluetooth adapter test.</string>
```

For iOS 12 and below also add `NSBluetoothPeripheralUsageDescription`. Add
`NSLocationAlwaysAndWhenInUseUsageDescription` only if the app actually needs
background location. WiFi SSID also needs the Access WiFi Information
entitlement, not an `Info.plist` key.

## Suggested structure

```text
lib/
├── sdk/rgbridge_service.dart
├── features/device_check/
│   ├── check_wifi_screen.dart        # one screen per test
│   ├── check_camera_screen.dart      # renders CheckCameraWidget
│   ├── check_touchscreen_screen.dart # renders CheckTouchscreenWidget
│   └── ...
├── features/webview/
├── features/form/
└── main.dart
```

## First-run checklist

1. `flutter pub get`
2. Run on a physical Android or iOS device (iOS volume/power need a real iPhone).
3. `getDeviceInfo()` returns brand/model.
4. One test stream emits a terminal result.
5. Only then wire WebView/Form with a valid `baseUrl` + `clientId`.
