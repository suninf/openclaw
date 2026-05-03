# AGENTS.MD

Telegraph style. Root rules only. Read scoped `AGENTS.md` before subtree work.

## Start

- Repo: `https://github.com/openclaw/openclaw`
- Replies: repo-root refs only: `extensions/telegram/src/index.ts:80`. No absolute paths, no `~/`.
- Run docs list first: `pnpm docs:list` if available; read relevant docs only.
- High-confidence answers only when fixing/triaging: verify source, shipped/current behavior, and dependency contracts before deciding.
- Dependency-backed behavior: read upstream dependency docs/source/types first. Do not assume APIs, defaults, errors, timing, or runtime behavior.
- Live-verify when feasible. Check env/`~/.profile` for keys; keep secret output redacted.
- Missing deps: `pnpm install`, retry once, then report first actionable error.
- CODEOWNERS: maint/refactor ok. Larger behavior/product/security/ownership: owner ask/review.
- Wording: product/docs/UI/changelog say "plugin/plugins"; `extensions/` is internal.
- New channel/plugin/app/doc surface: update `.github/labeler.yml` + GH labels.
- New `AGENTS.md`: add sibling `CLAUDE.md` symlink.

## Map

- Core TS: `src/`, `ui/`, `packages/`; plugins: `extensions/`; SDK: `src/plugin-sdk/*`; channels: `src/channels/*`; loader: `src/plugins/*`; protocol: `src/gateway/protocol/*`; docs/apps: `docs/`, `apps/`, `Swabble/`.
- Installers: sibling `../openclaw.ai`.
- Scoped guides exist in: `extensions/`, `src/{plugin-sdk,channels,plugins,gateway,gateway/protocol,agents}/`, `docs/`, `ui/`, `scripts/`.

## Architecture

- Core stays extension-agnostic. No bundled ids in core when manifest/registry/capability contracts work.
- Extensions cross into core only via `openclaw/plugin-sdk/*`, manifest metadata, injected runtime helpers, documented barrels (`api.ts`, `runtime-api.ts`).
- Extension prod code: no core `src/**`, `src/plugin-sdk-internal/**`, other extension `src/**`, or relative outside package.
- Core: no deep plugin internals (`extensions/*/src/**`, `onboard.js`). Use `api.ts`, SDK facade, generic contracts.
- Extension-owned behavior stays extension-owned: repair, detection, onboarding, auth/provider defaults, provider tools/settings.
- Owner boundary: fix owner-specific behavior in the owner module. Shared/core gets generic seams only; no owner ids, dependency strings, defaults, migrations, or recovery policy. If a bug names an extension or its dependency, start in that extension and add a generic core seam only when multiple owners need it.
- Legacy config repair: doctor/fix paths, not startup/load-time core migrations.
- New seams: backwards-compatible, documented, versioned. Third-party plugins exist.
- Channels: `src/channels/**` is implementation; plugin authors get SDK seams.
- Providers: core owns generic loop; provider plugins own auth/catalog/runtime hooks.
- Gateway protocol changes: additive first; incompatible needs versioning/docs/client follow-through.
- Config contract: exported types, schema/help, metadata, baselines, docs aligned. Retired public keys stay retired; compat in raw migration/doctor.
- Direction: manifest-first control plane; targeted runtime loaders; no hidden contract bypasses; broad mutable registries transitional.
- Prompt cache: deterministic ordering for maps/sets/registries/plugin lists/files/network results before model/tool payloads. Preserve old transcript bytes when possible.

## Commands

