# Prometheus Protocol Skills

Agent skills for building MCP servers for Prometheus Protocol and hosting/deploying them with ICForge.

Skills live in [`.claude/skills/`](.claude/skills/) in the [Claude Code skills format](https://code.claude.com/docs/en/skills) — a directory per skill containing a `SKILL.md` with YAML frontmatter. Claude Code discovers and invokes them automatically when the task matches.

## Quick start

**Easiest:** open a Claude Code session in this repo (or clone it and start one). The skills load automatically — just say what you want:

- *"Build me an MCP server for Prometheus Protocol"* → loads `build-prometheus-icp-mcp-server`
- *"Register my canister in the Prometheus app store"* → loads `byoc`

You can also invoke them explicitly with `/build-prometheus-icp-mcp-server` or `/byoc`.

**To use them from another project**, copy the skill directories into that project's `.claude/skills/` (or `~/.claude/skills/` to make them available everywhere):

```bash
mkdir -p /path/to/project/.claude/skills
cp -r .claude/skills/build-prometheus-icp-mcp-server .claude/skills/byoc /path/to/project/.claude/skills/
```

## Included skills

- [`.claude/skills/build-prometheus-icp-mcp-server/SKILL.md`](.claude/skills/build-prometheus-icp-mcp-server/SKILL.md)
  - End-to-end workflow for scaffolding, implementing, building, deploying, configuring, registering, and QA'ing a Prometheus Protocol MCP server on ICP.
- [`.claude/skills/byoc/SKILL.md`](.claude/skills/byoc/SKILL.md)
  - Bring-your-own-canister registration workflow for publishing an externally deployed canister to Prometheus Protocol and wiring ICForge CI/CD. Also owns asset generation, the GitHub release, and the `prometheus.yml` manifest — the build skill delegates its registration phase here.

## Suggested usage

1. Start with `build-prometheus-icp-mcp-server` if you're creating a new MCP server. It covers everything from scaffold to a live, QA'd app store listing.
2. Use `byoc` if you already have a deployed canister and want to register/update it in the Prometheus app store.
3. The only manual browser step in either workflow is logging into [ICForge](https://icforge.dev) and connecting the repo — the skills tell you when.

## Notes

- Namespace convention: `io.github.<owner>.<app-name>`
- MCP URL format: `https://<canister-id>.icp0.io/mcp`
- Certificate URL format: `https://prometheusprotocol.org/certificate/<namespace>`
- Prometheus discovery/registry canister: `grhdx-gqaaa-aaaai-q32va-cai`
- ICForge controller principal: `hu6sm-q3tid-2vbh7-wy3qp-r776c-zakxe-iueaq-2bsxh-zkluq-5j6wi-kae`
