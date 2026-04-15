## ADDED Requirements

### Requirement: Plugin interface
The SDK SHALL expose a `Plugin` interface with three methods: `Describe(DescribeRequest) DescribeResponse`, `Generate(GenerateRequest) GenerateResponse`, and `Scan(ScanRequest) ScanResponse`. The types SHALL mirror the proto contract fields using JSON-serializable Go structs.

#### Scenario: Plugin author implements interface
- **WHEN** a Go struct implements all three methods of the Plugin interface
- **THEN** the struct compiles with TinyGo targeting WASI and produces a valid `.wasm` module

### Requirement: Registration function
The SDK SHALL provide a `Register(Plugin)` function that stores the plugin implementation and wires the exported Wasm functions (`describe`, `generate`, `scan`) to the implementation methods. Plugin authors SHALL call `Register` from `main()`.

#### Scenario: Plugin registers and exports functions
- **WHEN** `Register(&myPlugin{})` is called in `main()`
- **THEN** the compiled `.wasm` module exports `describe`, `generate`, and `scan` functions

### Requirement: HTTP host function wrapper
The SDK SHALL provide `HTTPRequest(HTTPRequestInput) HTTPResponse` that wraps the `http_request` host function. The wrapper SHALL handle memory pointer marshaling so plugin authors work with Go structs, not raw pointers.

#### Scenario: Plugin makes HTTP GET request
- **WHEN** a plugin calls `HTTPRequest` with method GET and a URL
- **THEN** the host executes the request and the plugin receives an `HTTPResponse` with status code, headers, and body

#### Scenario: HTTP request fails
- **WHEN** a plugin calls `HTTPRequest` and the host encounters a network error
- **THEN** the `HTTPResponse.Error` field contains the error message and `StatusCode` is zero

### Requirement: Log host function wrapper
The SDK SHALL provide `Log(level LogLevel, msg string)` that wraps the `log_message` host function. Log levels SHALL be: Trace, Debug, Info, Warn, Error.

#### Scenario: Plugin logs a message
- **WHEN** a plugin calls `Log(LogInfo, "checked 47 namespaces")`
- **THEN** the message appears in `.complytime/complyctl.log` prefixed with the plugin's evaluator ID

### Requirement: Variable read host function wrapper
The SDK SHALL provide `ReadVar(key string) (string, bool)` that wraps the `read_var` host function. The function SHALL return the variable value and a boolean indicating whether the key was found.

#### Scenario: Plugin reads an existing variable
- **WHEN** a plugin calls `ReadVar("api_server")` and the variable exists in the merged context
- **THEN** the function returns the value and `true`

#### Scenario: Plugin reads a missing variable
- **WHEN** a plugin calls `ReadVar("nonexistent")` and the variable does not exist
- **THEN** the function returns empty string and `false`

### Requirement: ScanRequest includes configurations
The `ScanRequest` type in the SDK SHALL include `Configurations []AssessmentConfiguration` and `GenerateState []byte` fields. Wasm plugins that skip Generate SHALL receive assessment configurations directly in Scan.

#### Scenario: Stateless plugin receives configs in Scan
- **WHEN** a Wasm plugin that does not implement Generate is invoked via Scan
- **THEN** `ScanRequest.Configurations` contains the assessment configurations extracted from the policy graph

### Requirement: GenerateResponse includes state blob
The `GenerateResponse` type SHALL include a `State []byte` field. Wasm plugins that use Generate SHALL return opaque state bytes that the host persists and passes back via `ScanRequest.GenerateState`.

#### Scenario: Plugin returns state from Generate
- **WHEN** a Wasm plugin returns a non-empty `State` in `GenerateResponse`
- **THEN** the host persists the state and includes it in the subsequent `ScanRequest.GenerateState`

#### Scenario: Plugin returns no state from Generate
- **WHEN** a Wasm plugin returns an empty `State` in `GenerateResponse`
- **THEN** `ScanRequest.GenerateState` is nil on subsequent Scan

### Requirement: TinyGo build compatibility
All SDK types and functions SHALL compile with TinyGo targeting `wasi`. The SDK SHALL NOT use packages unsupported by TinyGo (e.g., `net/http`, `os/exec`, full `reflect`).

#### Scenario: SDK compiles with TinyGo
- **WHEN** a plugin importing `pkg/wasmplugin` is built with `tinygo build -target wasi`
- **THEN** compilation succeeds and produces a valid `.wasm` module
