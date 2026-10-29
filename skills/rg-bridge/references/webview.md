# WebView bridge

Requires a valid `baseUrl` + `clientId`.

## Flow

```text
initSessionWebView
  → register onEvent
  → createController
  → openWebView / loadHtmlString
  → receive events
  → offAllEvents + controller lifecycle
```

## Session

```dart
final session = await sdk.webview.initSessionWebView(
  type: 'pengecekan-laptop',
  firstName: 'John',
  lastName: 'Doe',
  memberCode: '123456',
  meta: {'conversation_id': 'conv-123'},
);
```

`type` accepts a free-form `String` or `WebviewType`:

| Enum | Wire |
|---|---|
| `WebviewType.workflow` | `workflow` |
| `WebviewType.conversationWebview` | `conversation-webview` |
| `WebviewType.conversationInboxWebview` | `conversation-inbox-webview` |

Returns `InitSessionWebViewModel { sessionToken, webViewUrl }`.

Re-fetch URL:

```dart
final again = await sdk.webview.getSessionWebView(
  sessionToken: session.sessionToken,
);
```

## Controller

```dart
final controller = sdk.webview.createController(
  onConsoleMessage: (msg) {
    debugPrint('[WebView:${msg.level.name}] ${msg.message}');
  },
);
```

`createController` takes **no session**. It configures:

- `JavaScriptMode.unrestricted`
- JS channel `RGMobileChannel`
- optional console forwarding
- WebView camera/microphone permission handling
- Android: media playback without an extra gesture

`controller` is a `webview_flutter` `WebViewController`; import
`package:webview_flutter/webview_flutter.dart` for the type and
`WebViewWidget`.

## Load content

```dart
await sdk.webview.openWebView(
  controller: controller,
  url: session.webViewUrl,
);
```

```dart
await sdk.webview.loadHtmlString(
  controller: controller,
  html: html,
);

await sdk.webview.loadHtmlStringWithInlineJs(
  controller: controller,
  html: html,
  jsInline: rgBrowserJs,
);
```

`loadHtmlString` sets a base URL so `<script src="rg_browser.js">` resolves.
Only load trusted content.

Render:

```dart
WebViewWidget(controller: controller)
```

## Events (JS → Dart)

Register per type, before loading:

```dart
for (final type in WebviewEventType.values) {
  sdk.webview.onEvent(type, _handle);
}

void _handle(WebViewEvent event) {
  debugPrint('${event.type.name}: ${event.data}');
}
```

Remove:

```dart
sdk.webview.offEvent(WebviewEventType.formSubmitted);
sdk.webview.offAllEvents();
```

| Event | Meaning |
|---|---|
| `formSubmitted` | Form submitted |
| `formCancelled` | Form cancelled |
| `formError` | Form error |
| `formSuccess` | Form success |
| `formClosed` | Form/flow closed |
| `imageCaptured` | Image captured |

Payload keys come from the web page; never assume them:

```dart
final name = event.data['name'] as String? ?? '-';
final message = event.data['message'] as String? ??
    event.data['code'] as String? ??
    '-';
```

## Dart → JS

```dart
await sdk.webview.sendToWebView(
  controller: controller,
  event: const WebViewEvent(
    type: WebviewEventType.formSuccess,
    data: {'id': 'abc'},
  ),
);
```

This dispatches a `CustomEvent` on `window`.

Channels (`WebViewBridgeChannels`):

| Direction | Channel |
|---|---|
| JS → Flutter | `RGMobileChannel` (`WebViewBridgeChannels.mobile`) |
| Flutter → JS | `RGBrowserEntryPointForMobile` (`WebViewBridgeChannels.browserEntry`) |

Must stay in sync with `rg_browser.js`.

## Full example

```dart
class WebViewScreen extends StatefulWidget {
  const WebViewScreen({super.key, required this.sdk});

  final RGBridge sdk;

  @override
  State<WebViewScreen> createState() => _WebViewScreenState();
}

class _WebViewScreenState extends State<WebViewScreen> {
  RGBridge get _sdk => widget.sdk;

  WebViewController? _controller;
  String? _token;
  String? _error;
  bool _loading = true;

  @override
  void dispose() {
    _sdk.webview.offAllEvents();
    super.dispose();
  }

  Future<void> _start() async {
    setState(() {
      _loading = true;
      _error = null;
    });
    try {
      final session = await _sdk.webview.initSessionWebView(
        type: 'pengecekan-laptop',
        firstName: 'John',
        lastName: 'Doe',
        memberCode: '123456',
      );
      _token = session.sessionToken;

      for (final type in WebviewEventType.values) {
        _sdk.webview.onEvent(type, _onEvent);
      }

      final controller = _sdk.webview.createController(
        onConsoleMessage: (m) => debugPrint('[JS] ${m.message}'),
      );
      await _sdk.webview.openWebView(
        controller: controller,
        url: session.webViewUrl,
      );

      if (!mounted) return;
      setState(() {
        _controller = controller;
        _loading = false;
      });
    } catch (e) {
      if (!mounted) return;
      setState(() {
        _error = '$e';
        _loading = false;
      });
    }
  }

  void _onEvent(WebViewEvent event) {
    if (!mounted) return;
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text('${event.type.name}: ${event.data}')),
    );
  }

  @override
  Widget build(BuildContext context) {
    if (_loading) {
      return const Scaffold(body: Center(child: CircularProgressIndicator()));
    }
    if (_error != null) {
      return Scaffold(
        body: Center(
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Text(_error!, textAlign: TextAlign.center),
              FilledButton(onPressed: _start, child: const Text('Retry')),
            ],
          ),
        ),
      );
    }
    return Scaffold(
      body: SafeArea(child: WebViewWidget(controller: _controller!)),
    );
  }
}
```

## Reload

```dart
Future<void> _reload() async {
  final controller = _controller;
  final token = _token;
  if (controller == null || token == null) return;

  final session = await _sdk.webview.getSessionWebView(
    sessionToken: token,
  );
  await _sdk.webview.openWebView(
    controller: controller,
    url: session.webViewUrl,
  );
}
```

## Permissions and security

- The WebView asks the OS for camera/microphone permission before granting.
- Only open backend-issued URLs. Never open arbitrary user input.
- Never log session tokens, form values, or PII.
- Use HTTPS in production.
- Treat any loaded HTML/JS as inside the app trust boundary.

## Pitfalls

| Symptom | Cause |
|---|---|
| Blank page | Empty/invalid `baseUrl` or URL |
| No events | `onEvent` registered after page calls the bridge |
| Events stop between screens | `offAllEvents()` called too early |
| `sendToWebView` has no effect | JS not listening for the custom event |
| Blank after `loadHtmlString` | `rg_browser.js` not found / base URL wrong |
| Permission denied | OS permission missing/denied |
| Compile error on old docs | Legacy signatures (`sendToWebView(type:, data:)`, `onEvent(cb)`, `createController(session:)`) are outdated |
