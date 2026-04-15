## Context

complyctl uses hashicorp/go-plugin to run scanning providers as native subprocesses communicating via gRPC. Gemara compliance content (catalogs, guidance, policies) ships as YAML layers in OCI artifacts. These are two independent distribution and execution paths.

The platform engineer persona installs content via `complyctl get` (OCI pull) and plugins via RPM or manual binary placement. Every plugin — regardless of whether it needs host access — runs as a subprocess with full user-level privileges.

Most scanning providers are API-based (Kubernetes, cloud, GitHub). They need outbound HTTP and nothing else. A minority (OpenSCAP, Lynis) require filesystem access, subprocess execution, and sometimes root. The current model grants maximum privileges to all.

**Existing infrastructure**: `oras-go/v2` for OCI operations, `internal/cache/` for local OCI layout storage, `pkg/plugin/` for discovery/lifecycle/routing, `internal/policy/` for graph resolution.

## Goals / Non-Goals

**Goals:**

- Structural least-privilege: Wasm plugins cannot access host filesystem, exec subprocesses, or open raw sockets — by construction, not policy
- Single-artifact distribution: one `complyctl get` fetches content and plugin together
- Platform-neutral plugins: Wasm modules run on any OS/arch without recompilation
- Coexistence: native (go-plugin/gRPC) and Wasm (wazero) plugins in the same scan workflow
- Familiar authoring: plugin authors write Go, implement the same Describe/Generate/Scan interface
- Zero new external services: wazero is pure Go, embedded in complyctl, no daemon or sidecar

**Non-Goals:**

- Replacing native plugins — host-scanning providers (OpenSCAP) remain native
- Mixed policies in a single `complytime.yaml` referencing both complypacks and content-only policies that route to the same evaluator (deferred — no clear end-user need)
- Runtime capability prompting — pull = trust; no interactive permission grants
- Multi-language plugin SDKs — Go (TinyGo) only for initial release
- Wasm Component Model — use WASI preview 1 with custom host functions; evaluate Component Model adoption later

## Decisions

### D1: Wazero as the Wasm runtime

**Choice**: github.com/tetratelabs/wazero

**Alternatives considered**:
- wasmtime-go: Requires CGo and a C toolchain. Adds build complexity and cross-compilation pain.
- wasmer-go: Also CGo-based. Less active Go community.

**Rationale**: Wazero is pure Go (zero CGo), compiles with standard `go build`, supports WASI preview 1, and provides host function registration. It's the only option that doesn't add a C dependency to complyctl's build. ~15MB binary size increase.

### D2: Host function ABI (three functions)

**Choice**: Three host functions define the complete Wasm plugin capability surface:

| Function | Purpose |
|:---|:---|
| `http_request` | Outbound HTTP — host executes, returns response |
| `log_message` | Structured logging to complyctl.log |
| `read_var` | Lazy variable resolution from merged context |

**Alternatives considered**:
- WASI networking (sock_open, sock_send): Exposes raw socket operations. Too broad — plugins could open arbitrary connections. Also poorly supported in WASI preview 1.
- Full WASI filesystem: Grants read/write to host filesystem. Defeats the sandboxing goal.
- Larger function set (separate GET/POST/PUT, file read, exec): More surface area to maintain and secure. Every host function is an attack surface.

**Rationale**: Three functions cover all API-based plugin needs. HTTP is the only I/O primitive API-based plugins need. Logging and variable access are operational necessities. The minimal surface is deliberate — adding a fourth function later (if needed) is a small, reviewable change.

### D3: JSON serialization over shared memory

**Choice**: Requests and responses serialized as JSON, passed via Wasm linear memory (pointer + length).

**Alternatives considered**:
- Protobuf: TinyGo protobuf support is experimental. Would require a separate code generator or hand-written marshaling.
- FlatBuffers / Cap'n Proto: Zero-copy but adds complexity and dependencies for the plugin author.
- MessagePack: Binary format, but no significant advantage over JSON for the payload sizes involved.

**Rationale**: TinyGo's `encoding/json` works (since 0.30+). JSON is human-debuggable. Plugin payloads are small (assessment configs, variable maps). Serialization overhead is negligible compared to HTTP round-trips in Scan.

### D4: OCI multi-layer complypack artifact

**Choice**: Complypack is a standard OCI manifest with Gemara YAML layers plus a Wasm plugin layer. The OCI config blob carries plugin metadata (evaluator_id, capabilities, version).

