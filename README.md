# LiFi Intent Docs for LLM Agents

This repository contains a local copy of LI.FI Intent documentation for LLM agents. The goal is to keep the intent-specific docs close at hand for agent workflows, code generation, and offline reference without requiring web browsing.

## Source

The docs were assembled from the LI.FI documentation index:

https://docs.li.fi/llms.txt

From that index, the `/lifi-intents/` sitemap entries were deduplicated. Each page was downloaded as Markdown by appending `.md` to the docs path, for example:

```text
https://docs.li.fi/lifi-intents/introduction.md
```

The files were first mirrored under `lifi-intents/[...path]`, then moved into the repository root while preserving the section folders:

```text
architecture/
for-solvers/
intents-api/
knowledge-database/
mcp-server/
authentication.md
introduction.md
quickstart.md
```

## Assembly Prompt

The original prompt used to build this docs snapshot was:

```text
Download documentation for LiFi intents. Start with https://docs.li.fi/llms.txt, find menu sections starting with `/lifi-intents/`, deduplicate, download Markdown versions of the docs by appending `.md` to the path, put each under the appropriate path (`/lifi-intents/[...path]`).
```

## Snapshot

Last refreshed: 2026-11-09.

This snapshot includes all 32 unique `/lifi-intents/` pages listed in the current documentation index. Each Markdown page matches the website's downloaded Markdown, including Bitcoin, Solana, and Tron documentation. The previous local Bitcoin exclusions are no longer applied.

## Attribution

The original documentation is by LI.FI and is likely protected by LI.FI's copyright. This repository is a local copy assembled for reference.
