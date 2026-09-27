# OpenCode Run-Scoped MCP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `opencode_local` receive only the assigned Paperclip MCP gateway(s) for each heartbeat run.

**Architecture:** Heartbeat already passes `runtimeMcp` to the adapter. The adapter reads its immutable `getServers()` snapshot and passes it into the temporary OpenCode XDG runtime config builder, which merges remote MCP entries with existing user entries and injects run-bound bearer headers. No persistent OpenCode or Paperclip config is modified. Remote execution transfers this temporary config through its existing runtime assets mechanism, and must not leave credentials in a persistent remote directory.

**Tech Stack:** TypeScript, Node, Vitest, pnpm, OpenCode JSON configuration.

**Spec:** `docs/superpowers/specs/2026-09-27-opencode-runtime-mcp.md`

## Global Constraints

- Preserve scoped repo change and existing local/provider configuration behavior; do not modify the live checkout directly.
- Do not print, check in, or persist gateway tokens in prompts, reports, or logs. Run-scoped tokens are revoked at run end.
- Do not delete old Paperclip data or create a backup; deployment is a distinct verification gate, with one writer only.

---

### Task 1: Runtime-config MCP delivery

**Files:** Modify `packages/adapters/opencode-local/src/server/runtime-config.ts`; test `packages/adapters/opencode-local/src/server/runtime-config.test.ts`.

**Interfaces:** `prepareOpenCodeRuntimeConfig(input)` gains optional `runtimeMcpServers?: AdapterRuntimeMcpServer[]` (type in `@paperclipai/adapter-utils`); keeps `{ env, notes, cleanup }` result. Server fields are `name,url,token,connectionId`.

- [ ] Add a failing test with an existing MCP entry and a stale same-name entry plus assigned `paperclip-assigned`: assert unrelated entry preserved, assigned entry overrides collision with `type:'remote'`, `enabled:true`, `oauth:false`, URL and Bearer header. Confirm test fails because assigned entry absent.
- [ ] Add a failing permission-opt-out test with assigned server and config `{dangerouslySkipPermissions:false}`; assert existing permission is retained while run MCP is present. Confirm failure because helper currently returns without config.
- [ ] Implement minimal config merge. Only short-circuit on disabled permission when there are no assigned servers; otherwise preserve permission. Avoid token-bearing notes and set private temporary directory/config mode (`0700`/`0600`) rather than relying solely on umask. Keep cleanup in `finally` at call site.
- [ ] Run `pnpm exec vitest run packages/adapters/opencode-local/src/server/runtime-config.test.ts` and `pnpm --filter @paperclipai/adapter-opencode-local typecheck`; expected exit 0. Test cleanup removes temp path.

### Task 2: Adapter handoff and documentation

**Files:** Modify `packages/adapters/opencode-local/src/server/execute.ts`, `packages/adapters/opencode-local/src/index.ts`; test an existing adapter execution test if feasible.

**Interfaces:** Pass `ctx.runtimeMcp?.getServers() ?? []` into `prepareOpenCodeRuntimeConfig`. `ctx.runtimeTools` continues to deliver connection-discovery tools separately.

- [ ] Write failing handoff test or a focused adapter invocation characterization asserting per-run server list reaches temporary config, without embedding a real token; confirm expected RED.
- [ ] Wire adapter's run-scoped MCP list. Ensure `localRuntimeConfigHome` triggers asset sync when a config is generated, including permissions-off mode. Check remote runtime lifecycle and cleanup for copied credential file; if remote cleanup cannot be guaranteed, fail closed or use an existing cleanup hook rather than leave a bearer header on persistent storage.
- [ ] Update adapter docs to distinguish assigned gateway tools from connection-discovery tools and note remote URL reachability.
- [ ] Run relevant adapter tests, typecheck, and `git diff --check`; commit scoped changes and report real outputs.

### Task 3: Independent acceptance and fresh-instance proof

**Files:** No unrelated repo changes; deployment docs only if actual commands/outcomes warrant updates.

- [ ] Janet inspects commit/diff and repeats focused tests and typecheck independently; confirm no gateway secret in tracked files and no unexpected config change.
- [ ] Build and push image using established K3s/registry procedure, then roll the fresh Deployment by image digest, without starting old Compose or a second writer. Verify rollout, private health, and HTTPS health; record digest and rollback handle.
- [ ] Trigger a bounded Paperclip `opencode_local` heartbeat with the assigned KB tools: verify OpenCode sees the assigned article tool, reads one harmless article and that token is revoked at run end; report if first-use signup/authorization or network reachability prevents this. Do not represent unit tests as end-to-end proof.
