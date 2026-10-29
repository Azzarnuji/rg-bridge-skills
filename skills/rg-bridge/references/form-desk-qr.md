# Form submission, desk inbox, and QR take-photo

All require a valid `baseUrl` + `clientId`.

## Form submission

```dart
final session = await sdk.formSubmission.initFormSubmission();
```

`InitFormSubmissionModel { sessionToken }`. Re-init if the token expires.

```dart
final now = DateTime.now().toUtc();

final form = SubmitFormModel(
  morphableId: session.sessionToken,
  morphableType: 'session',
  morphableIncomingFrom: 'sdk-rg-bridge',
  formPoolId: 'gadai_online_hp_manual',
  values: {'name': 'John Doe', 'phone': '08123456789'},
  need: {'appraisal': false},
  meta: {'source': 'example_app'},
  status: 'new',
  id: '',
  updatedAt: now,
  createdAt: now,
);

final result = await sdk.formSubmission.submitForm(
  sessionToken: session.sessionToken,
  form: form,
);
```

`SubmitFormModel` fields: `morphableId`, `morphableType`,
`morphableIncomingFrom`, `formPoolId`, `values`, `need?`, `meta?`, `status`,
`id`, `conversation?` (`ConversationModel { id, url }`), `updatedAt`,
`createdAt`.

The SDK posts `session_id` plus `form.toJson()` and returns the backend
`SubmitFormModel` (including `id` and optional `conversation`).

Error handling:

| Source | Handling |
|---|---|
| Network/timeout | `try/catch`, show retry |
| HTTP non-2xx | SDK throws `Exception('HTTP <code>: <body>')` |
| Invalid JSON | `jsonDecode` failure inside `try/catch` |
| Expired session | Re-init then submit |

## Desk inbox

```dart
final inbox = await sdk.desk.createCustomerInbox(
  name: 'Ahmad Demo',
  phone: '085881234567',
  email: 'ahmad.demo@example.com',
  memberCode: '0001',
  ttlHours: 24,
);
```

| Param | Required | Note |
|---|---|---|
| `name` | yes | |
| `phone` | yes | |
| `email` | yes | |
| `memberCode` | yes | |
| `ttlHours` | no | default `24`, must be > 0 |

`CustomerInboxModel`: `identityHash`, `token`, `exp`, `url`, `expiresAt`,
`customer` (`DeskCustomerModel`), `disputed`.

`DeskCustomerModel`: `id`, `name`, `phone`, `email`, `memberCode`,
`identityHash`, `isVerifiedMemberCode`.

Open the signed URL in a WebView:

```dart
final controller = sdk.webview.createController();
await sdk.webview.openWebView(controller: controller, url: inbox.url);
// WebViewWidget(controller: controller)
```

Do not rewrite the signed URL; treat `url` and `token` as sensitive.

## QR take-photo

```dart
final info = await sdk.qr.photo.show(maxFiles: 5);
```

`QRUseCaseModelType { url, transferSessionId?, expiresAt? }`.
`show()` initializes the backend session (`type: 'take-photo'`,
`channel: 'browser'`, `meta.max_files`) and connects the host receiver.

Display the QR:

```dart
sdk.qr.photo.buildQr(size: 220); // only after show()
final url = sdk.qr.photo.qrUrl;
```

The QR contains the URL only, never `host_token`.

Streams:

| Stream | Type |
|---|---|
| `onFileReceived` | `ReceivedFile` |
| `onStatus` | `FileTransferHostStatus` |
| `onPusherEvent` | `String` |
| `onSessionMessage` | `String` |

`ReceivedFile`: `transferId`, `sequenceIndex`, `filename`, `mime`, `bytes`
(`Uint8List`), `sha256`.

`FileTransferHostStatus`: `idle`, `connecting`, `connected`, `receiving`,
`completed`, `failed`, `cancelled`, `expired`, `sessionClosed`, `disposed`.

`sessionClosed` means the web closed the session (for example, `maxFiles`
reached); `qrUrl` is cleared.

Example:

```dart
class TakePhotoPage extends StatefulWidget {
  const TakePhotoPage({super.key, required this.sdk});

  final RGBridge sdk;

  @override
  State<TakePhotoPage> createState() => _TakePhotoPageState();
}

class _TakePhotoPageState extends State<TakePhotoPage> {
  final List<ReceivedFile> _files = [];
  final List<StreamSubscription<dynamic>> _subs = [];

  QRUseCaseModelType? _session;
  FileTransferHostStatus? _status;
  String? _error;

  RGBridge get _sdk => widget.sdk;

  @override
  void initState() {
    super.initState();
    _subs.add(_sdk.qr.photo.onFileReceived.listen((file) {
      if (!mounted) return;
      setState(() => _files.insert(0, file));
    }));
    _subs.add(_sdk.qr.photo.onStatus.listen((status) {
      if (!mounted) return;
      setState(() {
        _status = status;
        if (status == FileTransferHostStatus.sessionClosed) _session = null;
      });
    }));
  }

  @override
  void dispose() {
    for (final s in _subs) {
      s.cancel();
    }
    unawaited(_sdk.qr.photo.stop());
    super.dispose();
  }

  Future<void> _start() async {
    setState(() {
      _error = null;
      _files.clear();
    });
    try {
      final info = await _sdk.qr.photo.show(maxFiles: 5);
      if (!mounted) return;
      setState(() => _session = info);
    } catch (e) {
      if (mounted) setState(() => _error = '$e');
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Take Photo')),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          FilledButton(onPressed: _start, child: const Text('Show QR')),
          if (_error != null) Text('ERROR: $_error'),
          if (_status != null) Text('status: ${_status!.name}'),
          if (_session != null) ...[
            Center(child: _sdk.qr.photo.buildQr(size: 220)),
            SelectableText(_session!.url),
          ],
          for (final f in _files)
            ListTile(
              leading: Image.memory(f.bytes, width: 48, height: 48),
              title: Text(f.filename),
              subtitle: Text('${f.mime} · ${f.bytes.length} B'),
            ),
        ],
      ),
    );
  }
}
```

### stop vs dispose

| Method | Effect | When |
|---|---|---|
| `stop()` | Disconnect receiver, clear `qrUrl`, emit `disposed`, keep streams open | Session finished / screen closed / start a new session |
| `dispose()` | `stop()` + close all streams | SDK/use-case discarded permanently |

`show()` tears down the previous receiver, so calling it again is safe.

## Pitfalls

| Symptom | Cause |
|---|---|
| Submit 401/403 | Wrong `clientId` for `baseUrl` |
| `morphable_id` mismatch | Must use the `initFormSubmission` token |
| Inbox URL expired | `ttlHours` elapsed |
| QR blank | `buildQr` before `show()` |
| No files received | QR not scanned / pusher failed / `maxFiles` reached |
| Stream error after close | Using the photo use-case after `dispose()` |
