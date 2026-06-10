# Prometheus Protocol Skills

Practical skills and playbooks for building MCP servers for Prometheus Protocol and hosting/deploying them with ICForge.

## Included skills

- [`skills/build-prometheus-icp-mcp-server.md`](skills/build-prometheus-icp-mcp-server.md)
  - End-to-end workflow for scaffolding, implementing, building, deploying, configuring, registering, and QA'ing a Prometheus Protocol MCP server on ICP.
- [`skills/byoc.md`](skills/byoc.md)
  - Bring-your-own-canister registration workflow for publishing an externally deployed canister to Prometheus Protocol and wiring ICForge CI/CD.

## Suggested usage

1. Start with `build-prometheus-icp-mcp-server.md` if you're creating a new MCP server.
2. Use `byoc.md` if you already have a deployed canister and want to register/update it in the Prometheus app store.
3. Link the repo on [ICForge](https://icforge.dev) once your canister exists and ICForge has been added as a controller.

## Notes

- Namespace convention: `io.github.<owner>.<app-name>`
- MCP URL format: `https://<canister-id>.icp0.io/mcp`
- Certificate URL format: `https://prometheusprotocol.org/certificate/<namespace>`
- Prometheus discovery/registry canister: `grhdx-gqaaa-aaaai-q32va-cai`
- ICForge controller principal: `hu6sm-q3tid-2vbh7-wy3qp-r776c-zakxe-iueaq-2bsxh-zkluq-5j6wi-kae`
