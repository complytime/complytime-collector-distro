## ADDED Requirements

### Requirement: Wasm runtime check
`complyctl doctor` SHALL include a `wasm-runtime` check when at least one Wasm plugin is discovered. The check SHALL verify that the wazero runtime initializes without error. The check SHALL be blocking (failure prevents exit code 0). The check SHALL be hidden when no Wasm plugins are present.

#### Scenario: Wasm runtime available with Wasm plugins present
- **WHEN** `complyctl doctor` runs and at least one `.wasm` file exists in the providers directory
- **THEN** the output includes a `wasm-runtime` check with pass status

#### Scenario: Wasm runtime check hidden when no Wasm plugins
- **WHEN** `complyctl doctor` runs and no `.wasm` files exist in the providers directory
- **THEN** no `wasm-runtime` check appears in the output

### Requirement: Complypack integrity check
`complyctl doctor` SHALL include a `pack/{id}` check for each policy whose OCI manifest is a complypack (config media type `application/vnd.complypack.config.v1+json`). The check SHALL validate: (1) complypack is cached in policy store, (2) extracted `.wasm` file exists in providers directory, (3) extracted file digest matches the Wasm layer descriptor, (4) module compiles in wazero, (5) module exports `describe`, `generate`, and `scan` functions.

#### Scenario: Complypack passes all integrity checks
- **WHEN** `complyctl doctor` validates a complypack with a valid extracted Wasm module
- **THEN** the `pack/{id}` check passes with message including the short digest

#### Scenario: Wasm file not extracted
- **WHEN** `complyctl doctor` validates a complypack but the `.wasm` file is missing from providers directory
- **THEN** the `pack/{id}` check fails with message "wasm module not extracted" and remediation "run complyctl get"

#### Scenario: Digest mismatch
- **WHEN** `complyctl doctor` validates a complypack but the extracted file digest does not match the OCI layer descriptor
- **THEN** the `pack/{id}` check fails with message "digest mismatch" and remediation "run complyctl get --force"

#### Scenario: Module fails compilation
- **WHEN** `complyctl doctor` validates a complypack but the `.wasm` file fails wazero compilation
- **THEN** the `pack/{id}` check fails with the compile error and remediation "contact pack maintainer"

#### Scenario: Module missing required exports
- **WHEN** `complyctl doctor` validates a complypack but the compiled module is missing one or more of the required exports
- **THEN** the `pack/{id}` check fails listing the missing exports and remediation "contact pack maintainer"

### Requirement: Provider type in doctor output
The `provider/{id}` check SHALL report the plugin type (native or wasm) and access level alongside health status. Native plugins SHALL show "native, full host access". Wasm plugins SHALL show "wasm, sandboxed".

#### Scenario: Doctor reports native provider
- **WHEN** `complyctl doctor` checks a native provider
- **THEN** the output shows "healthy (v{version}, native, full host access)"

#### Scenario: Doctor reports Wasm provider
- **WHEN** `complyctl doctor` checks a Wasm provider
- **THEN** the output shows "healthy (v{version}, wasm, sandboxed)"

### Requirement: Policy complypack indicator
The `policy/{id}` check SHALL append "(complypack)" when the policy URL resolved to a complypack composition manifest. The indicator distinguishes policies with bundled plugin logic from content-only policies.

#### Scenario: Policy is a complypack
- **WHEN** `complyctl doctor` checks a policy that was resolved from a complypack manifest
- **THEN** the version output includes "(complypack)" (e.g., "v1.0.0 (latest, complypack)")

#### Scenario: Policy is content-only
- **WHEN** `complyctl doctor` checks a policy pulled directly as Gemara content
- **THEN** the version output does not include "(complypack)"

### Requirement: Verbose complypack detail
`complyctl doctor --verbose` SHALL expand Wasm provider and complypack checks to show: OCI origin URL, full digest, module size, exported functions, imported host functions, and WASI target.

#### Scenario: Verbose output for Wasm provider
- **WHEN** `complyctl doctor --verbose` reports on a Wasm provider
- **THEN** the expanded output includes type, access list, source path, and OCI origin

#### Scenario: Verbose output for complypack
- **WHEN** `complyctl doctor --verbose` reports on a complypack
- **THEN** the expanded output includes full digest, module size, exports, and imports

### Requirement: Doctor check ordering with Wasm
Doctor checks SHALL execute in dependency order: config → cache → wasm-runtime → provider/{id} → policy/{id} → pack/{id} → registry/{host} → variables/{id}. The `wasm-runtime` check SHALL gate Wasm provider and pack checks — if it fails, those checks SHALL be skipped with a message indicating the dependency.

#### Scenario: Wasm runtime fails, dependent checks skipped
- **WHEN** `complyctl doctor` runs and the wasm-runtime check fails
- **THEN** all `provider/{id}` checks for Wasm plugins and all `pack/{id}` checks are skipped with "skipped — wasm runtime unavailable"
