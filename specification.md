# LightAPI Description Specification 0.1.0

## Status

This document defines the draft LightAPI Description Specification format version `0.1.0`.

The normative schema is [schema/lightapi.yaml](schema/lightapi.yaml). Example documents are in [examples](examples).

## Purpose

LightAPI describes API capabilities in a machine-readable and agent-readable format. A LightAPI document tells agents, live test runners, and workflow generators:

- what operations exist
- which protocol each operation uses
- what logical inputs are required
- how inputs map to wire-level requests
- what result shape and behavior to expect
- which outputs can be extracted for later calls
- which operations are safe, destructive, retryable, public, internal, or restricted
- how endpoint metadata should be progressively disclosed to agents

LightAPI is not a workflow engine. Executable orchestration belongs in the agentic workflow specification.

## Document Shape

A LightAPI document is a YAML or JSON document with this root shape:

```yaml
lightapi: 0.1.0
profile: catalog

info:
  title: Portal API Capability Pack
  namespace: light-portal
  version: 1.0.0

operations: {}
```

The `lightapi` field is the specification format version. The `info.version` field is the content version of the document.

The optional `profile` field identifies how the document is intended to be used:

- `catalog`: a multi-operation authoring and registry document.
- `endpoint`: a single endpoint consumption document, usually served by the portal to an agent or workflow.
- `bundle`: a small curated set of endpoints for a task, skill, or permission grant.

When omitted, `profile` defaults to `catalog`.

## Top-Level Fields

### `lightapi`

Required. Must be `0.1.0` for this draft.

### `info`

Required. Describes the capability pack.

Required fields:

- `title`
- `namespace`
- `version`

`namespace` provides the organizational scope for composition, portal publishing, and agent discovery. Operation identifiers only need to be unique within the document namespace.

`contact` follows the OpenAPI Contact Object convention:

```yaml
contact:
  name: Steve Hu
  url: https://lightapi.net
  email: steve.hu@lightapi.net
```

Custom contact extension fields are allowed.

### `profile`

Optional. Identifies whether the document is an API-level catalog, endpoint-level consumption document, or curated endpoint bundle.

```yaml
profile: endpoint
```

Agents and workflows should prefer endpoint or bundle documents for consumption. API owners and portal ingestion flows usually author catalog documents.

### `sources`

Optional. Defines external source documents and capability catalogs referenced by operations.

Supported source types:

- `openapi`
- `openrpc`
- `protobuf`
- `grpc-reflection`
- `mcp`
- `http`
- `lightapi`

Each source should declare a `version` for reproducible live tests. The `document` field can be a path, URL, or embedded protocol-specific document.

MCP sources must declare `transport`.

```yaml
sources:
  portalRpc:
    type: openrpc
    version: 0.1.0
    document: ./openrpc/portal-command.openrpc.json
    servers:
      local:
        endpoint: http://localhost:6881/rpc

  gatewayMcp:
    type: mcp
    version: 0.1.0
    document: ./mcp-capabilities.yaml
    transport: streamable-http
```

### `secrets`

Optional. Declares the secret names required by the document. Secret values must not be stored inline.

```yaml
secrets:
  - portalClientId
  - portalClientSecret
```

### `environments`

Optional. Defines named environment variables for server resolution and request construction.

Environments may inherit with `extends`.

```yaml
environments:
  local:
    variables:
      portalBaseUrl: http://localhost:6881
  dev:
    extends: local
    variables:
      portalBaseUrl: https://dev.portal.lightapi.net
```

When an agentic workflow imports a LightAPI catalog, it should select the environment explicitly.

```yaml
use:
  catalogs:
    portal:
      endpoint: ./lightapi.yaml
      version: 0.1.0
      environment: dev
```

### `authentications`

Optional. Defines reusable authentication policies. Field names should align with the agentic workflow authentication policy model where possible.

```yaml
authentications:
  portalCommandToken:
    type: oauth2
    authority: ${env.portalBaseUrl}
    grant: client_credentials
    client:
      id: ${secrets.portalClientId}
      secret: ${secrets.portalClientSecret}
```