- Runtime: Node 22+. Keep Node + Bun paths working.
- Install: `pnpm install` (keep Bun lock/patches aligned if touched).
- CLI: `pnpm openclaw ...` or `pnpm dev`; build: `pnpm build`.
- Smart gate: `pnpm check:changed`; explain `pnpm changed:lanes --json`; staged preview `pnpm check:changed --staged`.
- Sparse worktrees: `pnpm check:changed` is sparse-safe and may skip sparse-missing typecheck projects; do not expand sparse checkout just to satisfy changed-gate tsgo. Direct `pnpm tsgo*` remains strict; use a fuller worktree when you need direct typecheck proof.
- Prod sweep: `pnpm check`.
- Typecheck: `tsgo` lanes only (`pnpm tsgo*`); do not add `tsc --noEmit`, `typecheck`, `check:types`.
- Formatting: use `oxfmt`, not Prettier. Prefer `pnpm format:check` / `pnpm format`; for targeted files use `pnpm exec oxfmt --check --threads=1 <files...>` or `pnpm exec oxfmt --write --threads=1 <files...>`.
- Linting: use repo wrappers (`pnpm lint:*`, `scripts/run-oxlint.mjs`); do not invoke generic JS formatters/lints unless a repo script uses them.
- Heavy checks: `OPENCLAW_LOCAL_CHECK=1`, mode `OPENCLAW_LOCAL_CHECK_MODE=throttled|full`; CI/shared use `OPENCLAW_LOCAL_CHECK=0`.
- Blacksmith/Testbox: on maintainer machines with Blacksmith access, broad/shared validation defaults to Testbox. This includes `pnpm check`, `pnpm check:changed`, Docker/live/package/build gates. Do not start those broad gates locally unless the user explicitly asks for local proof or sets `OPENCLAW_LOCAL_CHECK_MODE=throttled|full`.
- Local validation: targeted edit loops only — targeted formatter checks and small lint/type probes. If a local command expands beyond targeted proof, stop it and move the broad gate to Testbox.
- Testbox use: run from repo root, pre-warm early with `blacksmith testbox warmup ci-check-testbox.yml --ref main --idle-timeout 90`, reuse the returned `tbx_...` id for all `run`/`download` commands, and stop boxes you created before handoff. Timeout bins: `90` minutes default, `240` multi-hour, `720` all-day, `1440` overnight; anything above `1440` needs explicit approval and cleanup.

## GitHub / CI

- Triage: list first, hydrate few. Use bounded `gh --json --jq`; avoid repeated full comment scans.
- Automatic PR/issue discovery: skip maintainer-owned items unless directly relevant. Do not comment, close, label, retitle, rebase, fix up, or land them without Peter asking.
- PR scan/triage: no unsolicited PR comments/reviews. Report in chat only unless explicitly asked, or a close/duplicate action needs a reason comment.
- Search/dedupe: prefer `gh search issues 'repo:openclaw/openclaw is:open <terms>' --json number,title,state,updatedAt --limit 20`.
- GitHub search boolean text is fussy. If `OR` queries return empty, split exact terms and search title/body/comments separately before concluding no hits.
- PR shortlist: `gh pr list ...`; then `gh pr view <n> --json number,title,body,closingIssuesReferences,files,statusCheckRollup,reviewDecision`.
- After landing PR: search duplicate open issues/PRs. Before closing: comment why + canonical link.
- GH comments with markdown backticks, `$`, or shell snippets: avoid inline double-quoted `--body`; use single quotes or `--body-file`.
- PR execution artifacts/screenshots: attach them to the PR, comment, or an external artifact store. Do not add `.github/pr-assets` or other PR-only assets to the repo.
- PR review answer must explicitly cover: what bug/behavior we are trying to fix; PR/issue URL(s) and affected endpoint/surface; whether this is the best possible fix, with high-certainty evidence from code, CI, and shipped/current behavior.
- When working on an issue or PR, always end the user-facing final answer with the full GitHub URL.
- CI polling: exact SHA, needed fields only. Example: `gh api repos/<owner>/<repo>/actions/runs/<id> --jq '{status,conclusion,head_sha,updated_at,name,path}'`.
- Post-land wait: minimal. Exact landed SHA only. If superseded on `main`, same-branch `cancel-in-progress` cancellations are expected; stop once local touched-surface proof exists. Never wait for newer unrelated `main` unless asked.
- Wait matrix:
  - never: `Auto response`, `Labeler`, `Docs Sync Publish Repo`, `Docs Agent`, `Stale`.
  - conditional: `CI` exact SHA only; `Docs` only docs task/no local docs proof; `Workflow Sanity` only workflow/composite/CI-policy edits; `Plugin NPM Release` only plugin package/release metadata.
  - release/manual only: `Docker Release`, `OpenClaw NPM Release`, `macOS Release`, `OpenClaw Release Checks`, `Cross-OS Release Checks`.
  - explicit/surface only: `Install Smoke`, `CodeQL`, `Sandbox Common Smoke`, `Parity gate`, `Blacksmith Testbox`, `Control UI Locale Refresh`.
