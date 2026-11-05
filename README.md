# rg-bridge-skills

Agent skill for integrating the **`rg_bridge`** Flutter SDK (`RGBridge`).

Package: private GitLab pub registry (ask the maintainer for the hosted URL and secret token).

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

It teaches coding agents how to:

- initialize the singleton `RGBridge` instance;
- localize SDK text with Indonesian (default) or English via
  `RGBridge(locale: RgBridgeLocale.en)` / `RgBridgeI18n.configure(...)`;
- run the 16 hardware tests, each from its own screen, and handle
  `DeviceTestResult` / `REQUIRES_USER_ACTION`;
- build the WebView session + JS bridge flow;
- submit forms, create desk inboxes, and receive take-photo files over QR;
- handle permissions and debug platform limitations.
- let the library track hardware results and build a submission payload with
  `sdk.device.buildSummary()` (without performing HTTP requests).

## Install

Requires Node.js 22.20 or later.

```bash
# Interactive: choose agents
npx skills add Azzarnuji/rg-bridge-skills

# Pick the skill explicitly
npx skills add Azzarnuji/rg-bridge-skills --skill rg-bridge

# Non-interactive for OpenCode + Claude Code, project scope
npx skills add Azzarnuji/rg-bridge-skills \
  --skill rg-bridge -a opencode -a claude-code -y

# Global (available in every project)
npx skills add Azzarnuji/rg-bridge-skills --skill rg-bridge -g -y
```

Supports OpenCode, Claude Code, Codex, Cursor, and other agents supported by the
[`skills` CLI](https://github.com/vercel-labs/skills).

## Update / remove

```bash
npx skills check
npx skills update rg-bridge
npx skills remove rg-bridge
```

## What is inside

```text
skills/rg-bridge/
├── SKILL.md                      # concise router + non-negotiable rules
└── references/
    ├── quickstart.md             # install, init, lifecycle, platform config
    ├── device-tests.md           # get* APIs + 16 check* tests
    ├── webview.md                # session, controller, events, HTML
    ├── form-desk-qr.md           # form, desk inbox, QR file transfer
    └── troubleshooting.md        # errors, emulator/iOS limits, checklist
```

The skill uses progressive disclosure: `SKILL.md` stays short and points to the
reference file for the task at hand.

## Scope

The skill assumes the app already depends on the `rg_bridge` Flutter package.
It does not distribute the package itself.

Key rules the skill enforces:

- one host-built screen per hardware test; `camera` and `touchscreen` must use
  `CheckCameraWidget` / `CheckTouchscreenWidget` instead of custom UI;
- public barrel import only (`package:rg_bridge/rg_bridge.dart`), except the
  documented temporary `CheckCameraWidget` import for older package versions;
- one singleton SDK instance per app;
- result persistence is handled by the library; host subscriptions are only
  for UI updates;
- camera/touchscreen can be created through the device widget factories and
  are included automatically in the device session;
- `buildSummary({resetSession: true})` returns the payload with `qrValue: null`
  and resets the session after a successful build;
- subscribe to `getResultStream` before `checkStart()`;
- treat `REQUIRES_USER_ACTION` as intermediate;
- branch on `result.status` / `status.wire`, never on localized `message`;
  status tokens and `testId` / `action.type` are not translated;
- dispose controllers and unregister WebView events;
- never commit real `clientId` values or log tokens/PII.

## License

MIT. See [LICENSE](LICENSE).
