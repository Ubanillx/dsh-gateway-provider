# dsh-gateway-provider

[![gitleaks](https://github.com/Luck9Star/dsh-gateway-provider/actions/workflows/gitleaks.yml/badge.svg)](https://github.com/Luck9Star/dsh-gateway-provider/actions/workflows/gitleaks.yml)

> 中文文档：[docs/README.zh.md](docs/README.zh.md)

Use **all the models behind your LLM gateway** — newapi, LiteLLM, Higress,
or any OpenAI-compatible endpoint — directly in DeepSeek Harness.

Install the plugin, paste your API key, and every gateway model shows up in
dsh's model picker with its **real** parameters (context window, output cap,
reasoning support) fetched from [models.dev](https://models.dev). Requests
go out over each model's own native protocol — OpenAI, Anthropic, or Gemini
— so tool calls and streaming behave the way that model's maker intended.

## Why this exists

dsh ships one adapter per official provider. If your models live behind a
gateway, the manual alternative is a hand-maintained static model list with
guessed context windows and output caps. This plugin mounts the gateway
itself instead:

- **Nothing to maintain by hand** — the model list is read from the gateway
  (`GET /v1/models`, with a management-API fallback for newapi); add a model
  on the gateway side and it appears in dsh, no re-deploy.
- **Real numbers, not guesses** — models.dev data fills context window,
  output cap, reasoning levels, release date; config defaults only fill gaps.
- **Every wire format, one plugin** — each model routes over its own
  protocol (OpenAI chat completions / OpenAI responses / Anthropic messages
  / Gemini), handled by the same
  [pi-ai](https://www.npmjs.com/package/@earendil-works/pi-ai) SDK the
  official dsh adapter uses.
- **Multiple gateways at once** — a default `newapi` route plus one
  `gateway:<id>` route per extra gateway, each with its own cache and key.
- **A settings page instead of YAML** — **Settings → Gateway Models**: add
  gateways from templates (NewAPI / LiteLLM / Higress / OpenAI-compatible /
  fully custom), test connections, sync models, hide or override any model,
  add custom models.

## Requirements

- DeepSeek Harness (dsh) with a `web` profile (the settings page is a web-UI
  extension; the provider itself works in any profile).
- A gateway API key (e.g. a newapi token).

## Install from GitHub source (agent instructions)

Install the checkout itself, not the npm build: the published package may lag
this repo (registry builds predate unreleased fixes). The steps below are
written for whoever performs the install — exact paths, commands and checks.
Prereqs: `pnpm` (≥ 10); on Windows, Git Bash (`bash` on `PATH`, the repo's
`prepare` hook runs `scripts/link.sh`).

1. **Clone and install the repo** — it must own its pinned pi-ai; harness
   packages are linked to the hosting profile so the process keeps one copy:

   ```sh
   git clone <repo-url> dsh-gateway-provider
   cd dsh-gateway-provider
   DSH_HOME=<harness-home> pnpm install
   ```

   `<harness-home>` = `~/.dsh` for plain dsh CLI, or
   `%APPDATA%\dsh-desktop\harness` for DSH Desktop; pointing `DSH_HOME` there
   makes the `prepare` hook symlink `node_modules/@deepseek-ai/*` into that
   harness's shared tree (single instance). After install, check that
   `node_modules/@earendil-works/pi-ai/package.json` exists and
   `node_modules/@deepseek-ai/cordis` is a link (directory junction on
   Windows). Where symlinks are not permitted, recreate each failed link as a
   junction: `cmd /c mklink /J <link> <target>`.

2. **Wire the profile** (back up its `package.json` first):

   - profile dir: `$DSH_HOME/profiles/<name>` (CLI; usually `web`);
     Desktop: `%APPDATA%\dsh-desktop\harness\profiles\web`;
   - add to the profile `package.json` `dependencies`:
     `"dsh-gateway-provider": "link:<abs-path-to-clone>"`;
   - register the plugin **once** by listing the package in the profile's
     `dsh.profile.bundles`:

     ```json
     "bundles": [ "...", "dsh-gateway-provider" ]
     ```

     The plugin's own `cordis.patch.yml` (via its `dsh.bundle.patch`
     manifest) then supplies the `llm-newapi` row. Do **not** also insert a
     manual `- id: llm-newapi` row into the profile's `cordis.patch.yml` —
     the row would be defined twice and the harness refuses to boot with
     `duplicate loader entry id: llm-newapi` (DSH Desktop then auto-removes
     the plugin through plugin recovery).
   - run pnpm inside the profile dir: `pnpm install`. DSH Desktop must use its
     own runner (store pinning + EPERM recovery):
     `node "<desktop>\resources\app\node_modules\node\bin\node.exe" "<harness>\.desktop-bin\pnpm-runner.mjs" "<desktop>\resources\app\node_modules\pnpm\bin\pnpm.cjs" install`

3. **Restart the harness** — Desktop: fully quit and relaunch the app (the
   harness is a child of the shell); CLI: restart `dsh`.

4. **Verify** — `Settings → Gateway Models` renders the management UI; the
   model picker gains a "NewAPI" route. On failure check the harness log
   (Desktop: `%APPDATA%\dsh-desktop\logs\harness.log`; the plugin row is
   `llm-newapi`, package `dsh-gateway-provider`).

5. **Provide the API key** — write the credential (default env var name
   `NEWAPI_API_KEY`) into the harness credentials file (`~/.dsh/.credentials.yaml`;
   Desktop: `%APPDATA%\dsh-desktop\harness\.credentials.yaml`) or let the
   settings page store it. Open the gateway card → **Test** — expect
   `✓ Connected — N models`. Not the public newapi cloud? Set **Base URL**
   on the card first.

Iterating = editing the clone and restarting the harness; requests then run
the new code (`lib/client.js` is the shipped UI artifact — no build step).
Uninstall: remove the bundle entry and the dependency, then re-run
`pnpm install` in the profile.

## Daily use

Everything lives in **Settings → Gateway Models**:

- **Add more gateways** — "Add Gateway", pick a template (LiteLLM, Higress,
  OpenAI-compatible, or fully custom with per-protocol URLs), point it at
  the base URL, name its key env var, Test, Sync. Each gateway becomes its
  own route in the picker.
- **Tame the model list** — non-chat models (image / speech / embedding /
  rerank …) are excluded by default regexes; hide or rename any model; add
  a custom model by hand if the gateway hides it; per-model protocol,
  context window, output cap, reasoning levels, and image input are all
  editable. A vision model missing from models.dev (or a model you want to
  keep text-only) can be pinned via the model editor's "Image input" picker
  (`auto` / image-capable / text-only), which writes `inputModalities`.
- **Keys live in dsh's credential store** — the settings page shows a badge
  (`✓ Key set · NEWAPI_API_KEY` / `⚠ No key set`) and can write the key
  there for you.

## Configuration reference

Optional — everything below has a working default. Config lives in the
`llm-newapi:` section of `~/.dsh/settings.yaml` (the settings page edits
the same keys). The frequently used ones:

| Key | Default | Meaning |
| --- | --- | --- |
| `baseURL` | `https://api.newapi.ai` | Your gateway's base URL. Env fallbacks: `NEWAPI_BASE_URL`, `NEWAPI_API_URL`. |
| `apiKeyEnv` | `NEWAPI_API_KEY` | Which env/credential variable holds the key. |
| `label` | `NewAPI` | Route label shown in the picker. |
| `flavor` | `newapi` | Template label only (`newapi` / `litellm` / `higress` / `openai-compatible` / `custom`). |
| `gateways` | — | Array of extra gateways: `{ id, baseURL, apiKeyEnv, label, … }`, each becoming a `gateway:<id>` route. |
| `models` | — | Per-model overrides: `{ id, name, disabled, protocol, contextWindow, maxTokens, reasoningLevels, inputModalities }`. |
| `useModelsDev` / `modelsUrl` | `true` / models.dev | Parameter enrichment source (supports `file:` URLs for offline). |
| `excludePatterns` | image/speech/… | Regex list of model ids to keep out of the picker. |
| `sortModelsByRelease` | `true` | Newest models first. |
| `catalogMode` | `auto` | `v1` (`/v1/models` only) / `management` (newapi user API) / `auto`. |
| `endpointPriority` | responses → anthropic → openai → gemini | Which protocol to prefer when a model supports several. |
| `openaiURL` / `responsesURL` / `anthropicURL` | — | Fully-custom gateways only: per-protocol endpoint URLs; unset = that protocol off. |
| `maxTokens` / `defaultContextWindow` | `32768` / `128000` | Fallbacks when models.dev has no data. |
| `streamIdleTimeoutMs` | `600000` | Idle timeout while streaming. |
| `headers` | — | Extra HTTP headers sent to the gateway. |

## Troubleshooting

| Symptom | Cause → fix |
| --- | --- |
| Picker route exists but zero models | The plugin can't read your model list. Check the gateway base URL; try `catalogMode: "management"` for newapi gateways that restrict `/v1/models`. |
| `401` / auth errors on every request | Key missing or wrong: check the badge in Settings → Gateway Models, or `NEWAPI_API_KEY` in `~/.dsh/.credentials.yaml`. |
| Settings → Gateway Models stays blank, or boot shows `NewAPI 加载失败 … no API key` | An older build threw during boot-time model enumeration when no key was set, which also skipped the settings registration. Current builds report zero models instead: page renders with the `⚠ No key set` badge — enter the key there (or write the credential / export `NEWAPI_API_KEY`), then the route populates. |
| A model's context window looks wrong | models.dev had no match. Edit the model on the settings page (or a `models:` override). |
| Wrong format answers / tool calls flaky for one model | That model is routed over a protocol it handles poorly. Pin `protocol` on the model (`openai`, `openai-response`, `anthropic`, `gemini`). |
| Custom gateway with separate endpoints | Use `flavor: "custom"` and set `openaiURL` / `responsesURL` / `anthropicURL` explicitly. |

## How it works (one minute version)

At startup the plugin registers one provider route per gateway, pulls the
model list from the gateway, and fuzzy-matches each model id against
models.dev to fill in real parameters. When you pick a model, dsh's request
is translated to the pi-ai SDK's format and sent over that model's native
protocol; the streamed reply is translated back into dsh chunks. Catalogs
are cached (30 min by default) per gateway. No hand-written protocol code —
the bridge is lifted from the official `dsh-llm-pi-ai` adapter.

## Development

```sh
git clone <repo-url>        # the checkout being installed / developed
cd dsh-gateway-provider
pnpm install               # pi-ai (pinned) + @deepseek-ai/* profile symlinks, via the prepare hook
pnpm run test:client       # settings-UI render, both locales
pnpm run test:urls         # URL/derivation units
pnpm run smoke             # live gateway round-trip (needs a real key)
```

### Why the plugin pins its own pi-ai

`@earendil-works/pi-ai` is a direct dependency (exact-pinned), independent of
the pi-ai version bundled with the harness. This decouples the gateway model
catalog (thinking levels, per-provider compat such as zhipu GLM's
`supportsDeveloperRole: false`) from harness upgrades: a model missing from
the harness's older catalog no longer degrades request encoding. The
plugin↔harness boundary passes plain data (`GenerateOptions` in, dsh
`StreamChunk`s out; `lib/pi-bridge.js` never leaks pi-ai objects across), so
the plugin's pi-ai copy and the harness's own coexist safely in one process.

Developing from a checkout is the install loop above: edit the clone and
restart the harness. Keep the single registration described there — the
package listed in `dsh.profile.bundles`, no manual row in the profile's
`cordis.patch.yml` (a duplicate `llm-newapi` row is a loader error). Offline
suites: `pnpm run test:client`, `test:urls`, `test:schema`, `test:errors`;
`smoke` needs a live gateway key.

## References & credits

- [pi-ai SDK](https://www.npmjs.com/package/@earendil-works/pi-ai) — all
  four wire protocols; the bridge reuses the official `dsh-llm-pi-ai`
  adapter's translation layer.
- [models.dev](https://models.dev) — the parameter catalog (context
  windows, output caps, reasoning, release dates).
- [new-api](https://github.com/QuantumNous/new-api), 
  [LiteLLM](https://github.com/BerriAI/litellm),
  [Higress](https://github.com/alibaba/higress) — the gateways this plugin
  is tested against (any OpenAI-compatible endpoint works).

## Security

Keys live in dsh's credential store or the launching environment — never in
settings YAML. The repo runs [gitleaks](https://github.com/gitleaks/gitleaks)
in CI and pre-commit to keep secrets out.

## License

[MIT](LICENSE)
