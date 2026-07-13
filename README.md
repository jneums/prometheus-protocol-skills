# Prometheus Protocol Skills

Agent skills for building MCP servers for Prometheus Protocol and hosting/deploying them with ICForge.

Each skill lives in its own directory under `skills/` as a `SKILL.md` with YAML frontmatter, following the [Claude Code skills format](https://code.claude.com/docs/en/skills) — so Claude discovers and invokes them automatically when the task matches.

## Included skills

- [`skills/build-prometheus-icp-mcp-server/SKILL.md`](skills/build-prometheus-icp-mcp-server/SKILL.md)
  - End-to-end workflow for scaffolding, implementing, building, deploying, configuring, registering, and QA'ing a Prometheus Protocol MCP server on ICP.
- [`skills/byoc/SKILL.md`](skills/byoc/SKILL.md)
  - Bring-your-own-canister registration workflow for publishing an externally deployed canister to Prometheus Protocol and wiring ICForge CI/CD. Also owns asset generation, the GitHub release, and the `prometheus.yml` manifest — the build skill delegates its registration phase here.

## Installation

Copy the skill directories into your project's `.claude/skills/` (or `~/.claude/skills/` to make them available everywhere):

```bash
mkdir -p .claude/skills
cp -r skills/build-prometheus-icp-mcp-server skills/byoc .claude/skills/
```

Claude Code picks them up automatically — just describe what you want ("build me an MCP server for Prometheus Protocol", "register my canister in the app store") and the matching skill loads.

## Suggested usage

1. Start with `build-prometheus-icp-mcp-server` if you're creating a new MCP server. It covers everything from scaffold to a live, QA'd app store listing.
2. Use `byoc` if you already have a deployed canister and want to register/update it in the Prometheus app store.
3. Link the repo on [ICForge](https://icforge.dev) once your canister exists and ICForge has been added as a controller — this is the one manual browser step.

## Notes

- Namespace convention: `io.github.<owner>.<app-name>`
- MCP URL format: `https://<canister-id>.icp0.io/mcp`
- Certificate URL format: `https://prometheusprotocol.org/certificate/<namespace>`
- Prometheus discovery/registry canister: `grhdx-gqaaa-aaaai-q32va-cai`
- ICForge controller principal: `hu6sm-q3tid-2vbh7-wy3qp-r776c-zakxe-iueaq-2bsxh-zkluq-5j6wi-kae`
