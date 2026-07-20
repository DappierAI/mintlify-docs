# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is the **Dappier API Guide** documentation site, built with [Mintlify](https://mintlify.com). Content is authored in `.mdx` files (Markdown + JSX components) and configured entirely through `docs.json`. There is no application code, build step, or test suite — `package.json` is empty and the repo publishes automatically via Mintlify's GitHub App when changes land on the default branch.

## Development

Requires the Mintlify CLI (`npm i -g mintlify`).

```bash
mintlify dev          # Local preview server at http://localhost:3000
mintlify broken-links # Validate internal links before committing
mintlify install      # Reinstall dependencies if `mintlify dev` won't start
```

There are no lint or test commands. Content correctness is verified by previewing locally and checking for broken links.

## Architecture

- **`docs.json`** is the single source of navigation and site config (Mintlify `docs.json` schema). Every page must be registered here under `navigation.tabs` or it will not appear in the site, even if the `.mdx` file exists. The site is organized into four tabs: **Home**, **API Reference**, **Integrations**, and **Cookbook**. Page paths in `docs.json` are relative and omit the `.mdx` extension.

- **Content lives as flat `.mdx` files** at the repo root (Home tab pages like `quickstart.mdx`, `sales-agent-mcp.mdx`) and in three directories:
  - `api-reference/endpoint/` — API reference pages
  - `integrations/` — SDK and platform integration guides
  - `cookbook/recipes/` — end-to-end tutorial recipes
  Adding a page means creating the `.mdx` file **and** adding its path to the correct group in `docs.json`.

- **API reference pages are OpenAPI-driven.** `api-reference/endpoint/openapi.json` is the OpenAPI 3.1 spec for the Dappier API. Reference `.mdx` pages render an endpoint by declaring it in frontmatter, e.g. `openapi: "post /app/aimodel/{ai_model_id}"` — the page body can be empty and Mintlify generates the interactive docs from the spec. To document a new endpoint, add it to `openapi.json` first, then create a thin `.mdx` page pointing at it.

- **Page frontmatter** uses YAML with a `title` (and optionally `openapi`). MDX bodies use Mintlify components like `<Steps>`, `<Step>`, `<Card>`, `<Accordion>`, and raw HTML.

- **Assets**: images in `images/`, brand logos in `logo/` (referenced with root-relative paths like `/images/foo.png`). Theme colors, analytics (GA4/GTM), navbar, and footer are all set in `docs.json`.

- **`DEVELOPER_DOCS_SOURCE.md`** is a source-of-truth reference doc (derived from the Sales Agent MCP server implementation) used as raw material for authoring the `sales-agent-mcp.mdx` end-user docs. It is not published navigation.

## Conventions

- Register new pages in `docs.json` and place them in the tab/group that matches their audience (Developers, Advertisers, Publishers, SDKs, Platform Integrations, or a Cookbook recipe group).
- Reference assets with root-relative paths (`/images/...`, `/logo/...`).
- Keep API reference pages thin — put the contract in `openapi.json`, not in prose.
