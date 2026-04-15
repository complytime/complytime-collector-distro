## ADDED Requirements

### Requirement: Two-stream OCI model
The system SHALL support two distinct OCI artifact streams: Gemara content (compliance-authored standards and guidelines) and complypacks (engineering-authored Wasm plugin bundled with policy-as-code assessment content). These streams SHALL be distinguishable by their OCI config media type.

#### Scenario: Gemara content artifact
- **WHEN** an OCI artifact has a config media type other than `application/vnd.complypack.config.v1+json`
- **THEN** the system treats it as Gemara content (existing behavior)

#### Scenario: ComplyPack artifact
- **WHEN** an OCI artifact has config media type `application/vnd.complypack.config.v1+json`
- **THEN** the system treats it as a complypack and performs Wasm extraction in addition to content caching

### Requirement: ComplyPack artifact structure
A complypack SHALL be a single OCI artifact containing both PaC assessment content layers (using existing Gemara media types) and a Wasm plugin layer (`application/vnd.complypack.plugin.v1+wasm`). The config blob SHALL use media type `application/vnd.complypack.config.v1+json` and contain the `evaluator_id`, plugin `version`, declared `capabilities`, and `wasi_target`.

#### Scenario: ComplyPack contains content and plugin
- **WHEN** a complypack OCI manifest is inspected
- **THEN** it contains at least one PaC content layer (Gemara media types) and exactly one Wasm plugin layer

#### Scenario: ComplyPack config carries evaluator metadata
- **WHEN** a complypack config blob is read
- **THEN** it contains `evaluator_id`, `version`, `capabilities`, and `wasi_target` fields

### Requirement: Complypack handling during get
`complyctl get` SHALL detect complypacks by config media type. When a complypack is detected, the system SHALL cache all layers in the policy cache (same as content-only policies) and additionally extract the Wasm plugin layer to `~/.complytime/providers/{evaluator-id}.wasm`. The evaluator ID SHALL be read from the config blob.

#### Scenario: SRE pulls a complypack
- **WHEN** `complyctl get` processes a URL pointing to a complypack
- **THEN** the system caches all layers in `~/.complytime/policies/{id}/` and extracts the Wasm module to `~/.complytime/providers/{evaluator-id}.wasm`

#### Scenario: SRE pulls a content-only Gemara policy
- **WHEN** `complyctl get` processes a URL pointing to a standard Gemara content artifact
- **THEN** existing behavior applies — content is cached, no plugin extraction occurs

### Requirement: Evaluator ID from complypack config
The evaluator ID for a complypack Wasm plugin SHALL be read from the OCI config blob (`evaluator_id` field). This evaluator ID SHALL determine the extracted filename (`{evaluator-id}.wasm`) and the routing key for scan dispatch.

#### Scenario: Config specifies evaluator ID
- **WHEN** a complypack OCI config contains `{"evaluator_id": "kubernetes"}`
- **THEN** the extracted file is `~/.complytime/providers/kubernetes.wasm` and scan routes `kubernetes` evaluator requests to this module

### Requirement: Backward-compatible policy references
`complytime.yaml` policy entries SHALL reference complypacks using the same `url` field as content-only Gemara policies. The system SHALL detect complypack vs content-only from the OCI config media type, not from the workspace config syntax.

#### Scenario: SRE references a complypack
- **WHEN** `complytime.yaml` contains `url: registry.example.com/complypacks/kube-cis:v1.0.0`
- **THEN** `complyctl get` detects the complypack, caches content, and extracts the plugin — no additional config syntax required

### Requirement: Complypack update handling
When `complyctl get` syncs a complypack with a new digest, the system SHALL re-cache all content layers and re-extract the Wasm module, overwriting the previous `.wasm` file.

#### Scenario: Complypack updated to new version
- **WHEN** `complyctl get` syncs a complypack with a new digest
- **THEN** the content cache is updated and the `.wasm` file is overwritten with the new module

### Requirement: Authorship boundary
Gemara content and complypacks SHALL be independently publishable and versionable. A complypack version change SHALL NOT require republishing Gemara content, and a Gemara content update SHALL NOT require republishing a complypack.

#### Scenario: Plugin bug fix without content change
- **WHEN** an engineering team fixes a scanner bug and publishes a new complypack version
- **THEN** the Gemara content artifacts in the registry are unaffected

#### Scenario: Standards update without plugin change
- **WHEN** a compliance team publishes updated Gemara guidelines
- **THEN** existing complypacks in the registry are unaffected
