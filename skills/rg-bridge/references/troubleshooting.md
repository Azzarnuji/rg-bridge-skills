# Troubleshooting

## Outdated documented APIs

Some older README snippets do not match the code. Use these signatures:

| Old | Current |
|---|---|
| `sendToWebView(type:, data:)` | `sendToWebView(controller:, event:)` |
| `onEvent(callback)` | `onEvent(WebviewEventType, callback)` |
| `createController(session: session)` | `createController({onConsoleMessage})` |
| `loadHtmlString(html)` | `loadHtmlString(controller:, html:)` |
| 5 events | 6 events (includes `formClosed`) |

## Device tests

| Symptom | Cause | Fix |
|---|---|---|
| No result ever | Listener added after `checkStart` | Listen first, then start |
| UI finishes too early | `REQUIRES_USER_ACTION` treated as final | Check `status.isTerminal` |
| `FAILED - timeout` | Missing hardware / emulator | Use a physical device or notifiers |
| `NOT_SUPPORTED` | Permission permanently denied / platform stub | Check permissions / iOS limits |
| Second test never runs | Old controller not disposed | `checkDispose()` before creating |
| UI does not update | No `setState` in listener | Update state on result/progress |
| Timer keeps running | Not stopped on terminal | Stop UI timer on terminal result |
| `Stream has already been listened to` | Multiple listeners | Keep one stored subscription |
| Camera test run without preview/countdown | Controller-only run | Use `CheckCameraWidget` |
| Touchscreen result from a custom grid | Re-implemented UI | Use `CheckTouchscreenWidget` |
| Two tests share one controller | Controller reused across screens | One screen, one controller |

## WebView

| Symptom | Cause | Fix |
|---|---|---|
| White screen | Empty `baseUrl` / invalid URL | Set `baseUrl`, verify `webViewUrl` |
| No events | Listener registered late | Register before loading |
| Events lost between screens | `offAllEvents()` too early | Call it in `dispose` |
| `sendToWebView` inert | JS not listening | Ensure `rg_browser.js` loaded |
| JS errors hidden | Console not forwarded | Pass `onConsoleMessage` |
| Blank `loadHtmlString` | Missing `rg_browser.js` | Load from the correct source |
| Camera permission denied | OS permission not granted | Re-request / send to Settings |

## Form / Desk / QR

| Symptom | Cause | Fix |
|---|---|---|
| Submit 401/403 | Wrong `clientId` | Match the environment |
| Submit JSON error | Non-JSON body | Validate `values`/`meta` |
| `sessionToken` null | Forgot `initFormSubmission` | Init first |
| Inbox URL expired | TTL elapsed | Create a new inbox |
| QR missing | `buildQr` before `show` | Call after `show` |
| No files | `maxFiles` reached / pusher failure | Check status and pusher log |
| Stream error after close | Object used after `dispose` | Cancel subscriptions first |

## HTTP errors

`HttpClientAddon` throws `Exception('HTTP <status>: <body>')`.

| Code | Likely cause |
|---|---|
| 401 / 403 | Invalid `clientId` |
| 404 | `baseUrl` + path mismatch (trailing slash) |
| 422 | Backend rejected form/desk payload |
| 5xx | Backend issue; surface retry |

`baseUrl` is concatenated with the path. Use `https://api.example.com`, not
`https://api.example.com/`.

## Build and platform

| Symptom | Fix |
|---|---|
| `minSdkVersion` error | Raise host `minSdk` |
| iOS never asks for permission | Add the `Info.plist` keys |
| `MissingPluginException` | `flutter clean` + rebuild; non-fatal in the SDK |
| Button test fails on iOS | Native button interception is a stub on iOS |
| Emulator lacks sensors | Use simulation notifiers |

## Debug checklist

1. Is there exactly one `RGBridge` instance?
2. Are `baseUrl` / `clientId` correct for networked surfaces?
3. Is the result stream subscribed before start?
4. Is `REQUIRES_USER_ACTION` filtered out as terminal?
5. Are `checkDispose()` / `offAllEvents()` called?
6. Is the test running on a physical device?
7. Are OS permissions granted?
8. Are session tokens unexpired?
9. Does the log contain `HTTP <code>`?
10. Does the package version match the docs?

## Version sources

- `pubspec.yaml` → package version (source of truth)
- `CHANGELOG.md` → release history
- Older docs may mention different versions

Always trust `pubspec.yaml` + `CHANGELOG.md`.

## Escalation

If the issue is reproducible on a physical device with permissions granted:

1. Capture the exact `DeviceTestResult.toJson()` output.
2. Capture WebView console messages.
3. Capture the HTTP status/body for networked calls (redact secrets).
4. Attach the Flutter and package versions.
