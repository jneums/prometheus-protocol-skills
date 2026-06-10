# BYOC — Bring Your Own Canister

Register an externally-deployed canister with the Prometheus app store for discovery. Includes manifest prep, asset generation, GitHub release, ICForge CI/CD linking, and BYOC registration.

## When to use

You have a canister deployed on ICP and want it discoverable through the Prometheus app store. The namespace must exist in the registry and you must be a controller.

## Prerequisites

- A deployed mainnet canister (running, with a module hash)
- An identity that is a controller of the Prometheus namespace
- `app-store-cli` installed (latest version)
- A GitHub repo for the project
- A GitHub PAT with Contents: Read/Write on the repo

### Install tools

```bash
npm config set prefix /home/user/.npm-global
export PATH=/home/user/.npm-global/bin:$PATH
npm install -g @icp-sdk/icp-cli @prometheus-protocol/app-store-cli@latest
icp settings telemetry false
```

## Step 1: Prepare prometheus.yml

Full required structure:

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
  why_this_app: Why agents or users should use this app.
  key_features:
    - Feature one.
    - Feature two.
    - Feature three.
  tags: [tag1, tag2]
  visuals:
    icon_url: https://github.com/<owner>/<repo>/releases/download/<tag>/icon-<app-name>.webp
    banner_url: https://github.com/<owner>/<repo>/releases/download/<tag>/banner-<app-name>.webp
    gallery_images:
      - https://github.com/<owner>/<repo>/releases/download/<tag>/banner-<app-name>.webp
```

**Required fields:** `namespace`, `submission.name`, `submission.description`, `submission.repo_url`

**Namespace convention:** `io.github.<github-username>.<app-name>`

## Step 2: Generate visual assets

Generate icon (512×512) and banner (1600×900), convert to webp.

**IMPORTANT icon guidelines:**
- **No border radius** — icons should fill the full square canvas, no rounded corners
- **No white background** — use a dark or transparent background that blends with the app store
- The app store UI applies its own rounding/masking, so shipping rounded icons creates a double-round effect

```python
from PIL import Image
for src, dst, size in [
    ('icon.png', 'icon-<app-name>.webp', (512, 512)),
    ('banner.png', 'banner-<app-name>.webp', (1600, 900)),
]:
    img = Image.open(src).convert('RGB')
    img.thumbnail(size)
    canvas = Image.new('RGB', size, (10, 12, 28))
    x = (size[0] - img.width) // 2
    y = (size[1] - img.height) // 2
    canvas.paste(img, (x, y))
    canvas.save(dst, 'WEBP', quality=90, method=6)
```

When generating icons with AI image generation, explicitly specify:
- "No border radius, no rounded corners, fill the entire square"
- "No white background, use a dark background"

## Step 3: Push repo & create GitHub release

```bash
git remote set-url origin https://<user>:<token>@github.com/<owner>/<repo>.git
git add . && git commit -m "Initial import with assets" && git push origin main
```

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

Ensure prometheus.yml visual URLs match the resulting download URLs exactly.

## Step 4: Link repo to ICForge for CI/CD

### Add ICForge as a controller

ICForge principal: `hu6sm-q3tid-2vbh7-wy3qp-r776c-zakxe-iueaq-2bsxh-zkluq-5j6wi-kae`

```bash
# icp-cli doesn't have update-settings yet, use dfx
export DFX_WARNING=-mainnet_plaintext_identity
dfx canister update-settings <canister-id> \
  --add-controller hu6sm-q3tid-2vbh7-wy3qp-r776c-zakxe-iueaq-2bsxh-zkluq-5j6wi-kae \
  --network ic
```

### Link on ICForge

1. Go to https://icforge.dev
2. Log in with GitHub
3. Connect the repo
4. ICForge auto-detects `icp.yaml` and the canister

After linking, every push to `main` triggers automatic build + deploy to mainnet.

**Note:** ICForge build dependency resolution may take several pushes initially. Use empty commits to retrigger:

```bash
git commit --allow-empty -m "chore: trigger rebuild"
```

## Step 5: Register with BYOC

```bash
app-store-cli byoc register <canister-id> <namespace>
```

### Re-register (to update metadata)

```bash
app-store-cli byoc register <canister-id> <namespace>
```

Re-running register re-reads the manifest and pushes updated metadata on-chain. No need to unregister first.

## Step 6: Verify registration

```bash
# Via dfx (ground truth)
dfx canister call grhdx-gqaaa-aaaai-q32va-cai get_external_binding '("<namespace>")' --network ic
```

## Errors

| Error | Meaning |
| --- | --- |
| `NotController` | Caller is not a controller of the namespace |
| `NamespaceNotFound` | Namespace doesn't exist — needs to be created first |
| `AlreadyBound` | Namespace already has an external canister |
| `CanisterAlreadyBound` | Canister is already bound to another namespace |

## Pitfalls

1. **Always use latest app-store-cli** — older versions have bugs.
2. **Manifest must include `submission` block** — CLI validates name, description, repo_url.
3. **Namespace must already exist** — BYOC cannot create namespaces.
4. **Add ICForge controller BEFORE linking repo** — or builds will fail with permission errors.
5. **Asset URLs must match exactly** — character-for-character with GitHub release download URLs.
6. **Icons: no border radius, no white background** — the app store applies its own rounding. Shipping rounded icons creates a double-round effect.
7. **ICForge may need multiple retriggers** — empty commits work fine.

## Complete checklist

- [ ] Canister deployed and running on mainnet
- [ ] Identity is a controller of the namespace
- [ ] `app-store-cli` installed at latest version
- [ ] `prometheus.yml` has `namespace` and full `submission` block
- [ ] Icon (512×512 webp, no border radius, no white bg) and banner (1600×900 webp) generated
- [ ] Repo pushed to GitHub
- [ ] GitHub release created with assets uploaded
- [ ] Visual URLs in prometheus.yml match release URLs
- [ ] ICForge principal added as controller
- [ ] Repo linked on https://icforge.dev
- [ ] ICForge build succeeds on push to main
- [ ] `app-store-cli byoc register` succeeded
- [ ] On-chain `get_external_binding` confirms binding
