# Development

## Tooling

Linkamp uses [mise](https://mise.jdx.dev/) as the source of truth for development-tool versions. The repository pins Go and its Go development tools in `mise.toml`.

After cloning the repository:

```sh
mise trust
mise install
```

Verify the environment with:

```sh
mise exec -- go version
mise exec -- gopls version
mise exec -- dlv version
mise exec -- staticcheck -version
mise exec -- govulncheck -version
```

Do not update repo-wide tools with unpinned `go install ...@latest` commands. Update the relevant entry in `mise.toml` so every contributor receives the same version.

## VSCodium

Install the workspace's recommended `golang.go` extension when VSCodium prompts, or run:

```sh
codium --install-extension golang.go
```

The shared workspace settings enable formatting and import organization on save, enable Staticcheck diagnostics through gopls, and prevent the extension from independently updating tools managed by mise. The standalone mise-pinned `staticcheck` command remains available for explicit command-line and CI checks; configuring it as the client-side VSCodium linter as well would produce duplicate diagnostics.

If VSCodium was launched from a desktop menu and cannot find Go or its tools, launch it with the repository's mise environment:

```sh
mise exec -- codium .
```

Personal interface preferences should remain in VSCodium's user settings rather than `.vscode/settings.json`.

## What the tools do

- **Go** compiles and tests Linkamp.
- **gopls** provides navigation, completion, refactoring, and diagnostics.
- **goimports** formats Go and maintains imports.
- **Delve (`dlv`)** provides source-level debugging.
- **Staticcheck** finds correctness and maintainability problems beyond the compiler.
- **govulncheck** reports known vulnerabilities that affect the Go code and its dependencies.
