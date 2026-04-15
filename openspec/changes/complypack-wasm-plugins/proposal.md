## Why

complyctl's plugin system grants every scanning provider full user-level privileges — filesystem, network, subprocess exec — regardless of whether the plugin needs them. Most plugins are API-based (Kubernetes, cloud, GitHub) and only need outbound HTTP. A compromised or malicious API-based plugin today has the same blast radius as a host-scanning plugin that legitimately needs root.

Separately, content (Gemara YAML via OCI) and logic (native Go binaries via RPM/manual install) ship through two independent distribution paths. The platform engineer — the primary persona — must manage both paths to get a working compliance workflow.

WebAssembly plugins solve both problems: structural sandboxing enforces least-privilege by default, and Wasm modules distribute via OCI as part of a "complypack" — a self-contained OCI artifact bundling a Wasm plugin, its policy-as-code content, and a manifest. Gemara (compliance-authored standards and guidelines) remains a separate content stream. Two OCI streams: Gemara content and complypacks.

## What Changes

- Introduce "complypack" — an OCI artifact bundling a Wasm plugin module, its policy-as-code assessment content, and a manifest. Complypacks are a distinct OCI stream from Gemara content. Gemara defines standards (compliance-authored); complypacks define how to check them (engineering-authored).
- Add a Wasm plugin runtime (wazero, pure Go) to complyctl as an alternative to the native go-plugin/gRPC path
- Define a host function ABI (http_request, log_message, read_var) as the complete capability surface for sandboxed plugins
- Create a Go plugin authoring SDK (`pkg/wasmplugin`) targeting TinyGo compilation to Wasm
- Extend `complyctl get` to handle complypacks — cache policy-as-code content layers and extract the Wasm module into the providers directory
- Extend `complyctl doctor` with complypack integrity validation and privilege visibility
- Make `Generate` optional for Wasm plugins — `Scan` receives assessment configurations directly
- Add host-managed state blob passing for Wasm plugins that use Generate (opaque bytes, host persists)
- **BREAKING**: `ScanRequest` gains `Configurations` and `GenerateState` fields (backward-compatible for native plugins which ignore them)

## Capabilities

### New Capabilities

- `wasm-runtime`: Wazero-based Wasm plugin execution within complyctl — module loading, host function registration, memory management, timeout enforcement
- `wasm-plugin-sdk`: Go authoring SDK (`pkg/wasmplugin`) with Plugin interface, host function wrappers, and TinyGo build support
- `complypack-distribution`: Two-stream OCI model (Gemara content, complypacks), complypack artifact structure (plugin + PaC content + manifest), extraction during `complyctl get`, media type conventions
- `plugin-sandboxing`: Least-privilege enforcement for Wasm plugins — capability boundary between sandboxed (Wasm) and full-access (native) plugin paths
- `complypack-doctor`: Doctor checks for Wasm runtime availability, complypack integrity (digest, exports, module validity), and privilege visibility in provider status

### Modified Capabilities

(none — no existing specs to modify)

## Impact

- **complyctl dependencies**: Adds `github.com/tetratelabs/wazero` (pure Go, no CGo, ~15MB)
- **Plugin interface**: `ScanRequest` gains two fields. Native plugins unaffected (zero-value fields ignored). Proto contract adds fields without breaking wire compatibility.
- **Plugin discovery**: `pkg/plugin/discovery.go` extends to discover `*.wasm` alongside `complyctl-provider-*` executables
- **Plugin manager**: `pkg/plugin/manager.go` routes to `WasmClient` or native `Client` based on discovered type
- **Cache/sync**: `internal/cache/sync.go` adds complypack handling — cache PaC content layers alongside Gemara, extract Wasm module to providers directory
- **Doctor**: New check categories (`wasm-runtime`, `pack/{id}`) and modified provider output showing plugin type and access level
- **Build toolchain**: Plugin authors need TinyGo for Wasm compilation. complyctl itself builds with standard Go.
- **OCI media types**: New `application/vnd.complypack.plugin.v1+wasm` (Wasm layer), `application/vnd.complypack.config.v1+json` (complypack config with evaluator metadata)
