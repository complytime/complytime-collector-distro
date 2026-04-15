## ADDED Requirements

### Requirement: Module compilation and instantiation
The system SHALL compile and instantiate Wasm modules using wazero with WASI snapshot preview 1 support. Compilation SHALL validate the module is well-formed before any execution attempt.

#### Scenario: Valid module loads successfully
- **WHEN** a valid `.wasm` file exists in the providers directory
- **THEN** the system compiles the module without error and the module is available for Describe/Generate/Scan calls

#### Scenario: Invalid module rejected at load time
- **WHEN** a `.wasm` file contains corrupt or non-Wasm content
- **THEN** the system returns a compile error and does not attempt instantiation

### Requirement: Host function registration
The system SHALL register exactly three host functions in the `complyctl` namespace before module instantiation: `http_request`, `log_message`, and `read_var`. No WASI filesystem, socket, or process host functions SHALL be registered beyond the WASI snapshot preview 1 baseline required by wazero.

#### Scenario: Plugin calls registered host function
- **WHEN** a Wasm plugin invokes `http_request` during Scan execution
- **THEN** the host executes the HTTP request and returns the response to the plugin via shared memory

#### Scenario: Plugin attempts unregistered operation
- **WHEN** a Wasm plugin attempts a WASI operation not supported by the registered host functions (e.g., `fd_write` to an arbitrary file descriptor)
- **THEN** the operation returns an error or traps without granting access

### Requirement: Memory limits
The system SHALL enforce a maximum memory allocation for each Wasm module instance. The default limit SHALL be 256MB. The module instance SHALL trap if it exceeds the memory limit.

#### Scenario: Plugin within memory limit
- **WHEN** a Wasm plugin allocates memory within the 256MB limit during Scan
- **THEN** execution proceeds normally

#### Scenario: Plugin exceeds memory limit
- **WHEN** a Wasm plugin attempts to grow memory beyond the 256MB limit
- **THEN** the module traps and the system returns an error AssessmentLog for that evaluator

### Requirement: Timeout enforcement
The system SHALL propagate the complyctl command timeout (from `--timeout` flag or default) to Wasm module execution via context cancellation. A timed-out module SHALL be terminated and produce an error result.

#### Scenario: Plugin completes within timeout
- **WHEN** a Wasm plugin Scan completes before the deadline
- **THEN** results are returned normally

#### Scenario: Plugin exceeds timeout
- **WHEN** a Wasm plugin Scan does not complete before the deadline
- **THEN** the system cancels the context, terminates the module, and returns an error AssessmentLog with a timeout message and remediation guidance

### Requirement: Concurrent module isolation
The system SHALL instantiate each Wasm plugin in its own module instance. Execution of one Wasm plugin SHALL NOT affect the memory or state of another Wasm plugin running in the same complyctl process.

#### Scenario: Two Wasm plugins execute in the same scan
- **WHEN** two Wasm plugins are invoked during a single `complyctl scan`
- **THEN** each plugin operates on its own memory space and a failure in one does not crash the other