- `/landpr`: do not idle on `auto-response` or `check-docs`. Treat docs as local proof unless `check-docs` already failed with actionable relevant error.
- Poll 30-60s. Fetch jobs/logs/artifacts only after failure/completion or concrete need.

## Gates

- Pre-commit hook: staged formatting only. Validation explicit.
- Changed lanes:
  - core prod: core prod typecheck
  - extension prod: extension prod typecheck
  - public SDK/plugin contract: extension prod too
  - unknown root/config: all lanes
- Before handoff/push for code/runtime/config changes: run `pnpm check:changed` in Testbox by default on maintainer machines. Full prod sweep: run `pnpm check` in Testbox. Use local only for narrow targeted proof or when explicitly requested.
- If `pnpm check:changed` selects broad/shared lanes, it belongs in Testbox; do not let it continue locally after it fans out.
- Docs/changelog-only and CI/workflow metadata-only changes are not changed-gate work by default. Use `git diff --check` plus the relevant formatter/docs/workflow sanity check; escalate to `pnpm check:changed` only when scripts, generated docs/API, package metadata, or runtime/build behavior changed.
- Rebase sanity: after a green `pnpm check:changed`, a clean rebase onto current
  `origin/main` does not require rerunning the full changed gate when the rebase
  has no conflicts and the branch diff is materially unchanged. Do a quick
  `git status`, `git diff --check`, and diff/stat sanity check; rerun targeted or
  full checks only if conflict resolution, upstream overlap, generated drift,
  dependency/config changes, or touched-file content changes make the prior
  result stale.
- Landing on `main`: verify touched surface near landing. Default feasible bar: `pnpm check`.
- Hard build gate: `pnpm build` before push if build output, packaging, lazy/module boundaries, or published surfaces can change.
- Do not land related failing format/lint/type/build. If unrelated on latest `origin/main`, say so with scoped proof.
- Generated/API drift: `pnpm check:architecture`, `pnpm config:docs:gen/check`, `pnpm plugin-sdk:api:gen/check`. Track `docs/.generated/*.sha256`; full JSON ignored.

## Code

- TS ESM, strict. Avoid `any`; prefer real types, `unknown`, narrow adapters.
- No `@ts-nocheck`. Lint suppressions only intentional + explained.
- External boundaries: prefer `zod` or existing schema helpers.
- Runtime branching: discriminated unions/closed codes over freeform strings.
- Avoid semantic sentinels: `?? 0`, empty object/string, etc.
- Dynamic import: no static+dynamic import for same prod module. Use `*.runtime.ts` lazy boundary. After edits: `pnpm build`; check `[INEFFECTIVE_DYNAMIC_IMPORT]`.
- Cycles: keep `pnpm check:import-cycles` + architecture/madge green.
- Classes: no prototype mixins/mutations. Prefer inheritance/composition.
- Comments: brief, only non-obvious logic.
- Split files around ~700 LOC when clarity improves.
- Naming: **OpenClaw** product/docs; `openclaw` CLI/package/path/config.
- English: American spelling.

## Docs / Changelog

- Docs change with behavior/API. Use docs list/read_when hints; docs links per `docs/AGENTS.md`.
- Docs final answers: when doc files changed, end with the relevant full `https://docs.openclaw.ai/...` URL(s).
- Changelog user-facing only; pure internal usually no entry.
- Changelog placement: active version `### Changes`/`### Fixes`; every added entry must include at least one `Thanks @author` attribution, using credited GitHub username(s). Never add `Thanks @codex`, `Thanks @openclaw`, or `Thanks @steipete`.
- Changelog bullets are always single-line. No wrapping/continuation across multiple lines. Long entries stay on one long line so dedupe, PR-ref, and credit-audit tooling work and so the visual style stays uniform.

## Git

