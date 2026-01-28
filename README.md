# OpenCTI API Inventory (v6.9.10)

This repo provides a machine-readable inventory of the OpenCTI (Open Cyber Threat Intelligence) API surface area** for OpenCTI 6.9.10, intended as a practical companion for people building integrations, doing schema discovery, or documenting OpenCTI API behavior.

If you’re looking for OpenCTI API reference material, **GraphQL schema details, or a searchable snapshot of “what’s available” in a specific OpenCTI release, this is meant to be a useful starting point.

## What’s included

- A JSON-based “inventory” snapshot captured for **OpenCTI 6.9.10**
- A consistent, human-readable appendix-style representation derived from that JSON (for documentation / auditing / review).

## Quick start

1. Download or clone this repo.
2. Search and filter the JSON with tools like `jq`, `ripgrep`, or your editor.

Example workflows:
- Find names quickly: search for terms like `query`, `mutation`, `subscription`, `stix`, `report`, `indicator`, `identity`, `label`, etc.

## Why this exists

OpenCTI’s API (GraphQL + related behavior) is powerful but can be hard to navigate when you need:
- a version-pinned view of the API surface,
- a searchable catalog for integration planning,
- or an appendix you can ship alongside technical documentation.

This repo aims to keep the raw data (JSON) and the rendered appendix aligned and easy to reproduce.

## Notes / limitations

- This is a **snapshot for OpenCTI 6.9.10**. It may not match earlier or later OpenCTI releases.
- Treat OpenCTI’s official documentation and your running instance as the source of truth for runtime behavior and permissions.
