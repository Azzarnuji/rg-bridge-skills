# Transaction (scan diagnostic MyRG)

Requires a valid `baseUrl` + `clientId`.

The transaction surface submits a device scan result and then marks the
inspection request as completed.

```text
sdk.device.buildSummary()
        │
        ▼
sdk.transaction.deviceSubmission.execute(payload, args)
        │
        ▼
DeviceSubmissionModel (requestId, reequesCode, pricing)
        │
        ▼
sdk.transaction.deviceSubmissionAccept.execute(args: requestCode)
        │
        ▼
DeviceSubmissionAcceptModel (status COMPLETED)
```

## Submit the scan result

```dart
final result = await sdk.transaction.deviceSubmission.execute(
  await sdk.device.buildSummary(),
  args: DeviceSubmissionArgs(includePricing: true),
);
```

```dart
Future<DeviceSubmissionModel> execute(
  dynamic payload, {
  required DeviceSubmissionArgs args,
})
```

| Param | Required | Note |
|---|---|---|
| `payload` | yes | `await sdk.device.buildSummary()` (or an equivalent `Map<String, dynamic>`) |
| `args.includePricing` | yes | Sent as `?includePricing=true/false` |

Endpoint: `POST {baseUrl}/api/scan-diagnostic-myrg/webhook?includePricing=<bool>`.
The payload is sent as the JSON body unchanged.

`buildSummary({resetSession = true})` does no network I/O. It clears the
tracked test session after building unless `resetSession: false`.

`DeviceSubmissionModel`:

| Field | Type | Note |
|---|---|---|
| `result` | `bool` | Backend success flag |
| `message` | `String` | Backend message |
| `requestId` | `String` | `request_id` |
| `reequesCode` | `String` | `request_code`; keep it for accept |
| `pricing` | `DeviceSubmissionPricingModel` | Empty when pricing is not requested |

`DeviceSubmissionPricingModel`: `priceMin` (`int`), `priceMax` (`int`),
`message` (`String`). `DeviceSubmissionPricingModel.empty` is used when the
backend omits `pricing` or sends `null`.

## Accept the request

```dart
final accept = await sdk.transaction.deviceSubmissionAccept.execute(
  args: DeviceSubmissionAcceptArgs(requestCode: result.reequesCode),
);
```

```dart
Future<DeviceSubmissionAcceptModel> execute({
  required DeviceSubmissionAcceptArgs args,
})
```

`DeviceSubmissionAcceptArgs { requestCode }` comes from
`DeviceSubmissionModel.reequesCode`.

Endpoint: `PATCH {baseUrl}/api/scan-diagnostic-myrg/<requestCode>` with body
`{"status": "COMPLETED"}`.

`DeviceSubmissionAcceptModel`: `result` (`bool`), `message` (`String`),
`requestId` (`String`), `reequesCode` (`String`), and `status` (`String`,
usually `COMPLETED`).

## Parsing contract

`fromJson` accepts `snake_case` and `camelCase` keys and normalizes types:

- numbers may arrive as `int`, `double`, or `String` (`"1500000.00"` →
  `1500000`);
- booleans accept `true`, `"true"`, `1`, `"1"`, `"yes"`, `"success"`;
- a missing/`null` `pricing` becomes `DeviceSubmissionPricingModel.empty`.

## Error handling

| Condition | Thrown |
|---|---|
| HTTP `422` | `HttpValidationException` (`message`, `fieldMessage`) |
| Other non-2xx | `Exception('HTTP <code>: <body>')` |
| Invalid JSON body | `Exception('HTTP <code>: <body>')` |
| JSON is not an object | `Exception('HTTP <code>: unparseable JSON body')` |
| Network/timeout | Exception from package `http` |

```dart
try {
  final result = await sdk.transaction.deviceSubmission.execute(
    await sdk.device.buildSummary(),
    args: DeviceSubmissionArgs(includePricing: true),
  );
  await sdk.transaction.deviceSubmissionAccept.execute(
    args: DeviceSubmissionAcceptArgs(requestCode: result.reequesCode),
  );
} on HttpValidationException catch (e) {
  // Show e.fieldMessage, not only e.message.
} catch (e) {
  // Network / non-2xx / invalid JSON.
}
```

## Security

- `httpClient` is a private field (`_httpClient`) on every transaction. Do not
  try to access `sdk.transaction.deviceSubmission.httpClient`; it is not part of
  the public contract.
- Only the exported `execute(...)` methods may perform requests. Never add a
  public client getter or import `package:rg_bridge/src/addons/transaction/...`.
- Never log `clientId`, payload PII, or request codes in production.

## Pitfalls

| Symptom | Cause |
|---|---|
| Submit 401/403 | Wrong `clientId` for `baseUrl` |
| HTTP 422 on submit | Payload does not match the backend contract; show `fieldMessage` |
| `pricing` is zero | `includePricing: false` or backend returned no pricing |
| Accept fails | `requestCode` must be the submit response's `reequesCode` |
| Tests cleared unexpectedly | `buildSummary()` defaults to `resetSession: true` |
