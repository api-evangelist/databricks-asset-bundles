---
name: deploy-a-bundle
description: >-
  Validate, rehearse, deploy, run and tear down a Databricks Declarative
  Automation Bundle (formerly Databricks Asset Bundles) against a named target,
  without double-firing a job or destroying something you cannot get back.
api: Databricks Asset Bundles (Declarative Automation Bundles)
surface: cli
operations:
  - databricks bundle init
  - databricks bundle validate
  - databricks bundle plan
  - databricks bundle deploy
  - databricks bundle run
  - databricks bundle summary
  - databricks bundle destroy
  - databricks bundle sync
generated: '2026-09-05'
method: generated
source: https://docs.databricks.com/aws/en/dev-tools/cli/bundle-commands
---

# Deploy a Databricks bundle

There is no HTTP API for this product. The whole surface is the `databricks`
CLI reading a single `databricks.yml` at the project root. Every command below
is documented in the Databricks CLI bundle command reference; none is inferred.

## Before you start

- Authenticate first. Attended work uses OAuth U2M (`databricks auth login`,
  which writes a configuration profile); CI uses OAuth M2M with a service
  principal. Never put a credential in `databricks.yml` — select the profile
  with `-p`/`--profile` or pin it on the target under
  `targets.<name>.workspace.profile`.
- Know your target. `-t dev` and `-t prod` are different deployment modes with
  different safety behavior (see below).

## The flow

1. **Scaffold** (new project only): `databricks bundle init` and pick a
   template.
2. **Validate**: `databricks bundle validate` — checks configuration syntax.
   Cheap, non-mutating, run it on every edit.
3. **Rehearse**: `databricks bundle plan -t <target>` — "show deployment plan
   without making changes". Narrow it with `--select` when you only care about
   one resource. **Do not skip this on production.**
4. **Deploy**: `databricks bundle deploy -t <target>`. Re-running with unchanged
   configuration converges to the same workspace state rather than creating
   duplicates, so a retried deploy is safe. A deployment lock serialises
   concurrent deploys against one target; `--force-lock` overrides it and should
   be a deliberate act, not a reflex.
5. **Run**: `databricks bundle run -t <target> <resource>` to execute a job,
   pipeline or app.
6. **Inspect**: `databricks bundle summary` for what identity and resources the
   bundle currently owns; `databricks bundle open` to jump to a resource in the
   workspace.

## Rules that keep this safe

- **`deploy` is replay-safe. `run` is not.** Deploy converges on declared state;
  running twice runs the work twice. If a `run` appears to hang, check the
  workspace for an in-flight run before firing it again.
- **Rehearse a file sync** with `databricks bundle sync --dry-run` before
  `--full`.
- **`destroy` is a delete.** `databricks bundle destroy` deletes previously
  deployed jobs, pipelines and artifacts. Databricks publishes no window inside
  which a destroy can be undone, so treat it as permanent; the recovery path is
  re-deploying the bundle from version control, which restores the resources but
  not their run history or in-place state.
- **Do not paper over a production guardrail.** In production mode Databricks
  validates that the current Git branch matches the branch declared on the
  target. `--force` bypasses that check — if you need it, the bundle or the
  branch is wrong, not the check.
- **Development mode is not a smaller production.** It prefixes resource names
  with `[dev <user>]`, pauses all schedules and triggers, enables concurrent job
  runs and disables the deployment lock. Never read a dev deploy as evidence
  that a schedule works.

## Grounding

- Command surface and flags: `cli/databricks-asset-bundles-cli.yml`
- Idempotency, dry-run and reversibility verdicts:
  `conventions/databricks-asset-bundles-conventions.yml`
- Authentication chain: `authentication/databricks-asset-bundles-authentication.yml`
- Configuration schema (first-party, 917KB):
  `json-schema/databricks-asset-bundles-bundle-jsonschema.json`, or run
  `databricks bundle schema`.
