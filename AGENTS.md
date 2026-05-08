# AGENTS.md

## 文档规则
- 仓库级代理入口文件仍保持命名为 `AGENTS.md`，以兼容常见 agent / tooling 的自动发现。
- 如果使用 `init` 或类似初始化流程生成额外的 agent 说明文件，**统一命名为 `opencode_agent.md`**，不要生成其他英文变体文件名。
- 新增或更新 agent 文档时，**文档标题、说明性注释、规则描述一律使用中文**。

## Repo shape
- This is a **single Go module** (`go.mod`) with **many deployable binaries**, not a single app. Runtime entrypoints live under `cmd/`; service implementations live under `internal/`; reusable wiring lives under `pkg/`.
- The most useful wiring files are:
  - `magefile.go` — real local build/start/check entrypoints
  - `start-config.yml` — default binaries + replica counts used by `mage start`
  - `cmd/main.go` — aggregated launcher; **requires `-c <config dir>`** and exits if omitted
  - `pkg/common/config/config.go` — canonical YAML config filenames and shared config structs
  - `pkg/common/startrpc/start.go` — generic RPC bootstrap (discovery, interceptors, prometheus, gRPC server)
  - `internal/rpc/msg/server.go` — representative service assembly file

## Preferred workflow
- **Prefer `mage`, not guessed `make` targets.** The executable sources of truth (`magefile.go`, `Dockerfile`, CI workflow) all use Mage.
- Install Mage with either:
  - `./bootstrap.sh`
  - or `go install github.com/magefile/mage@latest`
- Core commands:
  - `mage build` — build binaries into `_output`
  - `mage start` — start the default service set from `start-config.yml`
  - `mage check` — verify started binaries
  - `mage stop` — stop binaries started by Mage
- `mage build` accepts a subset of binaries. Example from `magefile.go`: `mage build openim-api openim-rpc-user seq`

## Running services manually
- `cmd/main.go` is the all-in-one launcher for local runs: `go run ./cmd -c ./config`
- The aggregated launcher is for local/dev orchestration. It **forces standalone discovery** in code and loads config files from the directory passed by `-c`.
- Individual service mains live under `cmd/openim-*` and `cmd/openim-rpc/*`. Read the matching file in `internal/` before changing service wiring.

## Config and environment gotchas
- Config is directory-based, not single-file. The process expects a config directory containing specific YAML filenames defined in `pkg/common/config/config.go` and exemplified under `config/`.
- `start-config.yml` controls which binaries `mage start` launches and how many replicas each gets.
- Toolchain declarations conflict:
  - `go.mod` says `go 1.25.0`
  - CI and `Dockerfile` use Go 1.22
  - `install.sh` still mentions Go 1.20
  Verify your local Go version before treating build failures as code regressions.

## Testing and verification
- There is no root `Makefile` in this checkout. Docs mention `make test*`/`make release`, but do **not** assume those targets exist locally unless you verified them in your environment.
- Fastest repo-native verification for most changes:
  - `mage build`
  - if startup/wiring/config changed: `mage start && mage check`
- Integration / E2E prerequisites require infrastructure first:
  - `docker compose up -d`
- E2E commands from `test/e2e/README.md`:
  - full suite: `ginkgo -v --randomizeAllSpecs --randomizeSuites --failOnPending --cover --trace --race --progress`
  - focused suite: `ginkgo -v ./test/e2e/chat`
- CI does more than unit tests: `.github/workflows/go-build-test.yml` builds with Mage, starts services, then runs real HTTP smoke tests with `curl`. If you touch service wiring, config loading, ports, or auth flows, prefer a startup smoke test over `go test` alone.

## Code conventions that matter here
- Format Go code with **both** `gofmt` and `goimports` per `docs/contrib/go-code.md`.
- Logging convention from `CONTRIBUTING.md`: **do not use the standard `log` package**. Use `github.com/openimsdk/tools/log`.
- Error handling convention from `CONTRIBUTING.md`: wrap errors with `github.com/openimsdk/tools/errs`; do not use `panic` for normal error flow.
- `.golangci.yml` is strict and repo-specific; avoid noisy debug output and ad-hoc logging/printing changes unless truly necessary.

## Protobuf / generated code
- If you need proto generation, the repo documents a **custom protoc workflow** in `docs/contrib/protoc-tools.md`, not a buf-based flow.
- Documented invocation is `./protoc [OPTION] PROTO_FILES`. Do not assume stock `protoc` flags or external codegen wrappers are the intended path.

## Reading strategy for future agents
- For architecture questions, start with `magefile.go`, `cmd/main.go`, `pkg/common/config/config.go`, and `pkg/common/startrpc/start.go` before reading leaf handlers.
- For behavior changes inside one service, read the service entrypoint under `cmd/`, then the corresponding `internal/` package that builds its dependencies.
