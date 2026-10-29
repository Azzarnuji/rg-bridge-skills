# rg-bridge-skills

Agent skill for integrating the **`rg_bridge`** Flutter SDK (`RGBridge`).

It teaches coding agents how to:

- initialize one `RGBridge` instance and manage its lifecycle;
- run the 16 hardware tests and handle `DeviceTestResult` / `REQUIRES_USER_ACTION`;
- build the WebView session + JS bridge flow;
- submit forms, create desk inboxes, and receive take-photo files over QR;
- handle permissions and debug platform limitations.

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

- public barrel import only (`package:rg_bridge/rg_bridge.dart`);
- one SDK instance per app;
- subscribe to `getResultStream` before `checkStart()`;
- treat `REQUIRES_USER_ACTION` as intermediate;
- dispose controllers and unregister WebView events;
- never commit real `clientId` values or log tokens/PII.

## License

MIT. See [LICENSE](LICENSE).