- Commit via `scripts/committer "<msg>" <file...>`; stage intended files only. It formats staged files; still run gates.
- Commits: conventional-ish, concise, grouped.
- No manual stash/autostash unless explicit. No branch/worktree changes unless requested.
- `main`: no merge commits; rebase on latest `origin/main` before push. Do not
  keep chasing `main` with repeated full gates after one green run plus a clean
  rebase sanity pass.
- User says `commit`: your changes only. `commit all`: all changes in grouped chunks. `push`: may `git pull --rebase` first.
- Do not delete/rename unexpected files; ask if blocking, else ignore.
- Bulk PR close/reopen >5: ask with count/scope.
- PR/issue workflows: `$openclaw-pr-maintainer`. `/landpr`: `~/.codex/prompts/landpr.md`.

## Security / Release

- Never commit real phone numbers, videos, credentials, live config.
- Secrets: channel/provider creds in `~/.openclaw/credentials/`; model auth profiles in `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`.
- Env keys: check `~/.profile`.
- Dependency patches/overrides/vendor changes need explicit approval. `pnpm.patchedDependencies` exact versions only.
- Carbon pins owner-only: do not change `@buape/carbon` unless Shadow (`@thewilloftheshadow`, verified by `gh`) asks.
- Releases/publish/version bumps need explicit approval. Release docs: `docs/reference/RELEASING.md`; use `$openclaw-release-maintainer`.
- GHSA/advisories: `$openclaw-ghsa-maintainer`.
- Beta tag/version match: `vYYYY.M.D-beta.N` -> npm `YYYY.M.D-beta.N --tag beta`.

## Apps / Platform

- "restart iOS/Android apps" = rebuild/reinstall/relaunch, not kill/launch.
- SwiftUI: Observation (`@Observable`, `@Bindable`) over new `ObservableObject`.
- Mac gateway: dev watch = `pnpm gateway:watch` (tmux `openclaw-gateway-watch-main`, auto-attach). Noninteractive: `OPENCLAW_GATEWAY_WATCH_ATTACH=0 pnpm gateway:watch`; attach/stop: `tmux attach -t openclaw-gateway-watch-main` / `tmux kill-session -t openclaw-gateway-watch-main`. Managed installs: `openclaw gateway restart/status --deep`. No launchd/ad-hoc tmux. Logs: `./scripts/clawlog.sh`.
- Version bump touches: `package.json`, `apps/android/app/build.gradle.kts`, `apps/ios/version.json` + `pnpm ios:version:sync`, macOS `Info.plist`, `docs/install/updating.md`. Appcast only for Sparkle release.
- Mobile LAN pairing: plaintext `ws://` loopback-only. Private-network `ws://` needs `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1`; Tailscale/public use `wss://` or tunnel.
- A2UI hash `src/canvas-host/a2ui/.bundle.hash`: generated; ignore unless running `pnpm canvas:a2ui:bundle`; commit separately.

## Ops / Footguns

- Remote install docs: `docs/install/{exe-dev,fly,hetzner}.md`. Parallels smoke: `$openclaw-parallels-smoke`; Discord roundtrip: `parallels-discord-roundtrip`.
- Memory wiki: keep prompt digest tiny. The prompt should only say the wiki exists, prefer `wiki_search` / `wiki_get`, start from `reports/person-agent-directory.md` for people routing, use search modes (`find-person`, `route-question`, `source-evidence`, `raw-claim`) when useful, and verify contact data before use.
- People wiki provenance: generated identity, social, contact, and "fun detail" notes need explicit source class/confidence (`maintainer-whois`, Discrawl sample/stat, GitHub profile, maintainer repo file). Do not promote inferred details to facts.
- Rebrand/migration/config warnings: run `openclaw doctor`.
- Never edit `node_modules`.
- Local-only `.agents` ignores: `.git/info/exclude`, not repo `.gitignore`.
- CLI progress: `src/cli/progress.ts`; status tables: `src/terminal/table.ts`.
- Connection/provider additions: update all UI surfaces + docs + status/config forms.
- Provider tool schemas: prefer flat string enum helpers over `Type.Union([Type.Literal(...)])`; some providers reject `anyOf`. Not a repo-wide protocol/schema ban.
- External messaging: no token-delta channel messages. Follow `docs/concepts/streaming.md`; preview/block streaming uses edits/chunks and preserves final/fallback delivery.
