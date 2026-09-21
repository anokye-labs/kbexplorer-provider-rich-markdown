# Rich-Markdown provider architecture

This repository is deliberately small: the filesystem and host-facing wrapper live in [`src/providers/rich-markdown.js`](../src/providers/rich-markdown.js), while deterministic parsing and graph extraction live in the pure library at [`src/lib/rich-markdown.js`](../src/lib/rich-markdown.js). The goal is a single Markdown document → one `KBGraph` fragment with explicit identity, typed relationships, and provenance metadata, without leaking filesystem or network concerns into the extractors.

## Contract with KBX Core and Engine

The public entrypoint is the default export from [`src/providers/rich-markdown.js`](../src/providers/rich-markdown.js). It is a `defineProvider()` factory, not a renderer or app shell:

```mermaid
flowchart LR
  Host[Host / Engine / CLI] --> Config[config.yaml or provider config]
  Config --> Factory[defineProvider default export]
  Factory --> Resolve[provider.resolve(context)]
  Resolve --> Load[loadSource(options)]
  Load --> Ingest[ingestRichMarkdown]
  Ingest --> Fragment[KBGraph fragment { nodes, edges }]
  Fragment --> Merge[Host merges / renders / indexes]
```

The actual contract is exercised by the test suite in [`tests/providers/rich-markdown.test.js`](../tests/providers/rich-markdown.test.js):

- `apiVersion` is exported and must match `PROVIDER_API_VERSION` from core.
- `capabilities` is exported as `Object.freeze(['graph:nodes', 'graph:edges'])`.
- `checkProviderCompatibility(...)` is used to assert that a host supports the required graph affordances.

The provider declares the exact capability set it requires from the host engine:

- `graph:nodes`
- `graph:edges`

That is defined by the exported constant in [`src/providers/rich-markdown.js`](../src/providers/rich-markdown.js), and the tests confirm the compatibility gate.

## Public API and export surface

The package exports two entrypoints in [`package.json`](../package.json):

| Specifier | Public contract |
|---|---|
| `.` | `defineProvider()` default export, `apiVersion`, `capabilities`, `resolveIdentityOptions`, `RichMarkdownError`, `RichMarkdownErrorCode` |
| `./lib` | pure library: `ingestRichMarkdown`, `parseRichFrontmatter`, `extractEmbeddedBlocks`, `extractLinkEdges`, `makeSourceRef`, `RICH_MARKDOWN_BLOCK_LANGS` |

The root module is the provider boundary and intentionally stays thin. The library module is deterministic and side-effect free.

### Root provider export contract

From [`src/providers/rich-markdown.js`](../src/providers/rich-markdown.js):

- `default` is the factory returned by `defineProvider((config = {}) => { ... })`
- `apiVersion` is `PROVIDER_API_VERSION`
- `capabilities` is frozen and exposes the graph affordances required by the host
- `resolveIdentityOptions(providerOptions = {}, kbConfig = {})` calculates the effective identity configuration
- `RichMarkdownErrorCode` defines `RICHMD_NO_SOURCE` and `RICHMD_READ_FAILED`
- `RichMarkdownError` wraps provider misconfiguration and IO failures

### Pure library export contract

From [`src/lib/rich-markdown.js`](../src/lib/rich-markdown.js):

- `RICH_MARKDOWN_BLOCK_LANGS` is the fixed set `['dot', 'mermaid', 'ics', 'canvas']`
- `parseRichFrontmatter(raw)` splits frontmatter from body and returns `{ ok, frontmatter, body, raw, bodyOffset }`
- `extractEmbeddedBlocks(body, opts?)` yields block descriptors with `lang`, `contentHash`, `span`, `contentSpan`, `lines`, and `sourceRef`
- `extractLinkEdges(body, opts?)` yields link facts and decides whether an edge should be emitted
- `makeSourceRef({ source, span, contentHash })` creates the interim provenance pointer for the not-yet-released core provenance contract
- `ingestRichMarkdown(input)` creates the single-node fragment plus typed edges

## Provider registration and discovery behavior

The provider factory builds a runtime instance with a stable identity and a `resolve()` method:

```js
const richMarkdownProvider = defineProvider((config = {}) => {
  const options = config.options ?? {};
  const id = options.id ?? (config.name ? `rich-markdown-${slugify(config.name)}` : 'rich-markdown');
  return {
    id,
    name: config.name ?? 'Rich-Markdown Provider',
    requiredAffordances: [],
    async resolve(context = {}) { ... },
  };
});
```

This is the registration seam a host uses to load a provider without direct imports into the engine. The module exposes only the provider contract; host integration is external to this repository.

The provider accepts configuration through the `config` object and `options` block:

- `config.name` becomes the provider display name
- `config.cluster` and `options.cluster` set the cluster on the node
- `options.path` loads a file from disk when present
- `options.content` allows inline Markdown for hermetic tests or callers without a filesystem
- `options.sourcePath` labels the document when `content` is supplied inline
- `options.cwd` resolves `path` relative to a caller-specified working directory
- `options.scheme`, `options.authority`, and `options.sourceAuthorityKey` override identity addressing
- `options.entityType` overrides the document's declared entity type if present

When no usable source exists, it throws `RichMarkdownError` with `RICHMD_NO_SOURCE` or `RICHMD_READ_FAILED` as defined in [`src/providers/rich-markdown.js`](../src/providers/rich-markdown.js).

## Identity and configuration precedence

Addressing behavior is intentionally explicit and layered. `resolveIdentityOptions(...)` resolves in this order:

1. provider `options` values (`scheme`, `authority`)
2. `config.identity.sourceAuthorities[sourceAuthorityKey]`
3. `config.identity.authority`
4. `config.identity.scheme`
5. core defaults (`kg://`, no authority)

