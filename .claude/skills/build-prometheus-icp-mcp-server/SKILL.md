---
name: build-prometheus-icp-mcp-server
description: Build and ship a Prometheus Protocol MCP server on the Internet Computer (ICP), end to end - scaffold with create-motoko-mcp-server, implement tools in Motoko, deploy to mainnet, wire up ICForge CI/CD, enable the beacon, and register in the Prometheus app store. Use when the user wants to create, build, deploy, or publish a new MCP server for Prometheus Protocol, ICP, or ICForge.
---

# Build a Prometheus Protocol MCP Server on ICP

End-to-end workflow: scaffold → implement tools → build → deploy via ICForge CI/CD → enable beacon → register in the app store → QA via MCP client.

Uses `icp-cli` as the primary local toolchain and **ICForge** (<https://icforge.dev>) for automated mainnet CI/CD.

**ICForge creates the canister and handles all mainnet deployment.** The user never touches cycles, ICP tokens, wallets, or canister creation themselves — ICForge manages cycles for them and bills usage against a prepaid account balance (pay-as-you-go / top-up model). New accounts need an initial **$10 top-up after signing up**; canister creation, deploys, and running costs draw from that balance. Never run `icp deploy -e ic` or any manual canister creation — if a step appears to require cycles or a funded ICP identity, you've gone off-script.

## Working with the user

This skill is often driven by non-technical users. Follow these rules:

- **Gather everything upfront.** Before starting, ask for: the app name, a one-line description, the GitHub owner/repo to use, and (when you reach the release step) a GitHub PAT. Don't drip-feed questions mid-workflow.
- **Be upfront about costs.** Tell the user at the start: ICForge is pay-as-you-go — they top up their account (starting with $10 at signup) and ICForge manages the cycles, billing usage against that balance. They never buy cycles or ICP tokens, never set up a wallet, never create a canister.
- **There is exactly one manual step** the user must do in a browser — tell them clearly when it arrives: sign up / log in at <https://icforge.dev> with GitHub, top up $10, and connect the repo (Phase 4b). Everything else you do for them.
- **Handle secrets carefully.** Ask for the GitHub PAT only when needed, keep it in an environment variable for the session, and never write it into files, git config, or commit history.
- **Report progress in plain language.** Say "your app is now live at …" — not raw command output. Use the checklist at the bottom to track and show progress.
- **If a step fails, retry/diagnose yourself first** using the Pitfalls section before surfacing anything to the user.

## Phase 0: Environment Setup

### Install icp-cli + ic-wasm

```bash
npm config set prefix "$HOME/.npm-global"
export PATH="$HOME/.npm-global/bin:$PATH"
npm install -g @icp-sdk/icp-cli @icp-sdk/ic-wasm
icp --version
icp settings telemetry false
```

### Install mops and app-store-cli

```bash
npm install -g ic-mops @prometheus-protocol/app-store-cli@latest
```

### Set PATH for all subsequent commands

Every new shell needs this (or add it to the shell profile):

```bash
export PATH="$HOME/.npm-global/bin:$PATH"
```

### Identity setup

An identity is needed for signing calls (canister config, BYOC registration) — it never needs cycles or funding. icp-cli has its own identity store. In headless/sandbox environments, use plaintext storage (the OS keyring fails without X11/dbus):

```bash
# New identity
icp identity new myid --storage plaintext

# Or import from existing dfx PEM
icp identity import --from-pem "$HOME/.config/dfx/identity/default/identity.pem" --storage plaintext myid

# Set as default
icp identity default myid
icp identity principal
```

### Legacy dfx (optional — only for registry metadata queries in Phase 2)

```bash
export DFXVM_INIT_YES=1
sh -ci "$(curl -fsSL https://internetcomputer.org/install.sh)"
source "$HOME/.local/share/dfx/env"
```

## Phase 1: Scaffold

```bash
npx create-motoko-mcp-server <app-name>
cd <app-name>
npm install
mops install --lock ignore --no-toolchain
```

This generates a working skeleton with:

- `src/main.mo` — actor with MCP SDK wiring, auth/beacon blocks (commented out)
- `dfx.json` — legacy canister config (keep for mops compatibility)
- `mops.toml` — dependencies including `mcp-motoko-sdk`
- `prometheus.yml` — app store manifest template
- `test/` — test harness

### Add icp.yaml — MUST use the Motoko recipe

```yaml
canisters:
  - name: <app-name>
    recipe:
      type: "@dfinity/motoko@v4.0.0"
      configuration:
        main: src/main.mo
        args: ""

environments:
  - name: ic
    network: ic
    canisters: [<app-name>]
```

**CRITICAL:** Use `@dfinity/motoko@v4.0.0` recipe, NOT a custom build script. Custom scripts that call `$(mops toolchain bin moc)` will fail in ICForge because:

1. `mops toolchain init` is not run → "Toolchain management is not initialized"
2. The mops JS process OOMs in ICForge's constrained environment
3. `mops sources` never runs → `moc` can't find packages

The Motoko recipe handles `moc` resolution and package sources automatically.

Keep `dfx.json` alongside — `mops sources` and local tooling still reference it.

### Key dependencies (in mops.toml)

```toml
[dependencies]
mcp-motoko-sdk = "2.0.2"
json = "1.4.0"
map = "9.0.1"
http-types = "1.0.1"
datetime = "1.1.0"
base = "0.16.0"

[toolchain]
moc = "0.16.0"
```

### .gitignore

```gitignore
.mops
.dfx
.env
node_modules
out/
.icp/
```

## Phase 2: Implement Tools

### Architecture: Adapter pattern

```text
src/
  main.mo                    # Actor, MCP SDK wiring, config setters
  AppStoreAdapter.mo         # Adapter interface (type definition)
  MockAppStoreAdapter.mo     # Mock data for local dev/tests
  LiveRegistryAdapter.mo     # Live reads from mcp_registry canister
  McpRegistry.mo             # Actor binding for mcp_registry
  RegistryTypes.mo           # Candid types matching registry .did
  tools/
    ToolContext.mo           # Shared context passed to all tools
    search_apps.mo           # Tool: config() + handle()
    ...
```

### Adapter resolution at call time (not startup)

```motoko
func resolveAdapter() : AppStoreAdapter.AppStoreAdapter {
  if (liveMode) {
    LiveRegistryAdapter.create(Principal.fromText(registryCanisterIdText))
  } else {
    MockAppStoreAdapter.create()
  }
};
```

### Tool module pattern

Each tool exports `config()` and `handle()`:

```motoko
public func config() : McpTypes.Tool { ... };
public func handle(ctx : ToolContext) : McpTypes.ToolHandler { ... };
```

### Registry bindings (RegistryTypes.mo)

Pull the latest .did from the live canister:

```bash
dfx canister --network ic metadata grhdx-gqaaa-aaaai-q32va-cai candid:service > registry.did
```

**Critical types to include:**

- `AppListingStatus` — must include `#External` variant (for BYOC apps)
- `Bounty` with `ClaimRecord` (not `[ICRC16]`)
- All nested types: `AppListing`, `AppDetailsResponse`, `AppVersionDetails`, `BuildInfo`, `DataSafetyInfo`, `AuditRecord`, `AttestationRecord`, recursive `ICRC16`

**Common trap:** Missing variants cause IDL traps at runtime.

### Pagination for registry listings

`get_app_listings` uses cursor pagination (`prev`/`take`). Fetch all pages in a while loop.

### Timestamp formatting

```motoko
import DateTime "mo:datetime/DateTime";
func formatTimestampNanos(nanos : Nat) : Text { DateTime.DateTime(nanos).toText() };
```

### URL derivation rules

- **MCP URL:** `https://<canister-id>.icp0.io/mcp` — from actual canister ID, NOT namespace
- **Certificate URL:** `https://prometheusprotocol.org/certificate/<namespace>`

### Schema hardening

Arrays must include `items` with type. Bounded arrays need `minItems`/`maxItems`.

## Phase 3: Build & Test Locally

```bash
icp build <canister-name>
npm test
```

No local replica needed for builds. For tests needing a replica:

```bash
icp network start -d
icp deploy
npm test
icp network stop
```

## Phase 4: Deploy to Mainnet via ICForge

**ICForge creates the canister on its own** the first time it builds the repo. Do NOT create a canister locally, do NOT run `icp deploy -e ic`, and do NOT touch cycles or wallets. The flow is: push the repo → link it on ICForge → ICForge creates the canister and deploys.

### 4a. Push the repo to GitHub

`icp.yaml` (with the `@dfinity/motoko@v4.0.0` recipe) must be on `main` before linking — ICForge reads it to know what to build.

### 4b. Sign up, top up, and link repo on ICForge (user's manual step)

Ask the user to:

1. Go to <https://icforge.dev>
2. Sign up / log in with GitHub
3. **Top up their ICForge account** ($10 initial top-up for new accounts — pay-as-you-go: canister creation, deploys, and running costs draw from this balance)
4. Connect the repo

ICForge auto-detects `icp.yaml`, **creates the mainnet canister itself** (paid from the account credit), and runs the first build + deploy. This is the only step you cannot do for them.

When you need the canister ID (MCP URL, config calls, registration), read it from the ICForge dashboard or build output.

### 4c. Pushes auto-deploy from here on

Every push to `main` triggers ICForge to:

1. Clone the repo
2. Build with `@dfinity/motoko@v4.0.0` recipe
3. Deploy (upgrade) to mainnet automatically

### 4d. Configure live mode (one-time, after first deploy)

```bash
icp canister call <canister-name> set_live_mode '(true)' -e ic
icp canister call <canister-name> set_registry_canister_id '(principal "grhdx-gqaaa-aaaai-q32va-cai")' -e ic
```

If the canister name doesn't resolve locally, pass the canister ID (from the ICForge dashboard) directly instead of the name.

These are ordinary signed calls — they cost the caller nothing (canisters pay for their own execution on ICP), so no cycles are needed here either.

## Phase 5: Enable Beacon

In `src/main.mo`, uncomment/enable the beacon block:

```motoko
let beaconCanisterId = Principal.fromText("m63pw-fqaaa-aaaai-q33pa-cai");
transient let beaconContext : ?Beacon.BeaconContext = ?Beacon.init(
    beaconCanisterId,
    ?(15 * 60), // every 15 minutes
);
```

Push to main — ICForge will build and deploy. Verify in logs:

```bash
icp canister logs <canister-name> -e ic
```

Should show: `Beacon submission timer started.`

## Phase 6: Register in the App Store

Follow the **`byoc` skill** (invoke it, or read `.claude/skills/byoc/SKILL.md`) for this entire phase — it owns asset generation (icon/banner), the GitHub release, the `prometheus.yml` manifest, and BYOC registration.

Notes specific to this workflow:

- `prometheus.yml` was already scaffolded in Phase 1 — fill it in rather than creating from scratch.
- ICForge created the canister and already controls it, and the repo is already linked — skip the byoc skill's "Add ICForge as a controller" and "Link on ICForge" steps entirely (they exist only for canisters created outside ICForge).
- Namespace convention: `io.github.<owner>.<app-name>`.

## Phase 7: QA via MCP Client

Connect to `https://<canister-id>.icp0.io/mcp` and run all tools sequentially.
Look for `"freshness":"live-mainnet"` in responses.

## Key Facts & Constants

| Item | Value |
| --- | --- |
| mcp_registry canister | `grhdx-gqaaa-aaaai-q32va-cai` |
| Beacon canister | `m63pw-fqaaa-aaaai-q33pa-cai` |
| ICForge principal | `hu6sm-q3tid-2vbh7-wy3qp-r776c-zakxe-iueaq-2bsxh-zkluq-5j6wi-kae` |
| ICForge URL | `https://icforge.dev` |
| Namespace convention | `io.github.<owner>.<app-name>` |
| MCP URL format | `https://<canister-id>.icp0.io/mcp` |
| Certificate URL format | `https://prometheusprotocol.org/certificate/<namespace>` |
| Scaffold command | `npx create-motoko-mcp-server <name>` |
| Motoko recipe | `@dfinity/motoko@v4.0.0` |

## Pitfalls & Lessons Learned

1. **MUST use Motoko recipe in icp.yaml** — Custom build scripts using `$(mops toolchain bin moc)` OOM in ICForge. The `@dfinity/motoko@v4.0.0` recipe handles everything automatically.
2. **Recipe requires `args` field** — Even if empty, set `args: ""` or the recipe template fails with "Failed to access variable in strict mode".
3. **Never create the canister locally** — no `icp deploy -e ic`, no cycles, no wallet. Link the repo and let ICForge create + deploy the canister. (Adding ICForge as a controller manually is only for pre-existing canisters — see the byoc skill.)
4. **ICForge needs account credit** — pay-as-you-go: new accounts top up $10 after signup, and ICForge bills managed cycles usage against the balance. Without credit, canister creation/deploys won't go through. Don't tell users the workflow is free.
5. **ICForge may need multiple rebuild triggers** — Build dependency resolution can take several pushes. Use empty commits: `git commit --allow-empty -m "chore: trigger rebuild"`.
6. **Registry type drift** — Always pull latest .did. Missing variants like `#External` cause IDL traps.
7. **icp-cli identity in headless env** — Use `--storage plaintext`; keyring fails without X11/dbus.
8. **Keep dfx.json** — mops and local tooling still reference it.
9. **Beacon must be enabled** — Scaffold has it commented out. Uncomment before deploy.
10. **Don't derive MCP URL from namespace** — Use actual canister ID (from the ICForge dashboard).
11. **Asset/registration pitfalls** — see the `byoc` skill's Pitfalls section (icon corners, URL matching, manifest requirements).

## Complete Checklist

- [ ] Environment: icp-cli, mops, app-store-cli installed
- [ ] Identity set up with `--storage plaintext`
- [ ] Scaffolded with `create-motoko-mcp-server`
- [ ] `icp.yaml` uses `@dfinity/motoko@v4.0.0` recipe (NOT custom script)
- [ ] `icp.yaml` has `args: ""` in recipe configuration
- [ ] `dfx.json` kept for mops compatibility
- [ ] `.gitignore` includes `.icp/`
- [ ] Tools implemented with adapter pattern (mock + live)
- [ ] Registry types match latest .did (including `#External`)
- [ ] Beacon enabled in `main.mo`
- [ ] `icp build` passes clean locally
- [ ] Tests pass locally
- [ ] Repo pushed to GitHub with `icp.yaml` on `main`
- [ ] ICForge account created and topped up ($10 initial credit, user's manual step)
- [ ] Repo linked on <https://icforge.dev> — ICForge creates the canister and runs the first deploy
- [ ] Canister ID noted from the ICForge dashboard
- [ ] Subsequent pushes to main trigger successful ICForge builds
- [ ] `set_live_mode(true)` called
- [ ] `set_registry_canister_id` called
- [ ] Logs show `Beacon submission timer started`
- [ ] App store registration complete (full checklist in the `byoc` skill)
- [ ] MCP client connected and all tools return live data
