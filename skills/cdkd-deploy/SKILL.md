---
name: cdkd-deploy
description: Deploy, diff, bootstrap, inspect, roll back, or destroy AWS CDK applications with the community CDKD CLI while preserving safe production and state-management boundaries. Use when the user mentions cdkd or CDK Direct, asks for faster CDK deployments in development or test environments, or wants to evaluate whether an existing CDK app can use CDKD. Also use for CDKD import, export, state, events, and deployment troubleshooting. Prefer the upstream AWS CDK CLI for production and for existing CloudFormation-managed stacks unless the user explicitly approves a management migration.
---

# Deploy AWS CDK applications with CDKD

Use CDKD for fast development and test iteration without treating it as a transparent replacement for CloudFormation. CDKD calls AWS APIs directly and stores its own state in S3.

## Keep these boundaries

- Treat CDKD as a community project, not an AWS service or an AWS-supported CDK feature.
- Use CDKD for development and test by default. Use the upstream AWS CDK CLI for production and production-like staging. If the user explicitly requests CDKD there, state that the project describes itself as not production-ready and obtain explicit confirmation before changing AWS resources.
- Do not run `cdkd deploy` against an existing CloudFormation-managed stack as an implicit migration. Continue using `cdk deploy`, or discuss and explicitly approve `cdkd import --migrate-from-cloudformation` first.
- Do not migrate a CDKD-managed stack back to CloudFormation with `cdkd export` without explicit approval. Import and export change the system of record.
- Do not use `--no-wait` by default. Use it only when no subsequent step requires the resources to be ready, and verify readiness separately.
- Do not hand-edit or delete CDKD state objects. Use CDKD state and recovery commands.
- Do not assume the CDK bootstrap deployment role is sufficient. CDKD bypasses CloudFormation and needs direct permissions for every resource API it calls.
- Do not declare an application compatible with CDKD until synth, diff, dry-run, and resource-coverage checks have succeeded.

## Refresh volatile facts

CDKD changes quickly. Before installing, upgrading, migrating state, or relying on a flag:

1. Read the current project README and relevant reference page.
2. Query npm registry metadata for the published version and Node.js requirement:

   ```bash
   npm view @go-to-k/cdkd version engines --json
   ```

3. Compare the installed version with the registry version:

   ```bash
   cdkd --version
   ```

Do not auto-upgrade a working environment during an unrelated deployment. Pin an exact CDKD version in shared automation and CI.

Use these primary sources:

- Project and release guidance: <https://github.com/go-to-k/cdkd>
- CLI flags and wait semantics: <https://github.com/go-to-k/cdkd/blob/main/docs/cli-reference.md>
- Feature compatibility: <https://github.com/go-to-k/cdkd/blob/main/docs/supported-features.md>
- Resource coverage: <https://github.com/go-to-k/cdkd/blob/main/docs/supported-resources.md>
- State model and recovery: <https://github.com/go-to-k/cdkd/blob/main/docs/state-management.md>

## Follow the deployment workflow

### 1. Identify the target and current owner

Read repository instructions, `cdk.json`, package scripts, stack definitions, and deployment runbooks. Establish:

- environment class: development, test, staging, or production
- AWS profile, account ID, and region
- exact stack names
- current owner: CloudFormation/CDK, CDKD, another IaC tool, or unmanaged
- stateful resources and replacement risk

Before any AWS mutation, verify the effective identity:

```bash
aws sts get-caller-identity --profile <profile>
```

Pass the selected profile and region consistently. Never infer production safety from a stack name alone.

### 2. Check local prerequisites

Verify Node.js 20 or newer, the CDK app's normal synth command, and the installed CDKD version. If CDKD is absent, propose the official installation command and get approval before changing the user's global tool installation:

```bash
npm install --global @go-to-k/cdkd@<version>
```

Prefer an exact version when reproducibility matters.

CDKD uses its own bootstrap resources. Before the first `cdkd bootstrap`, show the resolved account and region and explain that the command creates CDKD state and asset storage in AWS. Do not replace or remove an existing CDK bootstrap stack merely because CDKD does not require it.

### 3. Prove compatibility before deployment

Use the app's existing dependency installation and synth workflow first. Then run CDKD's non-mutating checks, limiting stack-aware commands to an explicit stack:

```bash
cdkd synth
cdkd diff <stack>
cdkd deploy <stack> --dry-run
```

`cdkd synth` synthesizes the whole app and does not accept a stack selector. Limit the following diff, dry-run, and deploy commands to the intended stack.

Review the complete diff for replacements, deletions, IAM changes, security-group changes, and stateful resources. Confirm every resource type is supported by either a dedicated provider or the Cloud Control API fallback. Stop on unsupported resources or properties instead of adding broad compatibility bypasses.

If a stack with the same identity is already managed by CloudFormation and has not been imported into CDKD state, do not deploy it with CDKD.

### 4. Deploy the smallest explicit scope

Deploy named stacks instead of using `--all` unless the requested scope is genuinely every stack:

```bash
cdkd deploy <stack>
```

Use default wait behavior for ordinary development deployments. Choose deliberately when readiness semantics matter:

- use `--full-wait` when a smoke test or downstream step needs CloudFormation-like completion
- use `--no-wait` only when asynchronous completion is acceptable and a separate readiness check follows

Do not add `--yes`, `--force`, `--no-rollback`, unsupported-property bypasses, or concurrency overrides merely to make a failing command continue.

### 5. Verify the result

Treat only exit code `0` as full success. Exit code `2` is partial failure and requires investigation. Verify:

- CDKD state and deployment events
- expected outputs from CDKD state rather than assuming `aws cloudformation describe-stacks` contains them
- readiness of asynchronous resources, especially after `--no-wait`
- one focused functional smoke test when the application provides one

Useful inspection commands include:

```bash
cdkd state info
cdkd state show <stack>
cdkd events <stack>
```

Report the CDKD version, AWS account and region, stack, wait mode, and verification performed.

## Recover without hiding partial state

CDKD rolls back failed deployments by default. On failure:

1. Read the first meaningful resource error and `cdkd events <stack>`.
2. Inspect `cdkd state show <stack>` and any rollback journal before retrying.
3. Fix the underlying cause, then choose one recovery path: deploy forward, `cdkd rollback`, or destroy the failed development stack.
4. Treat exit code `2` as incomplete even if most resources were deleted or updated.

Use `cdkd force-unlock` only after proving that no deployment is still active. Never delete a lock merely because a command is slow.

## Handle destructive and ownership-changing commands separately

Before `destroy`, `state destroy`, `orphan`, `import`, `export`, rollback options that may delete resources, or any `--force` operation:

- resolve the exact stack and AWS account again
- explain which resources or state ownership will change
- check retention and snapshot behavior for stateful resources
- obtain explicit confirmation when the user's request did not already authorize that exact destructive or ownership-changing action

Prefer recoverable operations and preserve evidence needed to reconcile partial failures.
