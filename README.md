# Go Build

![Latest Release](https://img.shields.io/github/v/release/p6m-actions/golang-build?style=flat-square&label=Latest%20Release&color=blue)

## Description

Builds and tests a Go project, generating the code a Go service imports but does not commit.

The generation is the point. A p6m Go service imports two kinds of code that are **produced, not
rendered**: protobuf bindings under `gen/` for gRPC, and the gqlgen server for GraphQL. A fresh
clone does not compile until they exist. Running them here — inside the action that needs them —
is what keeps the CI path and the container build path from drifting apart; when they were only in
the Dockerfile, every rendered repository's first CI run failed.

Order matters, and it is not arbitrary:

1. **`protoc` first**, before `go mod tidy`. It is a standalone binary rather than a Go tool, so it
   needs nothing resolved first — and tidy has to see `gen/` to resolve what the service imports.
2. **`go mod tidy`**, which also materializes `go.sum` (a freshly rendered project ships none,
   because producing one needs the toolchain that rendering does not have).
3. **`gqlgen` after tidy**, unlike protoc: it runs through `go run`, so its own dependency has to be
   resolvable first. A second tidy picks up what the generated code imports.

Both generation steps are guarded on their inputs (`proto/`, `gqlgen.yml`) and are no-ops otherwise,
so the same action serves gRPC, GraphQL and plain REST services.

## Usage

```yaml
- uses: actions/checkout@v4
- uses: p6m-actions/golang-setup@v1
- uses: p6m-actions/golang-build@v1
```

## Inputs

### Code generation

| Input | Description | Required | Default |
|---|---|---|---|
| `run-codegen` | Run guarded code generation (protoc for `proto/`, gqlgen for `gqlgen.yml`) | No | `true` |
| `protoc-gen-go-version` | Version of `protoc-gen-go` to install when missing | No | `v1.36.5` |
| `protoc-gen-go-grpc-version` | Version of `protoc-gen-go-grpc` to install when missing | No | `v1.5.1` |

### Module hygiene

| Input | Description | Required | Default |
|---|---|---|---|
| `run-tidy` | Run `go mod tidy` | No | `true` |

### Checks

| Input | Description | Required | Default |
|---|---|---|---|
| `run-format-check` | Check formatting with gofmt | No | `false` |
| `format-command` | Command whose output lists misformatted files | No | `gofmt -l .` |
| `run-vet` | Run `go vet` | No | `false` |
| `vet-command` / `vet-args` | Vet command and arguments | No | `go vet` / `./...` |
| `run-lint` | Run golangci-lint | No | `false` |
| `lint-command` / `lint-args` | Lint command and arguments | No | `golangci-lint run` / `""` |
| `golangci-lint-version` | Version of golangci-lint to install | No | `v1.62.2` |

> **Why the checks default to off.** The p6m Go pipeline does not gate on formatting or vet today.
> Turning them on changes what every existing Go service's CI enforces, so it is a fleet decision
> that the CI-parity proofs (`prova-p6m-standards` S10, `p6m.ci.stacks.golang`) should carry
> deliberately — not a default that flips under repositories that were green yesterday. Enable them
> per repository, or flip the defaults together with the S10 mirror.

### Build and test

| Input | Description | Required | Default |
|---|---|---|---|
| `run-build` | Run the build | No | `true` |
| `build-command` / `build-args` | Build command and arguments | No | `go build` / `./...` |
| `run-test` | Run the tests | No | `true` |
| `test-command` / `test-args` | Test command and arguments | No | `go test` / `./...` |
| `archive-coverage` | Upload coverage results as an artifact | No | `false` |
| `coverage-path` | Path to the coverage reports | No | `coverage` |
| `working-directory` | Working directory for go commands | No | `.` |

## Outputs

| Output | Description |
|---|---|
| `build-status` | `success` or `failure` |

## Examples

### Enable the format check and vet for one repository

```yaml
- uses: p6m-actions/golang-build@v1
  with:
    run-format-check: "true"
    run-vet: "true"
```

### Test with coverage

```yaml
- uses: p6m-actions/golang-build@v1
  with:
    test-args: "./... -coverprofile=coverage/cover.out"
    archive-coverage: "true"
```

### Skip generation on a repository that commits its generated code

```yaml
- uses: p6m-actions/golang-build@v1
  with:
    run-codegen: "false"
```
