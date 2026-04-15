## ADDED Requirements

### Requirement: No filesystem access for Wasm plugins
Wasm plugins SHALL NOT have access to the host filesystem. No WASI filesystem preopens SHALL be granted. Plugins that need to read files (e.g., configuration) SHALL use `read_var` to access declared variables or receive data via request payloads.

#### Scenario: Wasm plugin attempts file read
- **WHEN** a Wasm plugin attempts a WASI `path_open` or `fd_read` on a host path
- **THEN** the operation fails with a permission error and does not access the host filesystem

### Requirement: No subprocess execution for Wasm plugins
Wasm plugins SHALL NOT have the ability to execute subprocesses on the host. No WASI `proc_exec` or equivalent host function SHALL be registered.

#### Scenario: Wasm plugin cannot exec
- **WHEN** a Wasm plugin attempts to spawn a subprocess
- **THEN** the operation is not possible — no host function exists for it

### Requirement: Network access only via host HTTP function
Wasm plugins SHALL access the network exclusively through the `http_request` host function. No raw socket operations SHALL be available. The host SHALL execute HTTP requests on behalf of the plugin using the host's network stack and TLS configuration.

#### Scenario: Plugin makes API call via host function
- **WHEN** a Wasm plugin calls `http_request` with a Kubernetes API URL
- **THEN** the host executes the HTTP request using its TLS CA bundle and returns the response

#### Scenario: Plugin cannot open raw socket
- **WHEN** a Wasm plugin attempts to open a TCP connection directly
- **THEN** the operation is not possible — no host function exists for raw sockets

### Requirement: Variable access scoped to declared variables
The `read_var` host function SHALL resolve variables from the merged context (target variables, then global variables). Wasm plugins SHALL NOT have access to host environment variables beyond what is explicitly declared in the workspace config variables sections.

#### Scenario: Plugin reads declared target variable
- **WHEN** a Wasm plugin calls `read_var("api_server")` and `api_server` is in the target's variables
- **THEN** the value is returned

#### Scenario: Plugin cannot read arbitrary environment variable
- **WHEN** a Wasm plugin calls `read_var("HOME")`
- **THEN** the function returns not-found unless `HOME` is explicitly declared in the workspace config variables

### Requirement: Privilege visibility in doctor output
`complyctl doctor` SHALL display the access level for each discovered provider: native providers SHALL show "full host access" and Wasm providers SHALL show "sandboxed" with the list of granted host functions.

#### Scenario: Doctor shows native plugin access
- **WHEN** `complyctl doctor` reports on a native plugin
- **THEN** the output includes the plugin type as "native" and access level as "full host access"

#### Scenario: Doctor shows Wasm plugin access
- **WHEN** `complyctl doctor` reports on a Wasm plugin
- **THEN** the output includes the plugin type as "wasm" and access level as "sandboxed"

#### Scenario: Doctor verbose shows granted capabilities
- **WHEN** `complyctl doctor --verbose` reports on a Wasm plugin
- **THEN** the output lists the specific host functions available to the plugin (http_request, log_message, read_var) and the OCI origin of the Wasm module

### Requirement: Native and Wasm plugins coexist in a single scan
The system SHALL support executing both native and Wasm plugins within a single `complyctl scan` invocation. Results from both plugin types SHALL be merged into a single EvaluationLog output.

#### Scenario: Scan uses both native and Wasm plugins
- **WHEN** a scan workflow routes requirements to both a native provider and a Wasm provider
- **THEN** both execute (native via gRPC subprocess, Wasm via in-process wazero), and their AssessmentLog results merge into one EvaluationLog

### Requirement: Trust asymmetry between plugin types
Native plugins SHALL require explicit installation (RPM, manual binary placement). Wasm plugins SHALL be distributed via OCI pull as part of complypacks. The installation act for native plugins represents a higher trust decision than the OCI pull for Wasm plugins, and the capability boundary SHALL reflect this asymmetry.

#### Scenario: Native plugin installed via RPM
- **WHEN** a native plugin binary is placed in `/usr/libexec/complytime/providers/` or `~/.complytime/providers/`
- **THEN** it runs with full host access as a subprocess

#### Scenario: Wasm plugin arrives via complyctl get
- **WHEN** a Wasm plugin is extracted from a complypack during `complyctl get`
- **THEN** it runs sandboxed with only the three declared host functions
