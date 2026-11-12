# Managed model context window selection

Eligible managed models and presets can expose explicit context-window choices up to 1M. The selection is an optional session budget; the server remains the source of truth for the default and for the allowed choices.

## Invariants

- Existing `contextWindow` retains its existing type, value, and default semantics. No bulk conversion to 400K or 1M.
- Optional `contextWindowOptions` declares allowed explicit budgets. It never selects a budget.
- Absence of user intent follows the current model/mode default, including after catalog refresh or restart.
- Only a deliberate window-menu action writes preference. Opening a selector, choosing a model, loading a session, or observing usage never converts a default to intent.
- Preferences are local, scoped by server origin and the existing personal/enterprise account owner key, then by concrete model or virtual mode.
- Existing historical default sessions do not inherit a preference newly set in another session. New manual chats may restore the user's previously explicit choice for that model/mode.
- Reset removes intent. Automatic IM, scheduled execution and non-fork subagents do not read manual-chat preference.

## Additive catalog contract

`GET /api/models/available`, existing preset catalog endpoints, and preset resolve responses may include:

```json
{
  "modelId": "deepseek-flash",
  "contextWindow": 400000,
  "contextWindowOptions": [400000, 1000000]
}
```

The server sends the new field only to clients declaring `context-window-selection-v1` in the existing `X-LobsterAI-Client-Capabilities` header. Missing/null/empty options expose no selector. Old clients retain the original catalog/default and inference behavior. A new client connected to a server without the field uses the original default path.

Affected existing endpoints: `GET /api/models/available`, `GET /api/model-presets/available`, and `POST /api/model-presets/{presetId}/resolve`. The resolve body remains unchanged. Existing JWT Bearer authentication, personal/enterprise account scope and model access checks continue to apply; no new permission or authentication path is introduced.

No inference request body, quota, billing, input hard limit, required parameter, auth, endpoint, or existing error contract changes. This remains a client budget choice, rather than a server 400K input rejection rule.

## Admin and persistence

- Add nullable `context_window_options` to `model_pricing` and `model_presets` using `sql/V138__model_context_window_options.sql` before deploying the new server.
- Existing admin update payloads omitting `contextWindowOptions` preserve its stored value. Explicit null clears it. Options are unique integers between 400,000 and 1,000,000, maximum eight entries.
- Verify all model routes and fallbacks support a budget before exposing it.
- Preset choices intersect configured mode policy with every eligible candidate's choices; resolve rechecks the bound real model without rerouting that session.
- Desktop SQLite adds nullable `context_window_override` and `context_window_owner_key`. No historical defaults are migrated into explicit choices.

## Runtime and UI

The dedicated `model-context-window:get/set` IPC validates metadata, account scope and active-session state. A successful gateway acknowledgement precedes local preference/session persistence. Model/mode changes consult that target's remembered explicit choice. User changes use session patching; they do not rewrite shared model defaults or request a gateway restart.

The model detail card shows the current context value in a compact row. Hovering or focusing that row opens a nested slider snapping to declared fixed lengths, with a restore-follow-default action. Merely opening/focusing it or previewing a drag never saves a preference. Release/keyboard confirmation commits the chosen budget. Explicit 400K differs from follow default even when both currently resolve to400K. A current default outside fixed choices is represented as follow-default, not a newly authorized fixed option. Both Chinese and English copy include credit and previously compacted history guidance.

The thinking and context rows use mutually exclusive nested panels. Editing the selected model's thinking level sends only `thinkingLevel`, so it cannot trigger the model-switch path and reset a session's independently selected window. Reselecting the same bound concrete model likewise retains the current session window. Both panels use the client's light and dark theme tokens and remain within the viewport on narrow windows.

Pinned OpenClaw `v2026.8.1` receives a replayable version patch. Only an exactly matched, declared `user-context:*` option on `lobsterai-server` may raise the scalar default. Embedded budget, preflight/memory flush, session projection and fork copy budgets must agree. Other providers and undeclared/default selections retain their existing path.

## Deployment and fallback behavior

- Deploy the additive database migration before the server reads the new columns. Existing rows remain null, and configuring selectable lengths is separate from configuring a default.
- Deploying the server before the desktop is supported by capability gating. Deploying the desktop before the server leaves the selector hidden and preserves the existing default path.
- Rebuild the pinned OpenClaw runtime with the version-scoped patch when distributing the desktop. Renderer selection and runtime budgeting must use the same declared options.
- Reject changes while a session is active. If the gateway rejects an update, keep the previous local preference and session state.
- Revalidate explicit selections against refreshed model capabilities. A withdrawn option cannot authorize a larger budget; the client restores the current model or preset default without turning it into explicit intent.

Increasing the budget does not recover history that was already compacted. A larger allowed window can increase credit consumption; the UI communicates both effects alongside the control.
