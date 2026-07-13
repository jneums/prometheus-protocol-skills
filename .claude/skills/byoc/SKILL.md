---
name: byoc
description: Register an externally deployed ICP canister (bring-your-own-canister) with the Prometheus Protocol app store - prepare the prometheus.yml manifest, generate icon/banner assets, publish a GitHub release, link ICForge CI/CD, and run app-store-cli byoc register. Use when the user wants to register, publish, list, or update an existing canister or MCP server in the Prometheus app store, or update its app store metadata/visuals.
---

# BYOC — Bring Your Own Canister

Register an externally-deployed canister with the Prometheus app store for discovery. Includes manifest prep, asset generation, GitHub release, ICForge CI/CD linking, and BYOC registration.

## When to use

You have a canister deployed on ICP and want it discoverable through the Prometheus app store. The namespace must exist in the registry and you must be a controller.

## Working with the user

This skill is often driven by non-technical users. Follow these rules:

- **Gather everything upfront.** Ask for: app name, title, one-line description, publisher name, category, GitHub owner/repo, canister ID, and namespace. Confirm the namespace follows `io.github.<github-username>.<app-name>`.
- **Ask for the GitHub PAT only at the release step** (Step 3). It needs Contents: Read/Write on the repo. Keep it in an environment variable for the session — never write it into files, git config, git remote URLs, or commit history.
- **The only manual browser step** is logging into <https://icforge.dev> and connecting the repo (Step 4) — tell the user clearly when that moment arrives.
- **Report progress in plain language** and verify each step succeeded before moving on. If something fails, check the Errors and Pitfalls sections and retry before surfacing it to the user.

## Prerequisites

- A deployed mainnet canister (running, with a module hash)
- An identity that is a controller of the Prometheus namespace
- `app-store-cli` installed (latest version)
- A GitHub repo for the project
- A GitHub PAT with Contents: Read/Write on the repo

### Install tools

```bash
npm config set prefix "$HOME/.npm-global"
export PATH="$HOME/.npm-global/bin:$PATH"
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

Generate an icon (512×512) and banner (1600×900), convert to webp.

### Icon guidelines — CRITICAL

- **No border radius / rounded corners** — the app store applies its own rounding; shipping rounded icons creates a double-round effect
- **No white background** — corners must be dark, not white
- Image generators often produce icons with white backgrounds and rounded corners even when told not to. When prompting, explicitly specify: "No border radius, no rounded corners, fill the entire square. No white background, use a dark background."
- **Fix for white/rounded corners:** scale the source image to **120%**, then center-crop back to original size — this pushes the white corners off the canvas

```python
from PIL import Image

# Icon — scale 120% to push white corners off canvas, then center-crop
img = Image.open('icon.png').convert('RGB')
w, h = img.size
scaled = img.resize((int(w * 1.2), int(h * 1.2)), Image.LANCZOS)
left = (scaled.width - w) // 2
top = (scaled.height - h) // 2
cropped = scaled.crop((left, top, left + w, top + h))
final = cropped.resize((512, 512), Image.LANCZOS)
final.save('icon-<app-name>.webp', 'WEBP', quality=90, method=6)

# Banner — fit onto a dark 1600x900 canvas
img = Image.open('banner.png').convert('RGB')
img.thumbnail((1600, 900))
canvas = Image.new('RGB', (1600, 900), (10, 12, 28))
x = (1600 - img.width) // 2
y = (900 - img.height) // 2
canvas.paste(img, (x, y))
canvas.save('banner-<app-name>.webp', 'WEBP', quality=90, method=6)
```

### Verify icon corners are dark (not white) — always run this before uploading

```python
for name, box in [('TL',(0,0,5,5)),('TR',(507,0,512,5)),('BL',(0,507,5,512)),('BR',(507,507,512,512))]:
    corner = final.crop(box)
    pixels = list(corner.getdata())
    avg = tuple(sum(c)//len(pixels) for c in zip(*pixels))
    assert not all(c > 240 for c in avg), f'{name} corner is white!'
```

## Step 3: Push repo & create GitHub release

Ask the user for a GitHub PAT with Contents: Read/Write on the repo, and export it for this session only:

```bash
export GH_TOKEN="<pat>"   # session env var only — never in files or git config
```

**Never embed the token in the git remote URL** (`git remote set-url origin https://user:token@...`) — it persists in `.git/config` and can leak. Use an ephemeral credential helper instead:

```bash
git -c credential.helper='!f() { echo "username=x-access-token"; echo "password=$GH_TOKEN"; }; f' \
  push origin main
```

Then create the release and upload assets:

```bash
REPO="<owner>/<repo>"  TAG="v0.1.0-assets"
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

Ensure prometheus.yml visual URLs match the resulting download URLs exactly, character-for-character.

## Step 4: Link repo to ICForge for CI/CD

*(Skip this step if ICForge was already set up — e.g. you arrived here from the `build-prometheus-icp-mcp-server` skill.)*

### Add ICForge as a controller

ICForge principal: `hu6sm-q3tid-2vbh7-wy3qp-r776c-zakxe-iueaq-2bsxh-zkluq-5j6wi-kae`

```bash
# icp-cli doesn't have update-settings yet, use dfx
export DFX_WARNING=-mainnet_plaintext_identity
dfx canister update-settings <canister-id> \
  --add-controller hu6sm-q3tid-2vbh7-wy3qp-r776c-zakxe-iueaq-2bsxh-zkluq-5j6wi-kae \
  --network ic
```

**Do this BEFORE linking the repo**, or builds will fail with permission errors.

### Link on ICForge (user's manual step)

Ask the user to:

1. Go to <https://icforge.dev>
2. Log in with GitHub
3. Connect the repo

ICForge auto-detects `icp.yaml` and the canister. After linking, every push to `main` triggers automatic build + deploy to mainnet.

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

Then tell the user their app is live, with its listing at `https://prometheusprotocol.org/certificate/<namespace>` and MCP endpoint at `https://<canister-id>.icp0.io/mcp`.

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
6. **Icons: no border radius, no white background** — the app store applies its own rounding. Fix with the 120% scale + center-crop trick, and always run the corner-darkness check before uploading.
7. **ICForge may need multiple retriggers** — empty commits work fine.
8. **Never persist the GitHub PAT** — no tokens in remote URLs, files, or commits; env var + ephemeral credential helper only.

## Complete checklist

- [ ] Canister deployed and running on mainnet
- [ ] Identity is a controller of the namespace
- [ ] `app-store-cli` installed at latest version
- [ ] `prometheus.yml` has `namespace` and full `submission` block
- [ ] Icon (512×512 webp) generated — **corners verified dark, not white**
- [ ] Banner (1600×900 webp) generated
- [ ] Repo pushed to GitHub (token via env var, not in remote URL)
- [ ] GitHub release created with assets uploaded
- [ ] Visual URLs in prometheus.yml match release URLs
- [ ] ICForge principal added as controller
- [ ] Repo linked on <https://icforge.dev> (user's manual step)
- [ ] ICForge build succeeds on push to main
- [ ] `app-store-cli byoc register` succeeded
- [ ] On-chain `get_external_binding` confirms binding
