# External calls audit

Date: 2026-08-30. Scope: every outbound network call made by `apps/api`, `apps/worker`,
`apps/bot`, `packages/*` at runtime, plus the third-party dependencies they load at
runtime (Mastra, AI SDK, auth, mailer, PDF parsing). Method: full read of the app code,
source inspection of the published npm packages the lockfile pins (@mastra/core 1.58.0,
@mastra/memory 1.26.1, @mastra/pg 1.20.0, @firecrawl/pdf-inspector 1.17.0, chat 4.37.0),
grep of the dependency tree for telemetry libraries and install hooks, and a walk of the
web client code.

## Verdict

No hidden exfiltration of project content (issues, comments, attachments, prompts) was
found. Every path that sends app data to a third party is either a feature the operator
explicitly switched on with their own credentials, or the documented daily instance
telemetry. But for a "local models only" setup there are still five external touchpoints
to know about, two of which are active out of the box:

| # | Channel | Destination | Active by default? | Carries app data? |
|---|---------|-------------|--------------------|-------------------|
| 1 | Instance telemetry | `telemetry.itsaplan.dev` | **Yes** (worker) | Metadata only, bucketed. No content, no names, no credentials. Opt out: `TELEMETRY_DISABLED=1` or `DO_NOT_TRACK=1` |
| 2 | Model catalogue | `models.dev/api.json` | On demand (agent config UI) | No. No auth, no query, no model key |
| 3 | Update check | `github.com/croffasia/itsaplan/releases.atom` | On demand (Settings > Updates) | No. The instance version is visible to github.com in the request |
| 4 | Mastra vendor analytics | `us.posthog.com` | **Code is on by default**, but no code path in this app reaches it (see 2.2) | Would carry hashed hostname, OS, agent counts, per-model token usage |
| 5 | Feature integrations (Jina, Firecrawl, Notion, Telegram, Instagram, Threads, Gitea tools; Resend mail; webhook delivery) | The vendor APIs | Only when configured/enabled | **Yes: prompt-derived content.** You chose this by wiring the tools |

And one misconfiguration trap (section 2.1): a local model registered under a
catalog provider key (e.g. `openai`) instead of `openai-compatible` routes prompts and
the "API key" to that provider's public API.

## 1. How model calls are routed (the local-model question)

- An internal agent's model is resolved in
  `apps/api/src/modules/agents/core/runtime/index.ts` (`resolveModel`): it reads the
  credential row, takes `integrationKey` as `providerId`, the agent's model string as
  `modelId`, the decrypted `apiKey`, and an optional `baseUrl`. The result is handed to
  Mastra as `{ providerId, modelId, apiKey, url? }`.
- Mastra's router (verified in the published `@mastra/core@1.58.0` source):
  - **With `url` set**, it builds `createOpenAICompatible({ baseURL: url })`. The call
    goes exactly to that URL and nowhere else. That is the path a local model takes.
  - **Without `url`**, it uses a *bundled* provider registry (generated from models.dev
    data at build time, compiled into `provider-registry-*.js`). `openai` resolves to
    the OpenAI native SDK (api.openai.com), `anthropic` to api.anthropic.com, and so on.
    No models.dev fetch happens at generation time; the live `fetch("https://models.dev/api.json")`
    in the gateway only runs in `listProviders()`/`listAvailableModels()`
    (catalog listing for Studio/CLI), which this app never calls.
- Practical rule: **register local Ollama/MLX/vLLM endpoints as the
  `openai-compatible` provider** (`apps/api/src/modules/agents/integrations/llm-providers.ts`,
  the only entry with `requiresBaseUrl: true`), base URL e.g.
  `http://<host>:11434/v1`, any API key string. Then the model call is a plain
  OpenAI-compatible HTTP call to your endpoint.
- Trap 1 (default model): if the provider is `openai` and the agent has no model set,
  the default is `gpt-5-mini`, so a run with no model set goes to api.openai.com.
- Trap 2 (provider key choice): storing credentials under `openai`/`anthropic`/`google`
  etc. with no base URL sends the prompt and key to that provider's public endpoint.
  There is no guard preventing that; it is a configuration choice.