Supported authentication types:

- `none`
- `basic`
- `bearer`
- `apiKey`
- `oauth2`
- `mtls`
- `custom`

### `operations`

Required. A map of operation id to operation definition.

An operation describes a single API capability. It can be invoked directly by a runner or converted into agentic workflow `call` and `assert` tasks.

All operations share common metadata:

```yaml
operations:
  registerApi:
    endpointId: light-portal/api.register
    protocol: jsonrpc
    summary: Register a new API.
    visibility: public
    lifecycle: active
    tags: [ api-admin, onboarding ]
    capability:
      group: api-management
      action: create
      resource: api
```

`endpointId` is the globally qualified identifier for an endpoint in the form `{namespace}/{operation-identifier}`. It serves as the portal storage key, cross-document reference, and agent permission grant. Example: `light-portal/api.register`.

`visibility` controls discovery:

- `public`
- `internal`
- `restricted`

`lifecycle` controls operational status:

- `draft`
- `active`
- `deprecated`
- `retired`

`capability.group`, `capability.action`, and optional `capability.resource` support structured agent discovery.

### `examples`

Optional. Defines reusable behavior examples. Examples should describe real behavior, not just payload shape.

### `fixtures`

Optional. Defines reusable data setup and teardown descriptions for live tests. Fixtures are not executable workflows by themselves; agentic workflows or runners execute them.

### `testSequences`

Optional. Defines linear API test sequences.

Test sequences are intentionally constrained:

- no branching
- no loops
- no human input
- no error-handling control flow

If a sequence needs branching, retry policy, or human decision-making, use `workflow.yaml`.

### `context`

Optional. Provides shared context inheritance for endpoint-level documents.

When a document has `profile: endpoint`, it typically contains a single operation with self-contained or inherited context. The `context` block allows it to inherit `authentications`, `environments`, `secrets`, and `sources` from a parent catalog without duplication.

```yaml
context:
  catalog: light-portal
  inherits: [ authentications, environments, secrets, sources ]
```

`catalog` identifies the parent catalog by name or path. `inherits` lists the top-level sections to resolve from the parent.

The runtime is responsible for resolving inherited context before evaluating expressions or executing operations.

### `components`

Optional. Defines reusable parameters, headers, request bodies, results, assertions, examples, fixtures, schemas, and security schemes.

### `agent`

Optional. Defines document-level agent skill metadata and progressive disclosure defaults.

## Operation Protocols

### HTTP

HTTP operations support both raw HTTP and OpenAPI-backed operations.

Raw HTTP requires `method` and `endpoint`.

```yaml
operations:
  healthCheck:
    protocol: http
    method: GET
    endpoint: ${env.gatewayBaseUrl}/health
    result:
      success:
        status: 200
```

OpenAPI-backed HTTP requires `operationId` and `source`.

```yaml
operations:
  getPet:
    protocol: http
    source: petstoreOpenApi
    operationId: getPetById
```

### JSON-RPC

JSON-RPC operations describe direct JSON-RPC 2.0 method invocation.

A JSON-RPC operation must have `method`. It must also have either `endpoint` or `source`. If `source` is provided, the endpoint can be resolved from the source server map.

```yaml
operations:
  registerApi:
    protocol: jsonrpc
    endpoint: ${env.portalBaseUrl}/rpc
    method: api.register
    request:
      params:
        name: ${input.name}
```

### OpenRPC

OpenRPC operations describe JSON-RPC methods backed by an OpenRPC source.

```yaml
operations:
  getUser:
    protocol: openrpc
    source: portalRpc
    method: user.get
    server: local
```

### gRPC

gRPC operations describe protobuf or reflection-backed method invocation.

```yaml
operations:
  reloadGateway:
    protocol: grpc
    source: controllerGrpc
    service: net.lightapi.controller.ControllerService
    method: ReloadConfig
    transport: grpc
```

Supported transports:

- `grpc`
- `websocket`

### MCP

