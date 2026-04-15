## ADDED Requirements

### Requirement: OCI manifest structure
A complypack SHALL be an OCI manifest containing Gemara content layers (existing media types) plus a plugin layer with media type `application/vnd.complypack.plugin.v1+wasm`. The OCI config blob SHALL use media type `application/vnd.complypack.config.v1+json` and contain the evaluator ID, pack version, declared capabilities, and WASI target.

#### Scenario: Complypack manifest contains all layers
- **WHEN** a complypack is pushed to an OCI registry
- **THEN** the manifest contains at least one Gemara content layer and exactly one Wasm plugin layer

#### Scenario: Content-only policy without Wasm layer
- **WHEN** a policy OCI artifact contains no layer with media type `application/vnd.complypack.plugin.v1+wasm`
- **THEN** the system treats it as a content-only policy (existing behavior, no plugin extraction)

### Requirement: Wasm layer extraction during get
`complyctl get` SHALL extract the Wasm plugin layer from cached OCI blobs to `~/.complytime/providers/{evaluator-id}.wasm` after a successful sync. The evaluator ID SHALL be read from the OCI config blob's `evaluator_id` field.

#### Scenario: Complypack synced for the first time
- **WHEN** `complyctl get` syncs a complypack that contains a Wasm layer
- **THEN** the Wasm blob is extracted to `~/.complytime/providers/{evaluator-id}.wasm` and the content layers are stored in the policy cache as usual

#### Scenario: Complypack updated to new version
- **WHEN** `complyctl get` syncs a complypack with a new digest
- **THEN** the extracted `.wasm` file is overwritten with the new version

#### Scenario: Content-only policy synced
- **WHEN** `complyctl get` syncs a policy with no Wasm layer
- **THEN** no file is written to the providers directory

### Requirement: Evaluator ID from OCI config
The evaluator ID for a complypack Wasm plugin SHALL be read from the OCI config blob (`evaluator_id` field), not derived from the policy URL or path. This evaluator ID SHALL match the filename used for the extracted `.wasm` file.

#### Scenario: Config specifies evaluator ID
- **WHEN** a complypack OCI config contains `{"evaluator_id": "kubernetes"}`
- **THEN** the extracted file is `~/.complytime/providers/kubernetes.wasm`

### Requirement: Backward-compatible policy references
`complytime.yaml` policy entries SHALL reference complypacks using the same `url` field as content-only policies. The system SHALL detect the presence of a Wasm layer from the OCI manifest, not from the workspace config.

#### Scenario: SRE references a complypack
- **WHEN** `complytime.yaml` contains `url: registry.example.com/complypacks/kube-cis:v1.0.0`
- **THEN** `complyctl get` fetches, caches content layers, and extracts the Wasm plugin — same `url` field, no additional config