- The model picker UI fetches `https://models.dev/api.json`
  (`apps/api/src/modules/agents/integrations/provider-models.ts`, 24h cache) to list
  models. No auth, no app data. A self-hosted instance reaching models.dev for this is
  the only "why did my box talk to the internet when I just opened the agent settings"
  of the kind.

## 2. Channels that do not depend on model choice

### 2.1 Project telemetry (active by default)

`apps/worker/src/telemetry.ts`, documented in `TELEMETRY.md`.

- One POST a day to `https://telemetry.itsaplan.dev/v1/telemetry` (first send at worker
  start, then a fixed random minute per day). Retries 5 times, then hourly.
- Opt-out: `TELEMETRY_DISABLED=1` or `DO_NOT_TRACK=1` (any value except 0/empty). The
  endpoint is hard-coded; it is not configurable to another host.
- What goes out (code verified against `telemetry-payload.ts` and `buildPulse`):
  random UUID instance id, UTC day, install day, app version, Bun version,
  platform/arch, Docker yes/no, Postgres major, bucketed counts (users, projects,
  issues, agents, runs, attachments size, webhook and run failure rates), feature
  flags (ever / last 30 days), integration catalogue keys for which credentials are
  stored (e.g. `openai`, `jina`), git provider names, locale codes, sign-up policy.
  No names, no project keys, no issue text, no URLs, no credentials, no exact counts.
- Minor note: the `integrationKeys` list is filtered to lowercase identifiers
  (`^[a-z0-9][a-z0-9._-]{0,31}$`), not to the fixed catalogue. Keys created through the
  UI are catalogue keys, but a hand-inserted row with a meaningful name would be sent.
- Impact in a local-only setup: daily DNS + HTTPS to the project's collector
  plus the metadata profile. If that is unacceptable, set `TELEMETRY_DISABLED=1`.
  Failure is logged and retried; it never affects delivery.

### 2.2 Mastra's bundled vendor analytics (dormant in this code path)

Highest-signal dependency finding: `@mastra/core@1.58.0` carries a hard dependency on
`posthog-node` (lockfile: `posthog-node@5.48.2`) and a telemetry module
(`dist/telemetry/index.js`) inspected from the published package:

- `POSTHOG_HOST = "https://us.posthog.com"`, a fixed API key,
  `isTelemetryEnabled()` returns **true unless `MASTRA_TELEMETRY_DISABLED` is 1/true/yes**.
- Events it can send: `mastra_instance_summary` (agent/workflow/tool counts, storage
  type), `mastra_model_token_usage` (provider + model + input/output token totals),
  `mastra_feature_usage`; all with system properties (os, node version, arch,
  `machine_id` = SHA-256 of the hostname truncated to 16, `project_id` = SHA-256 of the
  working directory truncated).
- What the audit found in the same dist: **no file other than `telemetry/index.js`
  references the PostHog key, the event names, or imports the telemetry module**. The
  Agent chunk contains 37 occurrences of the word "telemetry", but all are local
  observability variables (attempt/model-call counters), not PostHog. The app imports
  only `@mastra/core/agent`, `@mastra/core/tools`, and `@mastra/memory`/`@mastra/pg`,
  and `@mastra/memory@1.26.1` and `@mastra/pg@1.20.0` contain no PostHog code.
- Conclusion: with the versions this repo pins and the code paths it uses, nothing
  phones PostHog. The capability is present, on by default in the library code, and
  only the Mastra server/studio/CLI entry points would trigger it. A Mastra upgrade
  could wire it into the Agent path.
- Cheap hardening: set `MASTRA_TELEMETRY_DISABLED=1` in the api environment. Zero
  cost, removes the class of finding for any future version.

### 2.3 Update check and skill import (on demand)

- `apps/api/src/modules/settings/updates.ts`: fetches the GitHub releases atom feed
  when someone opens the update section. Sends nothing but a UA header; the request
  itself (and the response) reveals only that an instance checked the feed.
