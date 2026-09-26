# Paperclip live port — task report (recovery run)

Date: 2026-09-26
Branch/worktree: `live` at `/home/joelai/code/agent-worktrees/paperclip/live`
Base SHA: `d3e0f0a238ed66a05d50ab6f627eb55874f0f57e` (fork master)
Head SHA: see "Commit" below.

## Context

Forward-port of two local customizations onto fork master `d3e0f0a`:

1. `a29ee7e` — onboarding step 4 pinned to the server-configured OpenCode
   provider (`opencode_local`), first server-declared model, no hosted
   sign-in.
2. `6c761dc` — editable agent Capabilities field in `AgentConfigForm` +
   patch tests.

A prior run was killed mid-`pnpm test:run`; its six modified UI files were
already present in this worktree and were NOT discarded. This recovery run
finished the work narrowly (no full suite).

## Changed files

- `ui/src/components/OnboardingWizard.tsx` — ported a29ee7e onto the new
  wizard shape: `connectSources` row, step-4 auto-pin effect (now guarded so
  a prior error, e.g. from a failed hire, is not overwritten), first
  server-declared model effect, OpenCode step-4 panel, non-OpenCode paths
  (login card, api-key card, env embedding) explicitly excluded for
  `opencode_local` only.
- `ui/src/components/OnboardingWizard.test.tsx` — restore-gate test moved
  from `opencode_local` to `gemini_local` stand-in (OpenCode is now pinned,
  so it cannot stand in); new test asserting the pinned OpenCode step hires
  with the first server-declared model and creates no managed connection.
- `ui/src/components/OnboardingWizard.step.test.tsx` — step-arrival test
  updated for the pinned OpenCode step (no tile row, first declared model in
  the hire payload).
- `ui/src/components/AgentConfigForm.tsx` — editable Capabilities
  `DraftTextarea` in the identity section (null on clear).
- `ui/src/components/AgentConfigForm.render.test.tsx` — two tests: capabilities
  edit flows into the save patch; clearing sends an explicit `null`.
- `ui/src/lib/agent-config-patch.test.ts` — patch test for capabilities
  edit / untouched / cleared.

## Recovery-run edits (this run)

- `ui/src/components/OnboardingWizard.test.tsx`: fixed two TS errors in the
  prior run's new test — `buildAdapterConfig.mockImplementation` signature
  (mock is `() => Record<string, unknown>`), and hire-payload tuple access
  now uses the file's established `mock.calls.at(-1) as unknown[]` pattern.

## Commands and results (recovery run)

- `pnpm exec vitest run ui/src/components/AgentConfigForm.render.test.tsx
  ui/src/components/OnboardingWizard.step.test.tsx
  ui/src/components/OnboardingWizard.test.tsx
  ui/src/lib/agent-config-patch.test.ts --maxWorkers=1`
  → 4 files, 210 tests, all passed (59.6s). No full suite run (per brief).
- `pnpm check:token-gates` → all gates clean.
- `pnpm --filter @paperclipai/ui exec tsc --noEmit` → clean (after the two
  test-file fixes above).
- `pnpm --filter @paperclipai/ui run build` → built (pre-existing
  chunk-size warnings only).
- Re-run of `ui/src/components/OnboardingWizard.test.tsx` after the fixes →
  94 tests passed.

## Critical review (per recovery brief)

Step 4 pins to `opencode_local` whenever the adapter is present in the
display registry (`moreAdapters`, non-`comingSoon`), regardless of whether
the server actually has `PAPERCLIP_OPENCODE_PROVIDERS` / model declarations
configured. Concretely:

- On a deployment with no configured OpenCode provider, step 4 would pin to
  OpenCode and dead-end with the rendered errors ("No OpenCode model
  configured. Check PAPERCLIP_ADAPTER_MODELS on the server.") instead of
  offering the recommended tiles.
- A customer cannot intentionally choose another adapter on step 4; the pin
  effect overrides any selection back to `opencode_local`.

This matches the behavior of the original commit a29ee7e (it force-pinned
too) and is the intent for THIS deployment, which has OpenCode configured.
Preserving non-OpenCode behavior elsewhere was verified: every pin/credential
branch is gated on `adapterType === "opencode_local"` and the restore-gate
test proves a saved non-OpenCode draft still resolves against the visible
row. Making the pin conditional on actual server configuration was NOT done:
the UI has no observable signal for it — `/companies/:id/adapters/:type/models`
falls back to built-in default models when unconfigured, and
`AdapterCapabilities` carries no configuration flag. Detecting it would need
a server-side signal (outside the UI-only scope) or a product decision, so
per the brief this is reported rather than invented. If the fork is ever
deployed without OpenCode configured, this pin must be revisited.

## Unresolved concerns

- The one above (forced pin on unconfigured deployments) is the only open
  product-level concern.
- `pnpm -r typecheck`, `pnpm test:run`, and `pnpm build` were intentionally
  not run (recovery brief forbids full suites); UI typecheck/build and the
  four targeted test files cover the changed surface.
- No push; no production operations. Janet to verify independently.

## Commit

See `git log -1` on branch `live`. Message:
`fix(ui): port OpenCode onboarding pin and agent capabilities field to new master`
with `Co-Authored-By: Paperclip <noreply@paperclip.ing>` trailer.