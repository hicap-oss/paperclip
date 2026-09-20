# Hicap

Hicap (`https://api.hicap.ai/v1`) is an OpenAI-compatible inference endpoint.
This page documents how to route Paperclip agent runs through it.

## Why Hicap is not an `AI_PROVIDERS` entry

Paperclip's first-class AI provider list is deliberately closed. See
`packages/shared/src/ai-connections.ts`:

```ts
export const AI_PROVIDERS = ["anthropic", "openai", "openrouter", "xai"] as const;
```

Each entry in `AI_CONNECTION_CAPABILITIES` binds a provider to the local CLI
harness adapters that can actually authenticate against it (`claude_local`,
`codex_local`, `opencode_local`, `grok_local`). The enum is not a catalog of
"endpoints Paperclip can call"; it is the set of vendors whose *harness*
sign-in, subscription, and managed-credential paths Paperclip implements. It is
also mirrored into DB `CHECK` constraints
(`packages/db/src/schema/ai_provider_defaults.ts`,
`ai_connection_defaults.ts`) and into the key-validation endpoint table in
`server/src/routes/ai-connections.ts`.

Adding a vendor there implies harness-level support, not just a base URL. For a
plain OpenAI-compatible endpoint the correct layer is the OpenCode custom
provider config, which upstream already supports as declarative configuration.
This follows the packaging rule in [README](./README.md): place the change on
the narrowest layer that solves the problem, and do not create a parallel
provider framework.

## Routing runs through Hicap

`opencode_local` merges arbitrary OpenAI-compatible providers from the
`PAPERCLIP_OPENCODE_PROVIDERS` environment variable
(`packages/adapters/opencode-local/src/server/runtime-config.ts`). The value is
a JSON object in OpenCode's `provider` shape. `PAPERCLIP_OPENCODE_PROVIDERS` is
on the `AI_AUTH_ENV_KEYS` pass-through allowlist in
`server/src/services/ai-connection-runtime.ts`, so it survives into the run
environment.

Set the Hicap API key in the environment as `HICAP_API_KEY` and reference it
with a `{env:...}` placeholder rather than inlining the secret. Paperclip
resolves the placeholder server-side before writing the runtime
`opencode.json`.

```sh
export HICAP_API_KEY='<hicap-api-key>'
export PAPERCLIP_OPENCODE_PROVIDERS='{
  "hicap": {
    "npm": "@ai-sdk/openai-compatible",
    "name": "Hicap",
    "options": {
      "baseURL": "https://api.hicap.ai/v1",
      "headers": { "api-key": "{env:HICAP_API_KEY}" }
    },
    "models": {
      "hicap-default": {}
    }
  }
}'
```

Notes:

- Hicap authenticates with an `api-key` request header, not an
  `Authorization: Bearer` header, so the key goes under `options.headers` rather
  than `options.apiKey`.
- OpenCode resolves `--model provider/model` only when the model id exists in
  that provider's `models` map, so list every Hicap model you intend to select.
  Replace `hicap-default` with the real model ids your Hicap account serves.
- Select the model on the agent as `hicap/<model-id>`.
- If a run needs OpenCode's auxiliary small model (session titles and similar
  helper calls) to stay on Hicap, also set
  `PAPERCLIP_OPENCODE_SMALL_MODEL=hicap/<model-id>`. Otherwise OpenCode falls
  back to a built-in provider default that Hicap may not serve, and the helper
  call fails the run.
- Never commit a real key. `HICAP_API_KEY` belongs in the deployment
  environment or the instance vault, consistent with decision D2 in
  [README](./README.md).

## Promoting Hicap to a first-class provider

If Hicap later needs managed connections, key validation, and provider defaults
in the UI, that is a full connector change, not a config change. Follow the
[connection authoring runbook](./CONNECTOR-PLAYBOOK.md) and expect to touch:

1. `AI_PROVIDERS` and `AI_CONNECTION_CAPABILITIES` in
   `packages/shared/src/ai-connections.ts`, including which adapters can use it.
2. An app definition under `packages/shared/src/app-definitions/`.
3. The `provider` `CHECK` constraints and a migration in `packages/db`.
4. The `endpoints` table in `validateAiApiKey`
   (`server/src/routes/ai-connections.ts`).
5. `AI_AUTH_ENV_KEYS` in `server/src/services/ai-connection-runtime.ts`.

That work should be driven by a real product requirement for managed Hicap
credentials; the `PAPERCLIP_OPENCODE_PROVIDERS` route above is sufficient for
routing inference today.