- `apps/api/src/modules/agents/skills/skill-format.ts`: importing a skill from a public
  GitHub repo resolves a commit sha via `github.com/.../commits/<ref>` atom, then reads
  the file tree and file content through `data.jsdelivr.com` / `cdn.jsdelivr.net`
  (deliberate, to avoid GitHub API rate limits). Only public repos, only on explicit
  import. Exposes the repo path (which is public).

## 3. Feature-scoped egress (only when you configure it)

All of these use credentials the operator stored, and are only reachable if a tool is
enabled on an agent or a channel is switched on. They are the product's purpose, not
bugs. In a local-model setup they are the main way *issue content* can leave the LAN,
because the model can feed it into them.

| Feature | Destination | What leaves | Trigger |
|---|---|---|---|
| Jina AI tools (search, reader, deepsearch, segment, classify, rerank, grounding) | `s.jina.ai`, `r.jina.ai`, `api.jina.ai`, `deepsearch.jina.ai` | Query text, URLs, and content to classify/segment/rerank (i.e. whatever the agent passes), Jina key | Model calls a Jina tool |
| Firecrawl tools (scrape, map, search, crawl, extract) | `api.firecrawl.dev` | URLs and extract criteria, Firecrawl key | Model calls a Firecrawl tool |
| Notion tools | `api.notion.com` | Page/comment content, Notion token | Model calls a Notion tool |
| Instagram / Threads tools | `graph.instagram.com` (graph.facebook.com), `graph.threads.net` | Post text, media, replies, FB graph token | Model calls the tools |
| Gitea tools | the configured Gitea instance | Issue text, tokens | Model calls the tools |
| Telegram send tool + instance bot + Telegram notifications | `api.telegram.org` | Message text, bot token | Chat/notifications |
| E-mail (auth mail, notifications, invites) | `api.resend.com` **or** the configured SMTP host | Full message body | Any e-mail send |
| Webhook delivery | operator-defined URLs | Signed event payloads (issue/comment/agent-run data) | Any webhook event |
| Attachment import from URL | target URL (SSRF-guarded) | The fetched file is stored; the fetch is the egress (UA only) | Model or user imports an attachment URL |
| Google / generic OIDC sign-in | Google / configured IdP | Auth flows only | A user signs in via them |

The internal agent's work-item tools (route tools) call this instance's own API with an
internal API key; PDF, DOCX and XLSX parsing happens in-process
(`@firecrawl/pdf-inspector` is a local native text extractor, inspected: no network
code; `mammoth`/`exceljs` are pure parsers). Conversation memory is Postgres
(`@mastra/pg`). The MCP surface is an inbound server only. The web frontend makes no
third-party calls: all `fetch` calls in `apps/web/src` are relative to the configured
API origin, there is no CDN, no client-side analytics, and fonts ship from the
`inter-ui` package. The Telegram bot app and the `@itsaplan/runner` CLI talk only to
the instance URL they were pointed at.

## 4. SSRF guard observations (adjacent to the leak question)

`apps/api/src/shared/net.ts` (`assertPublicHttpUrl`) guards server-side fetches of
user/model-supplied URLs (used by attachment import): https-only in production,
rejects local hostnames and private/loopback/link-local/CGNAT IPs, resolves DNS and
rejects hostnames that map to private addresses, and the import route blocks redirects.
Weaknesses, in decreasing order of interest:

1. **TOCTOU / DNS rebinding**: the resolution happens at validation time; the actual
   `fetch` re-resolves. A hostname that resolves publicly at check time and to
   127.0.0.1 at fetch time (rebinding TTL trick) passes. Fix: perform the fetch with a
   custom `lookup` that re-checks the address, or connect to the resolved IP with a
   Host header.
2. **IPv4-mapped IPv6 bypass**: `isPrivateIp` does not recognize `::ffff:127.0.0.1`
   (and friends). The v4 regex does not match the mapped form, and the v6 checks miss
   it. A URL like `https://[::ffff:10.0.0.1]/...` is not flagged.
3. **Dev-mode relaxation**: with `NODE_ENV` none of `production`/`test`, the private
   check is skipped entirely. That matches "local dev hits local services", but it
   means a dev deployment is unguarded.
