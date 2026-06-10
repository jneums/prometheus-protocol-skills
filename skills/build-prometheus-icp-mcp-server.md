# Build a Prometheus Protocol MCP Server on ICP

End-to-end workflow: scaffold → implement tools → build → deploy via ICForge CI/CD → enable beacon → BYOC register → generate assets → GitHub release → QA via MCP client.

Uses `icp-cli` as the primary local toolchain and **ICForge** (<https://icforge.dev>) for automated mainnet CI/CD.

## Phase 0: Environment Setup

### Install icp-cli + ic-wasm

```bash
npm config set prefix /home/user/.npm-global
export PATH=/home/user/.npm-global/bin:$PATH
npm install -g @icp-sdk/icp-cli @icp-sdk/ic-wasm
icp --version
icp settings telemetry false
```

### Install mops

```bash
npm install -g ic-mops
```

### Install app-store-cli

```bash
npm install -g @prometheus-protocol/app-store-cli@latest
```

### Set PATH for all subsequent commands

```bash
export PATH=/home/user/.npm-global/bin:$PATH
```

### Identity setup

icp-cli has its own identity store. In headless/sandbox environments, use plaintext storage:

```bash
# New identity
icp identity new myid --storage plaintext

# Or import from existing dfx PEM
icp identity import --from-pem /home/user/.config/dfx/identity/default/identity.pem --storage plaintext myid

# Set as default
icp identity default myid
icp identity principal
```

### Legacy dfx (needed for update-settings and metadata queries)

```bash
export DFXVM_INIT_YES=1
sh -ci "$(curl -fsSL https://internetcomputer.org/install.sh)"
source /home/user/.local/share/dfx/env
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
.icp/cache/
```

Note: `.icp/data/mappings/` should NOT be gitignored — commit `ic.ids.json`.

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

### 4a. Initial canister creation (one-time, local)

```bash
icp deploy <canister-name> -e ic
```

This creates the canister and does the first install. Note the canister ID.

### 4b. Add ICForge as a controller

ICForge's principal: `hu6sm-q3tid-2vbh7-wy3qp-r776c-zakxe-iueaq-2bsxh-zkluq-5j6wi-kae`

icp-cli doesn't have `update-settings` yet, so use dfx:

```bash
export DFX_WARNING=-mainnet_plaintext_identity
dfx canister update-settings <canister-id> \
  --add-controller hu6sm-q3tid-2vbh7-wy3qp-r776c-zakxe-iueaq-2bsxh-zkluq-5j6wi-kae \
  --network ic
```

### 4c. Link repo on ICForge

1. Go to <https://icforge.dev>
2. Log in with GitHub
3. Connect the repo
4. ICForge will auto-detect `icp.yaml` and the canister

### 4d. Commit canister ID mapping

Ensure `.icp/data/mappings/ic.ids.json` is committed:

```json
{"<canister-name>": "<canister-id>"}
```

ICForge hydrates this from its DB, but having it in the repo ensures local deploys also work.

### 4e. Push to trigger builds

Every push to `main` triggers ICForge to:

1. Clone the repo
2. Build with `@dfinity/motoko@v4.0.0` recipe
3. Deploy (upgrade) to mainnet automatically

**No more manual `icp deploy -e ic` needed after this point.**

### 4f. Configure live mode (one-time, after first deploy)

```bash
icp canister call <canister-name> set_live_mode '(true)' -e ic
icp canister call <canister-name> set_registry_canister_id '(principal "grhdx-gqaaa-aaaai-q32va-cai")' -e ic
```

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

## Phase 6: Generate Assets & GitHub Release

### Icon guidelines — CRITICAL

- **No border radius / rounded corners** — the app store applies its own rounding
- **No white background** — corners must be dark, not white
- Image generators often produce icons with white backgrounds and rounded corners
- **Fix:** Scale the source image to **120%**, then center-crop back to original size — this pushes the white corners off the canvas

### Generate icon (512×512) and banner (1600×900)

Use image generation, then convert to webp with the 120% scale fix:

```python
from PIL import Image

# Icon — scale 120% to push white corners off canvas
img = Image.open('icon.png').convert('RGB')
w, h = img.size
scaled = img.resize((int(w * 1.2), int(h * 1.2)), Image.LANCZOS)
left = (scaled.width - w) // 2
top = (scaled.height - h) // 2
cropped = scaled.crop((left, top, left + w, top + h))
final = cropped.resize((512, 512), Image.LANCZOS)
final.save('icon-<app-name>.webp', 'WEBP', quality=90, method=6)

# Banner — same treatment if needed, or direct resize
img = Image.open('banner.png').convert('RGB')
img.thumbnail((1600, 900))
canvas = Image.new('RGB', (1600, 900), (10, 12, 28))
x = (1600 - img.width) // 2
y = (900 - img.height) // 2
canvas.paste(img, (x, y))
canvas.save('banner-<app-name>.webp', 'WEBP', quality=90, method=6)
```

### Verify corners are dark (not white)

```python
for name, box in [('TL',(0,0,5,5)),('TR',(507,0,512,5)),('BL',(0,507,5,512)),('BR',(507,507,512,512))]:
    corner = final.crop(box)
    pixels = list(corner.getdata())
    avg = tuple(sum(c)//len(pixels) for c in zip(*pixels))
    assert not all(c > 240 for c in avg), f'{name} corner is white!'
```

### Create GitHub release via API

```bash
GH_TOKEN="<pat>"  REPO="<owner>/<repo>"  TAG="v0.1.0-assets"

RELEASE=$(curl -s -X POST \
  -H "Authorization: Bearer $GH_TOKEN" -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/$REPO/releases \
  -d "{\"tag_name\":\"$TAG\",\"name\":\"$TAG\",\"body\":\"Assets\",\"draft\":false,\"prerelease\":false}")
RELEASE_ID=$(echo "$RELEASE" | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")

for ASSET in icon banner; do
  curl -s -X POST -H "Authorization: Bearer $GH_TOKEN" -H "Content-Type: image/webp" \
    --data-binary @assets/${ASSET}-<app-name>.webp \
    "https://uploads.github.com/repos/$REPO/releases/$RELEASE_ID/assets?name=${ASSET}-<app-name>.webp"
done
```

## Phase 7: prometheus.yml & BYOC Registration

### Full prometheus.yml template

```yaml
name: <app-name>
namespace: io.github.<owner>.<app-name>
title: <Title>
version: 0.1.0
description: Short description.
category: <Category>
mcp_path: /mcp
repo_url: https://github.com/<owner>/<repo>
payment: null
tags: [tag1, tag2]
submission:
  name: <Title>
  description: Short description.
  publisher: <Publisher>
  category: <Category>
  deployment_type: global
  repo_url: https://github.com/<owner>/<repo>
  mcp_path: /mcp
  why_this_app: Why agents or users should use this.
  key_features:
    - Feature one.
    - Feature two.
  tags: [tag1, tag2]
  visuals:
    icon_url: https://github.com/<owner>/<repo>/releases/download/<tag>/icon-<app-name>.webp
    banner_url: https://github.com/<owner>/<repo>/releases/download/<tag>/banner-<app-name>.webp
    gallery_images:
      - https://github.com/<owner>/<repo>/releases/download/<tag>/banner-<app-name>.webp
```

### Register & verify

```bash
app-store-cli byoc register <canister-id> <namespace>
icp canister call grhdx-gqaaa-aaaai-q32va-cai get_external_binding '("<namespace>")' -e ic
```

## Phase 8: QA via MCP Client

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
3. **Add ICForge controller BEFORE linking repo** — ICForge needs to be a controller to deploy. Use dfx since icp-cli lacks `update-settings`.
4. **ICForge may need multiple rebuild triggers** — Build dependency resolution can take several pushes. Use empty commits: `git commit --allow-empty -m "chore: trigger rebuild"`.
5. **Registry type drift** — Always pull latest .did. Missing variants like `#External` cause IDL traps.
6. **icp-cli identity in headless env** — Use `--storage plaintext`; keyring fails without X11/dbus.
7. **Commit `.icp/data/mappings/ic.ids.json`** — ICForge hydrates from its DB, but local deploys need this file.
8. **Keep dfx.json** — mops and local tooling still reference it.
9. **Beacon must be enabled** — Scaffold has it commented out. Uncomment before deploy.
10. **Don't derive MCP URL from namespace** — Use actual canister ID.
11. **Asset URLs must match exactly** — prometheus.yml visual URLs must match GitHub release URLs character-for-character.
12. **Icon white corners** — Image generators add white backgrounds with rounded corners. Fix by scaling source 120% then center-cropping back to original size. Always verify corners are dark (avg RGB < 240) before uploading.

## Complete Checklist

- [ ] Environment: icp-cli, mops, app-store-cli installed
- [ ] Identity set up with `--storage plaintext`
- [ ] Scaffolded with `create-motoko-mcp-server`
- [ ] `icp.yaml` uses `@dfinity/motoko@v4.0.0` recipe (NOT custom script)
- [ ] `icp.yaml` has `args: ""` in recipe configuration
- [ ] `dfx.json` kept for mops compatibility
- [ ] `.gitignore` includes `.icp/cache/` but NOT `.icp/data/`
- [ ] Tools implemented with adapter pattern (mock + live)
- [ ] Registry types match latest .did (including `#External`)
- [ ] Beacon enabled in `main.mo`
- [ ] `icp build` passes clean locally
- [ ] Tests pass locally
- [ ] Initial `icp deploy -e ic` succeeds
- [ ] ICForge principal added as controller via dfx
- [ ] Repo linked on <https://icforge.dev>
- [ ] `.icp/data/mappings/ic.ids.json` committed
- [ ] Push to main triggers successful ICForge build
- [ ] `set_live_mode(true)` called
- [ ] `set_registry_canister_id` called
- [ ] Logs show `Beacon submission timer started`
- [ ] Icon (512×512 webp) generated — **corners verified dark, not white**
- [ ] Banner (1600×900 webp) generated
- [ ] GitHub release created with assets uploaded
- [ ] `prometheus.yml` complete with submission block and matching visual URLs
- [ ] BYOC registered
- [ ] On-chain binding confirmed
- [ ] MCP client connected and all tools return live data
