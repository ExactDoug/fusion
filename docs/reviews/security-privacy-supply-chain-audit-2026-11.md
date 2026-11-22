# Security, privacy, telemetry and supply-chain audit (operator's view)

- **Date:** 2026-11-21 (UTC)
- **Source reviewed:** `main` at `eb7eef49d` (workspace version `0.78.0-beta.9`)
- **Published artifacts checked:** npm `@runfusion/fusion@0.77.0` (`latest`), npm `runfusion.ai@0.77.0`, npm `@earendil-works/pi-telemetry@1.1.0`
- **Question asked:** "Is it safe to run Fusion? What telemetry does it have, what does it collect, and how safe is its supply chain?"

> **Status (2026-11-22):** first pass complete. Section 3 has two corrections from watching the network traffic of a fresh 0.77.0 install: the OpenRouter and OrcaRouter fetches happen by default, and Fusion announces itself over mDNS by default. The [Open items](#open-items) are now tracked outside this fork. Branch `docs/security-privacy-audit`, PR ExactDoug/fusion#1 (fork only; never open it against upstream `Runfusion/Fusion`).

## Scope and method

This is a **targeted source review**, not a full security audit. It answers an operator's adoption questions by reading code, configuration and published npm metadata. Every claim below cites the file it rests on. Items that were *not* verified are listed under [Open items](#open-items), and should not be read as cleared.

Out of scope: a line-by-line review of the dashboard API's authorization, the remote-access tunnel feature, the Docker image, the desktop (Electron) build, plugins, and whether the npm tarball is byte-identical to a build of this source.

## Summary

| Area | Verdict |
|---|---|
| Phone-home telemetry / analytics | **None found.** No analytics or crash-reporting SDK. All "telemetry" in the code is written to Fusion's own local database. |
| What it collects | Detailed **local** activity records (usage events, agent logs and transcripts, run-audit). Encrypted secrets. Nothing of this is uploaded by Fusion itself. |
| Data leaving the machine | Dominated by **your configured AI provider**, which receives code, prompts and task context. A few small functional calls besides: the update check, model pricing, opt-in integrations, and, **by default**, OpenRouter and OrcaRouter model-list fetches on every start plus an mDNS announcement on the local network (corrected 2026-11-22). |
| Supply chain | **Mixed.** Good runtime download hygiene (pinned SHA-256), Dependabot, pinned agent SDK. But the `latest` release has **no npm provenance**, is published from a **single maintainer account**, and pulls a large dependency tree. |
| Agent execution | **The biggest real risk.** Agents run shell commands as the operator's user, **unsandboxed by default**. |
| Network exposure | Reasonable defaults: binds `127.0.0.1` with bearer-token auth. `/metrics` is unauthenticated by design. |

## 1. Telemetry

**No third-party telemetry or analytics was found in Fusion's source.**

- No analytics, crash-reporting or tracing SDK is a direct dependency of any workspace package (searched `packages/*/package.json`, `plugins/*/package.json` and the root `package.json` for PostHog, Sentry, Mixpanel, Segment, Amplitude, OpenTelemetry, Datadog, New Relic, Bugsnag, Rollbar, LogRocket, Vercel Analytics, Plausible and Umami).
- `sentry.io` and `app.fusion.dev` URLs appear in source trees **only inside test files**.
- `@opentelemetry/api@1.9.0` appears in `pnpm-lock.yaml` only as an optional peer of `drizzle-orm` and `vitest`. The API package is a no-op without a configured SDK/exporter, and none is configured.
- `@earendil-works/pi-telemetry@1.1.0` is a runtime dependency of the published package (pulled in by the pi agent SDK, pinned in `pnpm-workspace.yaml`). The tarball was downloaded and read: it contains only interfaces, a no-op context and an in-memory reference implementation. Its README states "no exporter, global current-span state, or dependency on a telemetry backend", and its `dist/*.js` contains no network calls. Fusion's own source never imports it.

What the code calls "telemetry" is local:

- `packages/engine/src/agents/agent-usage-telemetry.ts` emits session-boundary events through `TaskStore.emitUsageEvent` (`packages/core/src/store.ts:3752`).
- Those rows land in the `usage_events` table (`packages/core/src/postgres/schema/project.ts:2018`), which the Command Center analytics read.

## 2. What Fusion collects (locally)

| Record | Fields / content | Where |
|---|---|---|
| Usage events | Timestamp, kind (`tool_call`, `tool_result`, `tool_error`, `user_message`, `session_start`, `session_stop`), task, agent, node, model, provider, tool name, category, and a `meta` JSON blob. There is no free-text column; `user_message` events are documented as content-free. | `usage_events` table, `packages/core/src/postgres/schema/project.ts:2018`; `packages/core/src/tasks/usage-events.ts` |
| Agent logs / transcripts | Full per-task agent session logs, including model-visible tool results. | Per-task `agent-log.jsonl`, see `docs/storage.md` |
| Run-audit events | Lifecycle and mutation events. Project convention keeps metadata to ids, counts and fixed outcomes, never prose. | `run_audit_events`, see `AGENTS.md` → Run Audit |
| Secrets | AES-256-GCM ciphertext plus nonce. The master key is held in the OS keychain, with a file fallback. | `project.secrets`, `central.secrets_global`; `docs/secrets.md` |
| Provider logins (added 2026-11-22) | API keys and OAuth tokens, in plain JSON (mode 600), **not** in the encrypted secrets store. **Fusion also copies other CLIs' logins into it without asking:** Claude Code's OAuth token (`~/.claude/.credentials.json`, `~/.config/claude/.credentials.json`) and Codex's (`~/.codex/auth.json`) are imported whenever its auth storage is created, and Fusion then calls the provider and refreshes the token itself. Logging Fusion out suppresses the import only in memory, so the next process imports it again. Practical effect: logging in to Claude Code on a Fusion host makes Fusion's direct `anthropic` provider usable with the subscription token, even if the operator only meant to use the Claude CLI route. | `~/.fusion/agent/auth.json`; `packages/engine/src/auth/auth-storage.ts:513` (`getSupplementalAuthPaths`), `:777`, `:806`; `packages/core/src/secrets/oauth-credential-interop.ts` |

None of these are uploaded by Fusion. They do, however, hold your code context and agent conversations at rest, so protect the database and the `~/.fusion` directory as you would the source code.

## 3. What leaves the machine

| Destination | When | What is sent | Evidence |
|---|---|---|---|
| **Configured AI provider(s)** | Every agent session | Source code, prompts, task context, tool output. This is the dominant privacy exposure, and it is inherent to the product. | Provider/runtime configuration (`docs/settings-reference.md`) |
| `registry.npmjs.org` | Update check, **on by default** (`updateCheckEnabled: true`, `packages/core/src/config/settings-schema.ts:245`) | A plain unauthenticated GET for the package metadata | `packages/dashboard/src/update-check.ts:16`, `:430` |
| `raw.githubusercontent.com` | Model pricing refresh | Downloads LiteLLM's public price list; nothing of yours is sent. (Whether it can be disabled was not checked.) | `packages/core/src/ai/model-pricing.ts:45` |
| OpenRouter / OrcaRouter | **On every start, by default, even when neither is configured** (corrected 2026-11-22; this row first said "only if configured"). Turned off with `openrouterModelSync: false` and `orcarouterModelSync: false`. | An unauthenticated request for the public model list, with `HTTP-Referer: https://runfusion.ai` and `X-Title: Fusion` headers for app attribution | `packages/cli/src/commands/startup-model-sync.ts:5`, `:161`; defaults at `packages/core/src/config/settings-schema.ts:239`, `:243` (v0.77.0). Observed on a fresh 0.77.0 install with no provider configured. |
| Local network (mDNS multicast) | **On by default** (added 2026-11-22). Turned off with `localNetworkDiscoveryEnabled: false`. | A `_fusion._tcp` DNS-SD announcement carrying the machine's hostname, Fusion's node ID, its version and the dashboard port; Fusion also listens for other nodes | `packages/core/src/mesh/node-discovery.ts`; `packages/cli/src/commands/dashboard.ts` (around line 3080); default at `settings-schema.ts:149` (v0.77.0). Observed on a fresh 0.77.0 install. |
| `skills.sh` | Only when browsing or installing skills | Search queries (overridable with `SKILLS_API_URL`) | `packages/cli/src/commands/skills.ts:33`; `packages/dashboard/src/skills-adapter.ts` |
| `ntfy.sh` (or a custom ntfy server) | Only if notifications are configured | Notification messages. On public `ntfy.sh`, anyone who guesses the topic name can read them, so use a random topic or self-host. | `packages/engine/src/notification/ntfy-provider.ts:304`; `packages/engine/src/util/notifier.ts:23` |
| GitHub / GitLab / Linear | Only if connected | What each integration needs (issues, PRs, webhooks) | `packages/dashboard/src/github.ts`, integration routes |

## 4. Supply chain

### Concerns

1. **No provenance on the current release.** `npm view @runfusion/fusion@0.77.0 dist` shows an integrity hash and registry signatures but **no `attestations`**. The CI workflow is set up for provenance (`.github/workflows/version.yml:55`, `pnpm -r publish --provenance`), but releases are cut locally with `pnpm release`, and `scripts/release.mjs:6` says so directly: "CI publishes with npm provenance via OIDC; this script does not." Nothing cryptographically ties the npm tarball to this repository's source.
2. **Single maintainer.** Both `@runfusion/fusion` and the `runfusion.ai` alias package list exactly one npm maintainer, an individual account. If that account is compromised, every install is.
3. **Large dependency tree.** About 3,300 package entries in `pnpm-lock.yaml` (dev dependencies included). The published package has 32 runtime dependencies, including `electron`, `playwright-core`, `node-pty`, `dockerode`, `embedded-postgres`, several database drivers and the full pi agent SDK.
4. **Install scripts aren't restricted for end users.** `pnpm-workspace.yaml:8` (`onlyBuiltDependencies`) limits which packages may run install scripts, but only for contributors working in this workspace. An operator running `npm install -g @runfusion/fusion` runs every transitive install script.
5. **CI actions are mostly pinned by tag.** Most workflow `uses:` lines are pinned by tag (`actions/checkout@v7.0.1`, `changesets/action@v2`); only about 5 are pinned by commit SHA (for example in `threatcrush-scan.yml`). A moved tag could alter CI, including the provenance-publishing workflow if it is ever used.
6. **Additional trust surfaces.** `npx runfusion.ai` adds a second package name to trust (an exact-pinned alias, same maintainer; a mistyped name is a typosquat risk). The `curl -fsSL https://runfusion.ai/install.sh | sh` installer trusts that domain at install time.

### Good practices observed

- **Runtime binary downloads require a pinned SHA-256.** The external-integration manifest refuses assets while a tool is marked `upstream-pending-verification` (`packages/engine/src/external-integrations/manifest.ts:129`). Speech-to-text model downloads are hash-checked and refuse unpinned assets (`packages/dashboard/src/stt/model-manager.ts:131`, `:215`). The `cloudflared` install path refuses an unpinned manifest.
- **Dependabot** runs weekly for both npm and GitHub Actions (`.github/dependabot.yml`), and pull requests get a credential/vulnerable-pattern scan (`.github/workflows/threatcrush-scan.yml`).
- **The agent SDK is pinned exactly** (the `@earendil-works/*` closure at `1.1.0` in `pnpm-workspace.yaml`), enforced by `scripts/check-pi-versions-pinned.mjs`. Those upstream packages do carry SLSA provenance.
- **Auto-update is off by default** (`autoUpdateAndRestart: false`, `packages/core/src/config/settings-schema.ts:256`) and honours `updateCheckEnabled`. Keep it off: when on, it runs an unattended `npm install -g` every 6 hours (`packages/dashboard/src/auto-update.ts:34`), so a bad release would land with no human in the loop.

## 5. Agent execution and network exposure

- **Agents are unsandboxed by default.** `docs/sandbox.md:35`: "sandboxing is experimental, `backend` defaults to `native`." Agent `bash` tools run as the operator's OS user, so they can read anything that user can, including SSH keys, cloud CLI logins, registry tokens and git credentials. Isolating the user that runs Fusion is the most effective single mitigation.
- **Localhost binding plus bearer token by default.** Verified in code: `opts.host ?? "127.0.0.1"` in `packages/cli/src/commands/dashboard.ts:824`, `serve.ts:276` and `daemon.ts:264`. Auth is on unless `--no-auth` is passed (`docs/cli-reference.md`). Avoid `--host 0.0.0.0` and `--no-auth` together.
- **`/metrics` is unauthenticated by design** and its labels expose registered project ids and column names (`docs/diagnostics.md:255`). This matters only if the port is reachable beyond localhost.
- **Secrets under the `auto` policy reach the model.** A policy-allowed `fn_secret_get` delivers plaintext into the agent's model-visible transcript, which is persisted and sent to the provider (`docs/secrets.md:40`). Use `prompt` or `deny` for sensitive secrets.
- **No top-level `SECURITY.md`** or published vulnerability-disclosure policy was found.

## Recommendations for an operator

1. **Run Fusion in an isolated environment**: a dedicated VM, a container, or a dedicated OS user that holds none of your infrastructure credentials. This removes most of the agent-execution risk.
2. **Prefer building from a reviewed commit of this source** over installing from npm, because the npm release cannot be verified against the source. If you install from npm, pin an exact version.
3. Keep the defaults: `127.0.0.1` binding and bearer auth on. Keep `autoUpdateAndRestart` off.
4. Give integrations least-privilege credentials, such as a fine-grained GitHub token scoped to the repos Fusion manages.
5. Set sensitive secrets to `prompt` or `deny`.
6. Choose your AI provider with code privacy in mind: it sees everything the agents see.
7. If you use ntfy notifications, use a random topic or a self-hosted server.

## Open items

Not verified in this pass; do not treat as cleared:

- [ ] The default secret `access_policy` when a secret is created (UI and API).
- [ ] Whether the model-pricing refresh (`raw.githubusercontent.com`) can be disabled.
- [ ] Whether task worktrees run project dependency installs (`npm install`, `pip install`) automatically by default (`packages/engine/src/worktree/worktree-dependency-install.ts`). If so, agent-added dependencies run install scripts on the host.
- [ ] Remote-access tunnels (Cloudflare quick tunnels, Tailscale): what they expose and how they authenticate (`docs/remote-access.md`).
- [ ] The Docker image (`Dockerfile`, `docs/docker.md`) as an isolation option: base image, user, mounted credentials.
- [ ] Whether the published npm tarball matches a build of the tagged source.
- [ ] The sandbox backends (`bubblewrap`, `sandbox-exec`): how mature they are, and what fails open versus fails closed.
- [ ] A review of the dashboard API's authorization beyond the bearer-token middleware (for example the terminal routes and `--no-auth` paths).

## Reproducing the key checks

```bash
# No analytics/telemetry SDKs as direct dependencies
grep -hE '"(posthog[^"]*|@sentry/[^"]*|mixpanel[^"]*|@segment/[^"]*|amplitude[^"]*|@opentelemetry/[^"]*|@datadog/[^"]*)"' \
  packages/*/package.json plugins/*/package.json package.json

# Every external host hardcoded in non-test source
grep -rhoE "[\"'\`]https://[a-zA-Z0-9.-]+\.[a-z]{2,}" packages/*/src plugins/*/src --include=*.ts \
  | sort | uniq -c | sort -rn

# Provenance and maintainers of the published release
npm view @runfusion/fusion@latest version dist.attestations maintainers --json
npm view runfusion.ai version maintainers dependencies --json

# Inspect pi-telemetry without running install scripts (in an empty directory)
npm pack @earendil-works/pi-telemetry@1.1.0 && tar xzf earendil-works-pi-telemetry-1.1.0.tgz
grep -rnE "fetch\(|https?://|http\.request" package/dist/*.js
```