MCP operations describe tools, resources, and prompts. Runtime MCP discovery is useful but not sufficient; LightAPI documents should also describe input schemas, result schemas, examples, errors, and behavior.

Tool:

```yaml
operations:
  petstoreGetPet:
    protocol: mcp
    server: gatewayMcp
    kind: tool
    name: petstore_getPet
```

Resource:

```yaml
operations:
  readGatewayConfig:
    protocol: mcp
    server: gatewayMcp
    kind: resource
    uri: config://gateway/current
```

Prompt:

```yaml
operations:
  generateApiTestPrompt:
    protocol: mcp
    server: gatewayMcp
    kind: prompt
    name: generate_api_test
```

## Input And Request

LightAPI separates logical input from wire-level request mapping.

`input` describes what an agent or workflow author must provide.

```yaml
input:
  schema:
    type: object
    required: [ name, category ]
    properties:
      name:
        type: string
      category:
        type: string
```

`request` describes how input values are placed into the protocol-specific request.

```yaml
request:
  params:
    name: ${input.name}
    category: ${input.category}
```

This distinction lets the same logical operation be used by interactive agents, test runners, and workflow generators.

## Result Description

`result` describes expected outcomes and output extraction.

```yaml
result:
  success:
    body:
      json:
        $.result.apiId:
          exists: true
    outputs:
      apiId: $.result.apiId
  failure:
    duplicateName:
      error:
        code: -32001
```

Result descriptions should be convertible into workflow `assert` tasks.

Supported assertion operators:

- `exists`
- `equals`
- `notEquals`
- `contains`
- `matches`
- `gt`
- `gte`
- `lt`
- `lte`
- `hasLength`
- `schema`
- `rule`

`equals`, `notEquals`, and `contains` accept any JSON value.

`hasLength` accepts either an exact integer or a range object.

```yaml
$.checks:
  hasLength:
    gte: 1
```

## Output Path Roots

Output extraction expressions use JSONPath. The `$` root is protocol-specific:

| Protocol | `$` Root |
|---|---|
| HTTP / REST | Full HTTP response: `$.status`, `$.headers`, `$.body` |
| JSON-RPC | JSON-RPC response object: `$.result`, `$.error`, `$.id` |
| OpenRPC | Same as JSON-RPC |
| gRPC | Protobuf response message represented as JSON |
| MCP | MCP response: `$.content`, `$.isError` |

Examples:

```yaml
outputs:
  accessToken: $.body.access_token
  apiId: $.result.apiId
  petName: $.content[0].text
```

## Expression Contexts

LightAPI expressions use the `${...}` form. The following context roots are reserved.

| Root | Meaning |
|---|---|
| `${env.*}` | Active environment variables |
| `${secrets.*}` | Runtime-provided secret values |
| `${input.*}` | Current operation, sequence, or fixture action input |
| `${request.*}` | Resolved request object |
| `${response.*}` | Raw protocol response envelope |
| `${result.*}` | Normalized operation result |
| `${outputs.*}` | Current operation extracted outputs |
| `${steps.<stepId>.outputs.*}` | Prior test sequence step outputs |
| `${fixtures.<fixtureName>.*}` | Fixture execution outputs |

Fixture setup and teardown outputs are addressed by action index:

```text
${fixtures.portalUser.setup[0].outputs.userId}
${fixtures.portalUser.teardown[0].outputs.cleanupCount}
```

The expression syntax is intentionally lightweight in `0.1.0`. Runtimes may support additional expression functions, but they must preserve these roots.

## Fixtures

Fixtures describe data setup and teardown.

```yaml
fixtures:
  portalUserWithRole:
    setup:
      - operation: createUser
        input:
          email: steve.hu@lightapi.net
    teardown:
      - operation: deleteUser
        input:
          userId: ${fixtures.portalUserWithRole.setup[0].outputs.userId}
```

SQL fixture actions must be marked `privileged: true` and should be scoped to explicit environments.

```yaml
fixtures:
  seedTestData:
    environments: [ local ]
    setup:
      - sql: ./fixtures/seed.sql
        connection: ${env.dbUrl}
        privileged: true
```

