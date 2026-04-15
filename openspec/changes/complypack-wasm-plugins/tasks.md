## 1. Wasm Runtime Foundation

- [ ] 1.1 Add `github.com/tetratelabs/wazero` dependency to `go.mod`
- [ ] 1.2 Create `internal/wasm/runtime.go` — initialize wazero runtime with WASI snapshot preview 1, 256MB memory limit, context-based timeout
- [ ] 1.3 Create `internal/wasm/host_http.go` — implement `http_request` host function: read JSON from Wasm memory, execute `net/http` request, write JSON response back
- [ ] 1.4 Create `internal/wasm/host_log.go` — implement `log_message` host function: read level + message from Wasm memory, route to hclog logger with evaluator ID prefix
- [ ] 1.5 Create `internal/wasm/host_var.go` — implement `read_var` host function: resolve key from merged variable context (target → global), write value back to Wasm memory
- [ ] 1.6 Create `internal/wasm/module.go` — compile module, validate required exports (`describe`, `generate`, `scan`), instantiate with host functions registered
- [ ] 1.7 Write unit tests for runtime initialization, memory limits, and timeout enforcement

## 2. Wasm Plugin Client

- [ ] 2.1 Create `pkg/plugin/wasm_client.go` — implement `Plugin` interface (Describe/Generate/Scan) backed by wazero module invocation: marshal request to JSON, call exported function, unmarshal response
- [ ] 2.2 Handle `GenerateResponse.State` persistence: write to `~/.complytime/generation/{evaluator-id}.state`, read and pass via `ScanRequest.GenerateState`
- [ ] 2.3 Add `ScanRequest.Configurations` and `ScanRequest.GenerateState` fields to the domain types in `pkg/plugin/`
- [ ] 2.4 Write unit tests for WasmClient with a minimal test `.wasm` module

## 3. Dual-Path Discovery

- [ ] 3.1 Extend `pkg/plugin/discovery.go` to discover `*.wasm` files alongside `complyctl-provider-*` executables — derive evaluator ID from filename (strip `.wasm` extension)
- [ ] 3.2 Add `PluginType` field to `PluginInfo` (enum: `Native`, `Wasm`)
- [ ] 3.3 Update `pkg/plugin/manager.go` `LoadPlugins()` to instantiate `WasmClient` for Wasm plugins and existing `Client` for native plugins
- [ ] 3.4 Update `RouteGenerate` and `RouteScan` to pass `Configurations` and `GenerateState` when routing to Wasm plugins
- [ ] 3.5 Write unit tests for dual-path discovery and routing

## 4. Complypack Distribution (Two-Stream OCI)

- [ ] 4.1 Add media type constants to `internal/complytime/consts.go`: `application/vnd.complypack.plugin.v1+wasm`, `application/vnd.complypack.config.v1+json`
- [ ] 4.2 Implement complypack detection in `internal/cache/sync.go` — after `oras.Copy()`, inspect config media type to distinguish complypacks from content-only Gemara policies
- [ ] 4.3 Implement Wasm extraction: when complypack detected, read evaluator ID from config blob, extract `.wasm` layer by media type to `~/.complytime/providers/{evaluator-id}.wasm`
- [ ] 4.4 Handle update case: on complypack digest change, re-cache all layers and overwrite the `.wasm` file
- [ ] 4.5 Write integration test: sync a test complypack OCI artifact containing PaC content + Wasm layer, verify content cached and `.wasm` extracted to correct path

## 5. Plugin SDK (`pkg/wasmplugin`)

- [ ] 5.1 Create `pkg/wasmplugin/types.go` — all request/response types mirroring proto contract, JSON-serializable, TinyGo-compatible
- [ ] 5.2 Create `pkg/wasmplugin/plugin.go` — `Plugin` interface and `Register()` function with global storage and exported Wasm function wiring
- [ ] 5.3 Create `pkg/wasmplugin/host.go` — `HTTPRequest`, `Log`, `ReadVar` wrapper functions using `//go:wasmimport complyctl` directives
- [ ] 5.4 Create `pkg/wasmplugin/memory.go` — shared memory helpers for pointer/length marshaling (allocate, read, write)
- [ ] 5.5 Verify full SDK compiles with TinyGo (`tinygo build -target wasi`)
- [ ] 5.6 Create a minimal test plugin using the SDK, build to `.wasm`, use in unit tests from task 2.4

## 6. Doctor Changes

- [ ] 6.1 Add `wasm-runtime` check to `internal/doctor/doctor.go` — verify wazero initializes, conditional on Wasm plugins being discovered
- [ ] 6.2 Add `pack/{id}` check — implement five-step integrity chain: complypack cached → file extracted → digest matches → module compiles → exports present
- [ ] 6.3 Modify `provider/{id}` check to include plugin type (native/wasm) and access level (full host access/sandboxed) in output
- [ ] 6.4 Modify `policy/{id}` check to append "(complypack)" when policy was resolved from a complypack composition manifest
- [ ] 6.5 Implement `--verbose` expansion for Wasm providers: OCI origin, full digest, size, exports, imports
- [ ] 6.6 Implement check ordering: wasm-runtime gates Wasm provider and pack checks — skip with message on runtime failure
- [ ] 6.7 Write unit tests for all new and modified doctor checks

## 7. Scan Flow Integration

- [ ] 7.1 Update `complyctl scan` to always call `ResolvePolicyGraph()` regardless of plugin type
- [ ] 7.2 Update scan to pass `Configurations` from graph resolution into `RouteScan` for Wasm plugins
- [ ] 7.3 Add scan preamble output: print execution plan (targets, providers, requirement counts, plugin types) before invoking plugins
- [ ] 7.4 Verify merged EvaluationLog output when both native and Wasm plugins produce AssessmentLogs in the same scan
- [ ] 7.5 Write integration test: scan with one native and one Wasm plugin, verify single unified output

## 8. Documentation and Authoring Guide

- [ ] 8.1 Create Wasm plugin authoring guide (parallel to existing `PLUGIN_GUIDE.md`) covering SDK usage, TinyGo build, and host function API
- [ ] 8.2 Document two-stream OCI model (Gemara content vs complypacks), media type conventions, and authorship boundaries
- [ ] 8.3 Document Generate optionality and tradeoffs (pre-flight validation, execution plan preview, freshness detection) for Wasm plugins
- [ ] 8.4 Update `complyctl doctor` documentation with new check categories and verbose output
