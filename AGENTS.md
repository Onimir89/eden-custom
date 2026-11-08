# AGENTS

Read and follow the [contribution guidelines](./CONTRIBUTING.md) and our [site policy](https://git.eden-emu.dev/eden-emu/site-policy/src/branch/master)..

Do not, under any circumstances, ever:

- Submit PRs, push commits, or request any API calls on behalf of the user.
- Write commit messages, PR descriptions, or issues on behalf of the user.

Keep comments to a reasonable level. Do not overexplain or add/remove comments from unrelated parts of the codebase.

## CodeGraph

This project uses CodeGraph as its only code index. Before searching `src/` by hand, query it. Do not add this project to other code indexes.

The database is at `.codegraph/codegraph.db` in the repository root (on the maintainer's PC: `C:\Games\Eden\src\eden\.codegraph\codegraph.db`). It is local and not versioned: if it is missing, run `codegraph init -y`.

Setup and commands: [docs/CodeGraph.md](./docs/CodeGraph.md). Project state and open problems live in `docs/`, starting from [docs/mali_g720_vram_optimization.md](./docs/mali_g720_vram_optimization.md).