## Test Sequences

Test sequences are linear lists of operation calls.

```yaml
testSequences:
  onboardPetstoreApi:
    steps:
      - stepId: registerApi
        operation: registerApi
        input:
          name: Petstore
        outputs:
          apiId: $.result.apiId

      - stepId: createApiVersion
        operation: createApiVersion
        input:
          apiId: ${steps.registerApi.outputs.apiId}
```

For cross-document references, a step may use `endpointId` instead of `operation`:

```yaml
      - stepId: verifyTool
        endpointId: petstore-gateway/petstore_getPet
        input:
          petId: 1
```

Each step must specify either `operation` (local reference within the same document) or `endpointId` (globally qualified cross-document reference).

A test sequence step may reference a reusable result component:

```yaml
result:
  use: registerApiSuccess
```

Agentic workflow generators should map test sequence steps into `call` tasks followed by `assert` tasks.

## Progressive Disclosure

Agents should not load full operation details by default. LightAPI supports five disclosure levels.

| Level | Purpose |
|---|---|
| `index` | Discover operation ids, summaries, tags, capability groups, visibility, lifecycle |
| `summary` | Understand auth, safety, idempotency, inputs, available examples |
| `invocation` | Build the wire request: protocol, source, endpoint/server, request mapping |
| `behavior` | Understand results, examples, errors, edge cases |
| `full` | Load fixtures, test sequences, troubleshooting and support context |

Document-level defaults:

```yaml
agent:
  disclosure:
    defaultLevel: summary
    levels:
      index:
        include: [ operationId, summary, tags, capability, visibility, lifecycle ]
      invocation:
        include: [ protocol, source, request, endpoint, server ]
```

Operation-level disclosure:

```yaml
agent:
  disclose:
    level: summary
    includeByDefault: true
  skill:
    examples: [ happyPath ]
    relatedOperations: [ createApiVersion ]
```

## Portal Publishing

When LightAPI documents are managed by light-portal API endpoint admin, the portal acts as the publishing and discovery layer.

Recommended JSON-RPC query methods:

```yaml
method: lightapi.listOperations
params:
  namespace: light-portal
  level: index
  visibility: public
  lifecycle: active

method: lightapi.getOperation
params:
  namespace: light-portal
  operationId: registerApi
  level: invocation

method: lightapi.getCapabilityGroup
params:
  namespace: light-portal
  group: api-management
  level: summary
```

Agents should load `index` first, then deepen to `summary`, `invocation`, or `behavior` only for operations needed by the current task.

## Relationship To Agentic Workflow

LightAPI describes API capability contracts. Agentic workflow executes them.

LightAPI:

- operation catalog
- protocol details
- input schema
- request mapping
- result expectations
- examples
- fixtures
- test sequences
- progressive disclosure metadata

Agentic workflow:

- `call`
- `assert`
- `ask`
- branching
- loops
- retries
- scheduling
- state machine
- live test orchestration
- failure task assignment

In interactive mode, an agentic workflow may use LightAPI `input.schema` to decide what to ask the user. In live test mode, the workflow must use examples, fixtures, generated values, and secrets instead of user prompts.

## Versioning

`lightapi` is the format version. `info.version` is the document content version.

Rules for `0.1.0`:

- Consumers should reject documents with unsupported `lightapi` versions.
- Source versions should be pinned for reproducible test results.
- Workflow runs should record the LightAPI document version and resolved source versions.
- Minor format versions are expected to be backward-compatible once the specification reaches `1.0.0`.

## Conformance

A conforming LightAPI `0.1.0` document must validate against [schema/lightapi.yaml](schema/lightapi.yaml).

A conforming consumer should:

- resolve environment inheritance
- resolve source/server references
- resolve authentication references
- evaluate the reserved expression context roots
- enforce protocol-specific operation requirements
- apply result assertions consistently
- preserve standard assertion failure shape
- support progressive disclosure filtering

Cross-reference validation, such as checking that an operation `source` exists in `sources`, is a tooling/runtime responsibility and is not fully expressible in the JSON Schema.
