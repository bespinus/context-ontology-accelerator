# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Context Ontology Accelerator (COA): a semantic context layer on AWS. Knowledge graphs, OWL ontologies, and rules are served to AI agents. The pipeline runs **Scan → Model → Serve**: ingest sources, induce and manage ontologies and metrics, then answer queries through SPARQL-to-SQL federation (VKG) and MCP.

This repo is a public read-only mirror of an internal repo; PRs are not accepted upstream. Some directories (`docs/`, `packages/*/tests/integ/`, `tests/integ/load/`) are stripped, and the Makefile targets that use them print a warning instead of running.

## Commands

Prereqs: Python 3.12 (uv), Node 22+ (pnpm), Java 17 + Gradle (Smithy), Docker.

```bash
make setup        # runs `make generate` (Smithy codegen) then installs uv + pnpm deps
make generate     # regenerate smithy-generated/ — required after any change in models/
make format       # ruff format + ruff --fix + nx format
make lint         # version-check + `nx run-many -t lint` (ruff, ruff format --check, mypy per package)
make test         # nx unit tests for all projects + tests/unit + scripts/agents
make build        # regenerates NOTICE, then nx build
```

Single project or single test:

```bash
pnpm nx run control-plane:test                 # one package's unit suite (with coverage gate)
pnpm nx run control-plane:lint
uv run pytest packages/control-plane/tests/unit/test_foo.py::test_bar -m unit
uv run pytest tests/unit/test_sync_version.py  # repo-level cross-package tests
cd packages/web-app && pnpm vitest run src/path/to/file.test.tsx
pnpm --filter coa-infra exec jest test/services/api-stack.test.ts
pnpm nx run infra:synth
```

Connectors (Java/Maven, separate from Nx): `cd connectors && mvn -B test -pl example -am`, and for a connector's CDK stack `cd connectors/<id>/cdk && pnpm test`.

Python tests that import `coa_common.response` need `ALLOWED_ORIGIN` set (any value, e.g. `https://ci-test.example.com`). The module fails closed without it.

## Architecture

**API contracts start in Smithy.** `models/src/main/smithy/*.smithy` defines two services, `ControlPlaneService` and `DataLayerService`. `scripts/smithy-generate.sh` builds them into `smithy-generated/`, which is gitignored and must be generated before anything installs:
- TypeScript clients `@coa/control-plane-client` and `@coa/data-layer-client`, used by `web-app`
- OpenAPI specs, used by API Gateway in `infra` and by the external docs
- Pydantic models `coa_control_plane_server` / `coa_data_layer_server` (via openapi-generator), used by Python Lambda handlers to parse requests

These generated packages are members of both the uv workspace (`pyproject.toml`) and the pnpm workspace, so codegen has to run before `uv sync`/`pnpm install`. To change an API, edit Smithy, run `make generate`, then update handlers.

**Python services** live under `packages/<name>/src/coa_<name>/` with `tests/unit/`. Each is a uv workspace member depending on `libs/common` (`coa_common`: config, DynamoDB DAO, auth authorizers, Bedrock helpers, response/CORS helpers; `coa_authorization`: Cedar schema, policy evaluator, seed policies per role).
- `control-plane`: namespaces, roles, grants. Lambda handlers are split one per operation (`*_handler.py`).
- `sources`: database and document ingestion.
- `ontology-engine` (`coa_ontology`): ontology induction, stores, validation. Runs as a container.
- `metric-service` (`coa_metrics`): metric authoring and resolution.
- `vkg` (`coa_vkg`): Ontop 5.x wrapper for SPARQL→SQL translation only. It holds an H2 schema, not real data, and loads ontology/mappings from S3 at startup.
- `context-manager` (`coa_serve`): Serve-layer query orchestration with tiered resolution (`tier1/`, `tier2/`, `tier3/`), agents, upstream clients, and an SSE emitter.
- `mcp-server` (`coa_mcp`): MCP tools for agents. `mcp-proxy` is its TypeScript counterpart.
- `data-layer`: query, retrieval, and traversal APIs.

**TypeScript:** `packages/web-app` (React + Cloudscape + Vite, vitest, Playwright e2e), `libs/ts-shared` (`@coa/shared`), and `infra` (CDK).

**Infra** (`infra/bin/app.ts`): reads deploy config from the SSM parameter `/<prefix>/config` at synth time. It creates foundation stacks (`lib/stacks/foundation/`: network, storage, authnz, IdP, WAF, guardrail, web) and then service stacks (`lib/stacks/services/`), one per package plus the API, namespace, and serve stacks. Stack names are `<prefix>-<env>` (default `coa-dev`).

**Access model:** namespace isolation, with namespace-scoped roles (owner, maintainer, data-steward, data-analyst) and platform roles (`platform-admin`, `platform-viewer`), enforced through Cedar policies in `libs/common/src/coa_authorization`.

## Conventions enforced by tests/CI

- **Every Python unit test needs `@pytest.mark.unit`** (or `pytestmark`). Package test targets run with `-m unit`, so an unmarked test is silently deselected, and `tests/unit/test_unit_marker_coverage.py` fails on it. Tests that are not unit tests use `integ`, `slow`, or `security`.
- Per-package coverage gates are set in each `project.json` (`--cov-fail-under`, 80–85).
- **Versioning:** the repo-root `VERSION` is the single source of truth. Bump it, then run `make version`. Never edit a package manifest version by hand; `make lint` fails on drift.
- New `.py/.ts/.tsx/.js/.sh/.smithy` files need the Apache-2.0 header (`Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.` / `SPDX-License-Identifier: Apache-2.0`), checked by `scripts/ci/check-license-headers.sh`.
- Ruff: line length 120, Google-style docstrings required under `src/` (not in tests, demos, or scripts). mypy runs on each package's `src/`.
- `tests/unit` also checks that README/Makefile paths exist and that `external-docs/content/package-guide.md` matches the real packages. Update those docs when adding or renaming a package; the package guide describes the `pyproject.toml`/`project.json` layout a new package needs.
- `pnpm-workspace.yaml` has supply-chain settings (`minimumReleaseAge` in minutes, `trustPolicy`, scoped `overrides`). Read the inline comments there before changing anything.
- Nx `affected` has gaps (web-app has no `project.json`, and `smithy-generated` is not declared as a dependency). Use `run-many` to verify cross-cutting changes.

## Deploy

`make preflight`, `make deploy-dev`, and `make deploy-serve` run the `scripts/deploy*.sh` scripts against a real AWS account. `make destroy-dev` tears down all stacks (see the Makefile comments for wait-budget env vars). Deployment guides are in `external-docs/content/` (`getting-started.md`, `deploying.md`).