The function is a thin precedence layer; the actual canonical identity is minted later by `buildAddress(...)` in the library.

## Content representation and interpretation

The content interpretation path is deterministic and mostly local to [`src/lib/rich-markdown.js`](../src/lib/rich-markdown.js):

```mermaid
flowchart TD
  Raw[Markdown source text] --> Frontmatter[parseRichFrontmatter]
  Frontmatter --> Body[body text]
  Body --> Blocks[extractEmbeddedBlocks]
  Body --> Links[extractLinkEdges]
  Frontmatter --> Connections[frontmatterConnections]
  Body --> Title[deriveTitle / identityBody]
  Title --> Node[ingestRichMarkdown -> single KBNode]
  Blocks --> Node
  Links --> Node
  Connections --> Node
  Node --> EdgeSet[KBEdge[]]
```

### Frontmatter and node payload

`parseRichFrontmatter(raw)` normalizes line endings to `\n`, extracts YAML-like frontmatter, and preserves arbitrary key/value data under `node.data` instead of flattening to a known schema. The parser intentionally supports typed scalars, inline flow collections, block sequences, nested maps, and block scalar values. It is not a full general YAML parser; it is a deterministic subset meant to preserve arbitrary author data without losing structure.

The resulting node includes:

- `id`: local slug / graph key (`identityBody`)
- `title`: first heading or best-effort fallback
- `cluster`: assigned cluster id
- `identity`: canonical address from `buildAddress(...)`
- `source`: source metadata (`path` or `readme` placeholder)
- `sourceFile`: `{ path, raw, format: 'markdown' }` when a file path is known
- `data`: the full frontmatter plus `richMarkdown` metadata
- `entityType` and `jsonld` when an entity type is declared

The node is intentionally distinct from the canonical identity: `node.id` is the local slug, while `node.identity` is the cross-provider address. This distinction is called out in the tests and in the code comment above `ingestRichMarkdown()`.

### Embedded blocks

`extractEmbeddedBlocks(body, opts)` scans fenced code blocks and retains only `dot`, `mermaid`, `ics`, and `canvas`. Each block is stored with:

- `lang`
- `content`
- `contentHash` from `contentHashOf(...)`
- `span` and `contentSpan` for document offsets
- `lines` for source line numbers
- `sourceRef` created via `makeSourceRef(...)`

This is a contract-oriented abstraction for downstream consumers that need to inspect code-like assets without re-parsing full Markdown.

### Links and relation mapping

`extractLinkEdges(body, opts)` masks fenced and inline code, then scans for Markdown links using the `LINK_RE` pattern. It records:

- raw `href`
- parsed attributes from `{rel=...}`
- `relation` mapped via `mapRelation(...)`
- `isAddress` via `isAddress(...)`
- `emitEdge` flag deciding whether the fact becomes a graph edge

The edge rule is deliberate:

- annotated links with `rel=...` emit an edge
- links whose target is an address (for example `kg://target`) emit an edge
- relative or non-address links are recorded under `node.data.richMarkdown.links` for provenance only

This keeps the provider from creating dangling edges from plain relative links such as `./other.md`.

## Output flow to rendering and viewer consumers

The provider never renders itself. Its contract is the return object from `resolve()`:

```js
return {
  nodes: [node],
  edges,
};
```

This is the handoff boundary to the host engine and any downstream viewer. The repository’s contract is specifically a `KBGraph` fragment: one document becomes one node with typed edges. The renderer or app shell is not part of this package; the provider does not own a UI or any presentation layer.

In other words:

- the provider resolves raw Markdown into graph facts
- the host engine merges those facts into a larger graph
- viewer/rendering surfaces consume node metadata and edge relations, but they are outside this package’s scope

The tests confirm this contract by asserting that `resolve()` yields `nodes.length === 1` and the expected `edges` relation values for inline and frontmatter links.

## Constraints and extension points

This provider has several intentionally narrow constraints:

- It ingests exactly one document at a time.
- It only captures the four embedded block languages listed in `RICH_MARKDOWN_BLOCK_LANGS`.
- Its YAML handling is deterministic but not a full YAML 1.2 implementation.
- It stores provenance in the interim `sourceRef` shape until the formal core provenance contract lands.
- It emits graph facts only for node/edge relationships that are explicitly addressable or annotated.

The extension points are explicit in code:

- add a new embedded language by extending `RICH_MARKDOWN_BLOCK_LANGS`
- change the `parseFrontmatterBlock` subset if a new authoring pattern is required
- adjust relation normalization by changing `mapRelation(...)` consumption in `extractLinkEdges(...)`
- swap `makeSourceRef` for the eventual core `SourceRef` contract when the formal provenance seam is released

## Compatibility expectations

The compatibility expectation is simple and enforced by code:

- the provider version must match the core provider API version (`apiVersion === PROVIDER_API_VERSION`)
- the host must provide at least `graph:nodes` and `graph:edges`
- `checkProviderCompatibility()` is the canonical gate used in [`tests/providers/rich-markdown.test.js`](../tests/providers/rich-markdown.test.js)

This repository does not add a runtime build or custom bundling step. It ships pure ESM JS with Node 22+ and relies on the existing `kbexplorer-core` contract rather than introducing another layer.

## Related references

- [`src/providers/rich-markdown.js`](../src/providers/rich-markdown.js)
- [`src/lib/rich-markdown.js`](../src/lib/rich-markdown.js)
- [`docs/rich-markdown-provider.md`](./rich-markdown-provider.md)
- [`tests/providers/rich-markdown.test.js`](../tests/providers/rich-markdown.test.js)
- [`tests/lib/rich-markdown.test.js`](../tests/lib/rich-markdown.test.js)
