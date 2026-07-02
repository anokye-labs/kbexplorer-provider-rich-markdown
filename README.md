# @anokye-labs/kbexplorer-provider-rich-markdown

A loadable [kbexplorer](https://github.com/anokye-labs) **provider** that ingests
a single rich Markdown document into a `KBGraph` fragment — one node plus its
typed edges. It is pluggable infrastructure in the same category as an iCal or
dot-graph reader: any kbexplorer surface (the CLI, the template/website SPA, the
future canvas) loads it through the `config.yaml` → `defineProvider()` seam.

It depends only on the shared-contracts package
[`@anokye-labs/kbexplorer-core`](https://github.com/anokye-labs/kbexplorer-core)
(`v0.3.0`). The filesystem boundary lives in the provider; the parsing/extraction
core is pure (no I/O, no network, no LLM — identical input → byte-identical
output).

## Install

```bash
npm install github:anokye-labs/kbexplorer-provider-rich-markdown#v0.1.0
```

## Exports

| Specifier | What it is |
|---|---|
| `@anokye-labs/kbexplorer-provider-rich-markdown` | The `defineProvider()` factory (default export) plus `apiVersion`, `capabilities`, `resolveIdentityOptions`, `RichMarkdownError`, `RichMarkdownErrorCode`. This is what a host loads. |
| `@anokye-labs/kbexplorer-provider-rich-markdown/lib` | The pure ingestion library: `ingestRichMarkdown`, `parseRichFrontmatter`, `extractEmbeddedBlocks`, `extractLinkEdges`, `makeSourceRef`, `RICH_MARKDOWN_BLOCK_LANGS`. |

## Usage

As a loaded provider (the host supplies config and calls `resolve()`):

```js
import provider from '@anokye-labs/kbexplorer-provider-rich-markdown';

const instance = provider({
  name: 'My Docs',
  cluster: 'docs',
  options: { path: 'docs/intro.md' },
});
const { nodes, edges } = await instance.resolve({ config: kbConfig });
```

Directly against the pure library (no filesystem):

```js
import { ingestRichMarkdown } from '@anokye-labs/kbexplorer-provider-rich-markdown/lib';

const fragment = ingestRichMarkdown({ content: markdownString, cluster: 'docs' });
```

See [`docs/rich-markdown-provider.md`](docs/rich-markdown-provider.md) for the
full configuration reference and
[`docs/samples/rich-markdown-sample.md`](docs/samples/rich-markdown-sample.md)
for a worked example.

## Develop

```bash
npm ci
npm test   # node --test, via scripts/run-tests.js
```

No build step: the package ships pure ESM JavaScript directly from `src/`.

## License

MIT