4. Webhook delivery by design has no guard (it is the feature), but note any member
   with webhook permission in a project can point deliveries at internal URLs.

None of these is content exfiltration; they are internal-reach vectors reachable when
an agent (any model, including a local one) can be induced to call the import endpoint.

## 5. Supply chain

- `bun.lock` contains **zero install/preinstall/postinstall/prepare scripts** for any
  package. The Dockerfiles run `bun install --frozen-lockfile`; nothing executes at
  install time.
- Telemetry-adjacent libraries in the resolved tree: `posthog-node` (via @mastra/core,
  section 2.2) only. No Sentry, Datadog, Amplitude, Mixpanel, Segment, or OpenTelemetry
  exporters.
- No `node_modules` in the audited checkout; dependency claims above were verified
  against the exact published tarballs in the lockfile (versions pinned in section 2.2).

## 6. Recommendations for a local-model, leak-averse deployment

1. `TELEMETRY_DISABLED=1` (or `DO_NOT_TRACK=1`) in the worker/api environment.
2. `MASTRA_TELEMETRY_DISABLED=1` in the api environment (dormant today, cheap to fix).
3. Store the local model under the `openai-compatible` integration with its base URL.
   Do not store it under `openai`/`anthropic`/other catalogue keys unless the traffic
   to those vendors is intended.
4. Enable only the agent tools you actually want; each enabled tool is a route for
   issue content to a vendor (section 3). A "local brain, local tools" agent with zero
   custom tools and no e-mail/webhook/Telegram channels is a closed loop: the only
   egress left is the model endpoint itself.
5. If total egress control matters, run the stack behind an egress firewall that allows
   only your model endpoint. With (1) and (2) set, every remaining external call in
   this audit is either feature-triggered or will fail visibly (telemetry retries are
   logged).
6. Consider fixing the two SSRF guard gaps (mapped-IPv6 recognition, resolve-at-connect)
   if agents with the attachment-import tool run against the public API.

## Appendix: external endpoints found in runtime code

| Endpoint | File | Purpose |
|---|---|---|
| `https://telemetry.itsaplan.dev/v1/telemetry` | apps/worker/src/telemetry.ts | daily instance snapshot |
| `https://models.dev/api.json` | apps/api/.../integrations/provider-models.ts; @mastra/core (catalog listing only) | model list for UI |
| `https://github.com/croffasia/itsaplan/releases.atom` | apps/api/.../settings/updates.ts | update check |
| `https://github.com/<owner>/<repo>/commits/...` (atom) | apps/api/.../skills/skill-format.ts | pin ref to sha for skill import |
| `https://data.jsdelivr.com/v1/packages/gh`, `https://cdn.jsdelivr.net/gh` | apps/api/.../skills/skill-format.ts | skill file content |
| `https://s.jina.ai/`, `r.jina.ai`, `g.jina.ai`, `api.jina.ai`, `deepsearch.jina.ai` | packages/agent-tools/src/tools/jina/* | Jina tools |
| `https://api.firecrawl.dev/v2` | packages/agent-tools/src/tools/firecrawl/* | Firecrawl tools |
| `https://api.notion.com/v1` | packages/agent-tools/src/tools/notion/* | Notion tools |
| `https://graph.facebook.com/*`, `graph.threads.net/*` | packages/agent-tools/src/tools/{instagram,threads}/* | social tools |
| configured Gitea URL | packages/agent-tools/src/tools/gitea/* | Gitea tools |
| `https://api.telegram.org/bot...` | apps/api/.../telegram/service.ts, .../notifications/send.ts, packages/agent-tools/src/tools/telegram/* | bot + notifications |
| `https://api.resend.com/emails` | packages/mailer/src/index.ts | e-mail via Resend |
| `https://us.posthog.com` | @mastra/core dist/telemetry/index.js (dormant here) | vendor analytics |
| provider public APIs (api.openai.com, api.anthropic.com, ...) | bundled registry in @mastra/core | model calls for non-custom provider keys |
| user-configured SMTP host, IdP discovery URLs, webhook URLs, S3 endpoint | various | operator-chosen destinations |
