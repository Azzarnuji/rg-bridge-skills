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
DeviceSubmissionModel (requestId, requestCode, pricing, hdc)
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
| `requestCode` | `String` | `request_code`; keep it for accept |
| `pricing` | `DeviceSubmissionPricingModel` | Empty when pricing is not requested |
| `hdc` | `DeviceSubmissionHdcModel` | HDC info; empty when backend omits `hdc` |

`DeviceSubmissionPricingModel`: `priceMin` (`int`), `priceMax` (`int`),
`message` (`String`). `DeviceSubmissionPricingModel.empty` is used when the
backend omits `pricing` or sends `null`.

`DeviceSubmissionHdcModel`:

| Field | Type | Wire key | Note |
|---|---|---|---|
| `value` | `bool` | `value` | `true` = device **is** HDC; `false` = device is **not** HDC |
| `reason` | `String` | `reason` | Missing/`null` → `''` |
| `chipsetApi` | `String` | `chipset_api` / `chipsetApi` | Missing/`null` → `''` |
| `chipsetDevice` | `String` | `chipset_device` / `chipsetDevice` | Missing/`null` → `''` |

`hdc.value` is the HDC verdict: `true` means the device is an HDC device,
`false` means it is not. Branch on `hdc.value`; never infer HDC status from
`reason` or the chipset fields.

`DeviceSubmissionHdcModel.empty` is used when the top-level `hdc` is absent or
`null`. A present-but-partial `hdc` object is safe: each missing/`null` field
falls back to its default (`false` / `''`), so `empty.value == false` means
"not HDC / no HDC information".

## Serialization

Response models expose `toJson()` in the backend's snake_case shape, so
`Model.fromJson(model.toJson())` round-trips:

- `DeviceSubmissionPricingModel.toJson()` →
  `{price_min, price_max, message}`
- `DeviceSubmissionHdcModel.toJson()` →
  `{value, reason, chipset_api, chipset_device}`
- `DeviceSubmissionModel.toJson()` →
  `{result, message, request_id, request_code, pricing, hdc}`
- `DeviceSubmissionAcceptModel.toJson()` →
  `{result, message, request_id, request_code, status}`

`DeviceSubmissionArgs` / `DeviceSubmissionAcceptArgs` are request options
(`includePricing` is a query param, `requestCode` is a path param) — they are
not wire models and have no `toJson()`.

## Accept the request

```dart
final accept = await sdk.transaction.deviceSubmissionAccept.execute(
  args: DeviceSubmissionAcceptArgs(requestCode: result.requestCode),
);
```

```dart
Future<DeviceSubmissionAcceptModel> execute({
  required DeviceSubmissionAcceptArgs args,
})
```

`DeviceSubmissionAcceptArgs { requestCode }` comes from
`DeviceSubmissionModel.requestCode`.

Endpoint: `PATCH {baseUrl}/api/scan-diagnostic-myrg/<requestCode>` with body
`{"status": "COMPLETED"}`.

`DeviceSubmissionAcceptModel`: `result` (`bool`), `message` (`String`),
`requestId` (`String`), `requestCode` (`String`), and `status` (`String`,
usually `COMPLETED`).

## Parsing contract

`fromJson` accepts `snake_case` and `camelCase` keys and normalizes types:

- numbers may arrive as `int`, `double`, or `String` (`"1500000.00"` →
  `1500000`);
- booleans accept `true`, `"true"`, `1`, `"1"`, `"yes"`, `"success"`;
- a missing/`null` `pricing` becomes `DeviceSubmissionPricingModel.empty`;
- a missing/`null` top-level `hdc` becomes `DeviceSubmissionHdcModel.empty`;
- inside `hdc`, `value` is boolean-normalized and missing/`null`
  `reason`/`chipsetApi`/`chipsetDevice` become `''`; HDC also accepts the
  camelCase `chipsetApi`/`chipsetDevice` keys.

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
    args: DeviceSubmissionAcceptArgs(requestCode: result.requestCode),
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
| Accept fails | `requestCode` must be the submit response's `requestCode` |
| Tests cleared unexpectedly | `buildSummary()` defaults to `resetSession: true` |