| Layer | Media Type |
|:---|:---|
| Catalog | `application/vnd.gemara.catalog.v1+yaml` |
| Guidance | `application/vnd.gemara.guidance.v1+yaml` |
| Policy | `application/vnd.gemara.policy.v1+yaml` |
| Plugin | `application/vnd.complypack.plugin.v1+wasm` |

Config: `application/vnd.complypack.config.v1+json`

**Rationale**: Reuses existing oras-go OCI infrastructure. Content-only policies (no wasm layer) remain valid — backward compatible. `complyctl get` already iterates manifest layers by media type; adding one more type is incremental.

### D5: Generate is optional, Scan carries configurations

**Choice**: `ScanRequest` gains `Configurations []AssessmentConfiguration` and `GenerateState []byte`. Wasm plugins receive everything in Scan. Generate is available but not required.

**Alternatives considered**:
- Always require Generate: Forces a two-step workflow even for stateless API plugins. Adds friction for the SRE persona.
- Remove Generate entirely for Wasm: Prevents future use cases (OPA pre-compilation, drift baselines) that benefit from a preparation step.

**Rationale**: Optional Generate preserves flexibility. The `GenerateState` blob allows Wasm plugins that do use Generate to pass opaque state through the host without breaking the stateless sandbox model. Native plugins ignore the new fields (zero-value).

### D6: Host-managed state blob for Wasm Generate

**Choice**: `GenerateResponse` gains `State []byte`. Host persists to `~/.complytime/generation/{evaluator-id}.state`. Host passes blob into `ScanRequest.GenerateState` on subsequent Scan.

**Rationale**: The plugin stays stateless (no filesystem). The host handles persistence using existing generation state infrastructure. The blob is opaque — host never inspects it. Clean separation of concerns.

### D7: Dual-path discovery — file extension discrimination

**Choice**: Discovery identifies plugin type by file:
- `*.wasm` → Wasm plugin (wazero runtime)
- `complyctl-provider-*` (executable, no extension) → native plugin (go-plugin/gRPC)

Evaluator ID derived from filename:
- `kubernetes.wasm` → evaluator ID `kubernetes`
- `complyctl-provider-openscap` → evaluator ID `openscap`

**Rationale**: Consistent with existing naming convention. No manifest files. No configuration. File presence and name encode everything.

### D8: Wasm extraction during `complyctl get`

**Choice**: After `oras.Copy()`, scan manifest layers. If a layer with media type `application/vnd.complypack.plugin.v1+wasm` exists, extract the blob to `~/.complytime/providers/{evaluator-id}.wasm`. Evaluator ID read from OCI config blob.

**Rationale**: Reuses existing sync infrastructure. Extraction is a post-sync hook, not a separate command. The SRE runs `complyctl get` and both content and plugin are ready.

## Risks / Trade-offs

**[TinyGo stdlib limitations]** → TinyGo doesn't support `net/http`, `reflect` (fully), or `os/exec`. Plugin authors must use host-function wrappers for HTTP. Mitigated by the SDK abstracting all host calls behind idiomatic Go functions. Document limitations clearly in plugin authoring guide.

**[Wazero version stability]** → Wazero is pre-1.0 in API stability terms but post-1.0 in releases. API breaking changes possible. → Pin exact version. Wrap wazero calls in an internal adapter package to isolate complyctl from API churn.

**[Performance of JSON serialization in Wasm]** → JSON marshal/unmarshal in TinyGo is slower than native Go. → For the payload sizes involved (assessment configs, variable maps, assessment logs), this is sub-millisecond. Not a concern unless payloads grow to megabytes.

**[Binary size increase]** → Wazero adds ~15MB to the complyctl binary. → Acceptable for a CLI tool. No impact on Wasm plugin size (2-5MB typical).

**[Plugin author ecosystem friction]** → Requiring TinyGo (not standard Go) is a new toolchain dependency for plugin authors. → Document clearly. Provide Makefile targets. TinyGo installation is one command (`go install` or package manager). complyctl itself still builds with standard Go — only plugin authors need TinyGo.

**[Host function escape hatch pressure]** → Over time, plugin authors may request filesystem access, subprocess exec, etc. Each addition erodes the sandboxing value. → Treat each new host function as a security decision requiring explicit justification. Default answer is "use a native plugin" for capabilities beyond HTTP/log/vars.
