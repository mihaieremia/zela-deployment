# zela-deployment

Reusable GitHub Actions workflows for building and deploying [Zela](https://zela.io) custom procedures.

## Quick start

To build and deploy a procedure in a single workflow run:

```yaml
# .github/workflows/deploy.yml
name: Deploy to Zela
on:
  workflow_dispatch:
    inputs:
      procedure:
        type: choice
        options: [hello_world, block_time, priority_fees]

jobs:
  deploy:
    uses: mihaieremia/zela-deployment/.github/workflows/build-and-deploy.yml@v1
    with:
      procedure: ${{ inputs.procedure }}
      project:   ${{ vars.ZELA_PROJECT }}
    secrets:
      key-id:     ${{ vars.ZELA_KEY_ID }}
      key-secret: ${{ secrets.ZELA_KEY_SECRET }}
```

## Available workflows

| Workflow | Purpose |
|---|---|
| [`build-and-deploy.yml`](.github/workflows/build-and-deploy.yml) | Builds the procedure and deploys it in two chained jobs. |
| [`build-procedure.yml`](.github/workflows/build-procedure.yml) | Builds the procedure and uploads the wasm as an artifact. Use for pull-request validation or any pre-deploy check. |
| [`deploy-procedure.yml`](.github/workflows/deploy-procedure.yml) | Deploys a previously built wasm artifact. Use when the build runs in a custom job. |

### `build-and-deploy.yml`

```yaml
jobs:
  deploy:
    uses: mihaieremia/zela-deployment/.github/workflows/build-and-deploy.yml@v1
    with:
      procedure: hello_world           # required: dashboard procedure name
      project:   ${{ vars.ZELA_PROJECT }}  # required: Zela project UUID
      # Optional inputs:
      cargo-package:    hello_world    # defaults to `procedure`
      rust-toolchain:   stable         # defaults to "stable"
      wasi-sdk-version: "30"           # defaults to "30"
      runs-on:          ubuntu-latest  # defaults to "ubuntu-latest"
      cargo-build-args: ""             # extra args appended to `cargo build`
      core-url:         https://core.zela.io
      auth-url:         https://auth.zela.io/realms/zela/protocol/openid-connect/token
    secrets:
      key-id:     ${{ vars.ZELA_KEY_ID }}      # required
      key-secret: ${{ secrets.ZELA_KEY_SECRET }} # required
```

### `build-procedure.yml`

```yaml
jobs:
  build:
    uses: mihaieremia/zela-deployment/.github/workflows/build-procedure.yml@v1
    with:
      procedure: hello_world

  inspect:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: ${{ needs.build.outputs.artifact-name }}
      - run: ls -la *.wasm
```

### `deploy-procedure.yml`

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # ... custom build, ending in:
      - uses: actions/upload-artifact@v4
        with:
          name: my-procedure-wasm
          path: target/wasm32-wasip2/release/my_procedure.wasm

  deploy:
    needs: build
    uses: mihaieremia/zela-deployment/.github/workflows/deploy-procedure.yml@v1
    with:
      procedure:     my_procedure
      project:       ${{ vars.ZELA_PROJECT }}
      wasm-artifact: my-procedure-wasm
    secrets:
      key-id:     ${{ vars.ZELA_KEY_ID }}
      key-secret: ${{ secrets.ZELA_KEY_SECRET }}
```

## Repository configuration

Configure the following in **Settings → Secrets and variables → Actions**:

| Type | Name | Value |
|---|---|---|
| Variable | `ZELA_PROJECT` | Zela project UUID from the dashboard |
| Variable | `ZELA_KEY_ID` | Project key ID |
| Secret | `ZELA_KEY_SECRET` | Matching project key secret |

The key pair requires the `zela-builder:read zela-builder:write` scope, which the dashboard grants by default to project keys.

## Versioning

Releases follow SemVer git tags.

| Pin form | Example | When to use |
|---|---|---|
| Major (moving) | `@v1` | Recommended. Receives non-breaking updates automatically. |
| Exact | `@v0.1.0` | Required for audit-relevant deploys or when reproducibility matters. |
| `main` | `@main` | Not recommended. The branch may change at any time. |

The `v1` tag tracks the latest `1.x.y` release.

## Building locally

To reproduce the CI build on a workstation:

```sh
# Install wasi-sdk (use the platform-appropriate package).
brew install --cask wasi-sdk    # macOS
rustup target add wasm32-wasip2

cargo build --release --locked --target wasm32-wasip2 --package <procedure>
```

## License

[MIT](LICENSE)
