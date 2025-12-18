---
title: "The small-project lane"
description: "multi-tenant deployments on AWS App Runner with Pulumi + GitLab OIDC"
categories: ["lza", "aws", "projects"]
tags:
  ["aws", "lza", "coding", "projects", "experimentation"]
showSummary: true
publishDate: 18. December 2025
---

A landing zone is usually discussed as an answer to scale: many accounts, clear boundaries, repeatable patterns.

But in real organizations, the opposite problem shows up just as often: **the long tail**.

Small services. Prototypes. Internal tools. A “quick thing” that quietly becomes production.

You *can* run a well-architected multi-account setup and still struggle with these projects, because the operational overhead of a full account stack doesn’t always match the size of the workload.

This post is about what we built for that gap: **apps-platform**, a paved road for small projects that still require serious guardrails.

---

## Our context

We already operate the classic structure:

- a deployment/automation account (“team-deployment”)
- separate environment accounts (dev/test/prod)

For large products, this remains the gold standard.

However, we also needed a solution for projects that are **too small to merit full account-per-project overhead**, while still meeting the same expectations:

- **strong identity binding** from CI to AWS
- **least privilege that stays least privilege**
- **no lateral movement** between services
- a workflow where onboarding is predictable and boring

---

## The challenge

The challenge is not “how do we deploy App Runner”.

It’s this:

> How do we let many GitLab repositories deploy into shared AWS accounts, without letting them touch each other’s resources?

If you’ve built platforms, you’ve likely seen the two traps:

- **Permission snowball**: every team needs “just one more permission”, and the deployment role slowly turns into admin.
- **Account explosion**: account-per-project works, but you pay with org complexity (guardrails, networking, identity, billing, automation, compliance) even for tiny services.

We wanted a third option: a shared runtime environment that is multi-tenant **by design**, where the guardrails are enforced by AWS itself.

---

## What we built

The apps-platform consists of two cooperating layers.

### 1) The base stack (privileged, per environment)

This repository (`apps-platform/base`) is a privileged Pulumi program that:

- provisions shared environment infrastructure in the apps-platform account
- onboards services by creating the minimum IAM + state scaffolding required for safe, per-repo deployments

### 2) The service stacks (unprivileged, per repository)

Each service repository owns its own Pulumi program that:

- creates the service resources (ECR, App Runner, optional Aurora, Secrets, DNS)
- runs with a per-service deployment role that is constrained to that service boundary

---

## How it works (the short version)

The design hinges on a tight trust chain and two enforcement mechanisms.

### The trust chain: GitLab OIDC → CI role → deployment role

1. GitLab CI requests an OIDC token.
2. The pipeline assumes a **repo-specific CI role** in the deployment account (`sts:AssumeRoleWithWebIdentity`).
3. That CI role can assume exactly one **service deployment role** in the apps-platform account (`sts:AssumeRole`).
4. Pulumi runs with that deployment role and creates/updates only the service’s resources.

### Enforcement #1: tags as tenant boundaries

We require mandatory tags on service resources:

- `platform=apps-platform`
- `app=<serviceSlug>`
- `env=<environment>`
- `managedBy=pulumi`

Deployment permissions are split:

- **Create** is allowed only with `aws:RequestTag/app == <serviceSlug>` and `aws:RequestTag/platform == apps-platform`.
- **Manage** is allowed only with `aws:ResourceTag/app == <serviceSlug>`.

### Enforcement #2: DNS scoping

Route53 is shared state, so we treat it as one.

We restrict record changes to:

- `${serviceSlug}.${zoneName}`
- `*.${serviceSlug}.${zoneName}`

---

## Useful code examples (builder pattern + rollout ideas)

The point of the platform isn’t that it’s theoretically secure. The point is that **the secure path is the easy path**.

These snippets show the style we push teams toward.

### Example 1: keep your `index.ts` as a service definition (builder-first)

The `AppsPlatformApp` builder is designed so the entrypoint reads like “here is my service”, not “here is a platform integration thesis”.

```ts
import * as pulumi from "@pulumi/pulumi";
import { apps_platform } from "@idg-cloud-platform/pulumi-components";
import { loadAppsPlatformContext } from "./context";

export = () => {
  const ctx = loadAppsPlatformContext();

  // Two-phase rollout: infra first, service only when image exists.
  if (!ctx.deployService) {
    return {
      serviceSlug: ctx.serviceSlug,
      ecrRepositoryUrl: ctx.imageRepositoryUrl,
      serviceUrl: pulumi.output(undefined),
      customDomainUrl: pulumi.output(undefined),
    };
  }

  const app = new apps_platform.AppsPlatformApp(ctx.serviceSlug, {
    serviceSlug: ctx.serviceSlug,

    // Shared environment primitives (provided as config)
    vpcId: ctx.vpcId,
    privateSubnetIds: ctx.privateSubnetIds,
    hostedZoneId: ctx.hostedZoneId,
    zoneName: ctx.zoneName,

    // Per-service IAM roles created by onboarding
    executionRoleArn: ctx.executionRoleArn,
    ecrAccessRoleArn: ctx.ecrAccessRoleArn,

    // Image to run
    imageRepositoryUrl: ctx.imageRepositoryUrl,
    imageTag: ctx.appImageTag,

    // Always tag consistently
    environment: ctx.environment,
    tags: ctx.tags,
  })
    .withWebService({
      port: "8080",
      healthCheckPath: "/healthz",
      runtimeEnvironmentVariables: { NODE_ENV: "production" },
    })
    // Optional capability: Aurora Postgres + secret wiring + VPC connector
    .withDatabase({
      dbName: ctx.dbName,
      dbUsername: ctx.dbUsername,
      dbPassword: ctx.dbPassword,
      serverlessMinCapacity: 0,
      serverlessMaxCapacity: 1,
    })
    // Optional capability: custom domain (DNS records are created for you)
    .withCustomDomain()
    .build();

  return {
    serviceUrl: pulumi.output(app.serviceUrl),
    customDomainUrl: pulumi.output(app.customDomainUrl),
    imageIdentifier: pulumi.output(app.imageIdentifier),
    connectionSecretArn: pulumi.output(app.connectionSecretArn),
  };
};
```

What this shows off:

- the **fluent builder pattern** keeps configuration readable
- optional capabilities (`withDatabase`, `withCustomDomain`) are additive
- the **two-phase rollout** is a single boolean gate

### Example 2: a tiny context loader (platform wiring lives here)

To keep `index.ts` clean, we load shared values (VPC/subnets/DNS/roles) from Pulumi config.

```ts
import * as pulumi from "@pulumi/pulumi";
import { apps_platform } from "@idg-cloud-platform/pulumi-components";

export interface AppsPlatformContext {
  environment: string;
  serviceSlug: string;

  // Shared infra
  vpcId: string;
  privateSubnetIds: string[];
  hostedZoneId: string;
  zoneName: string;

  // IAM scaffold (created by onboarding)
  executionRoleArn: string;
  ecrAccessRoleArn: string;

  // Image / rollout
  imageRepositoryUrl: pulumi.Output<string>;
  appImageTag: string;
  deployService: boolean;

  // Optional DB settings
  dbName: string;
  dbUsername: string;
  dbPassword: pulumi.Output<string>;

  // Required tags
  tags: Record<string, pulumi.Input<string>>;
}

export function loadAppsPlatformContext(): AppsPlatformContext {
  const environment = pulumi.getStack();
  const cfg = new pulumi.Config();

  const serviceSlug = cfg.require("serviceSlug");

  const vpcId = cfg.require("vpcId");
  const privateSubnetIds = cfg.requireObject<string[]>("privateSubnetIds");
  const hostedZoneId = cfg.require("hostedZoneId");
  const zoneName = cfg.require("zoneName");

  const executionRoleArn = cfg.require("executionRoleArn");
  const ecrAccessRoleArn = cfg.require("ecrAccessRoleArn");

  const appImageTag = cfg.get("appImageTag") ?? "latest";
  const deployService = cfg.getBoolean("deployService") ?? false;

  const dbName = cfg.get("dbName") ?? "appdb";
  const dbUsername = cfg.get("dbUsername") ?? "appuser";
  const dbPassword = cfg.requireSecret("dbPassword");

  // Central idea: enforce the platform’s tenant boundary via tags.
  const tags = {
    platform: "apps-platform",
    app: serviceSlug,
    env: environment,
    managedBy: "pulumi",
  };

  // Rollout helper: always create the ECR repo so CI can push before deploying the service.
  const repo = new apps_platform.apprunner.EcrRepository(`${serviceSlug}-ecr`, {
    serviceSlug,
    repositoryName: serviceSlug,
    environment,
    tags,
  });

  return {
    environment,
    serviceSlug,
    vpcId,
    privateSubnetIds,
    hostedZoneId,
    zoneName,
    executionRoleArn,
    ecrAccessRoleArn,
    imageRepositoryUrl: repo.repositoryUrl,
    appImageTag,
    deployService,
    dbName,
    dbUsername,
    dbPassword,
    tags,
  };
}
```

This is the “simple rollout” idea in code:

- infra can safely run even when the image isn’t there yet
- once CI pushes an image tag, the same program can deploy the service by flipping one config flag

### Example 3: onboarding a service is configuration, not a ticket

Onboarding into an environment is a small config change in the base stack:

```yaml
config:
  apps-platform:services:
    - slug: payments-api
      gitlabProjectPath: your-group/payments-api
      branchRestriction: protected
```

After merge, the base stack creates:

- CI role bound to `project_path`
- per-service deployment/execution/ECR roles
- per-service Pulumi state backend

That’s the paved road: you don’t negotiate IAM per repo; you provide identity and a slug.

---

## the apps-platform base stack

The base stack is where we make onboarding boring.

### Shared environment infrastructure (apps-platform account)

It provisions:

- environment hosted zone (Route53)
- shared VPC (public/private subnets + NAT)
- platform KMS key (for Secrets Manager)
- optional Transit Gateway attachment

### Service onboarding scaffolding (both accounts)

A service is onboarded via a Merge Request that appends an entry under `apps-platform:services` in the environment stack config:

```yaml
config:
  apps-platform:services:
    - slug: webservice-aurora-example
      gitlabProjectPath: intersport3/apps-platform/webservice-aurora-example
      branchRestriction: any
```

After merge, for each service we create:

- **Deployment account**
  - per-service Pulumi state backend (S3)
  - per-service KMS key for Pulumi secrets provider
  - per-repo GitLab CI role bound via OIDC

- **Apps-platform account**
  - per-service deployment role (runs Pulumi)
  - per-service execution role (App Runner runtime)
  - per-service ECR access role (App Runner build, plus push workflows)

---

## Why this is safe

This approach isn’t “trust people in a shared account”. It’s a layered model:

- **Repo identity binding**: OIDC trust conditions pin `project_path` (and optionally branch restrictions).
- **Role chaining**: a CI role can assume only its matching deployment role.
- **Tenant boundaries**: tag- and DNS-scoped policy conditions prevent cross-service access.
- **Runtime isolation**: execution roles can read only their own secrets (tag scoped) and decrypt only with the platform key.

Any single mechanism failing is not enough to create lateral movement.

---

## When you should *not* use this

This is the small-project lane — not a replacement for a full landing zone approach.

If you need strict compliance segmentation, different network per product, or materially different guardrail sets, you’ll still want account boundaries.

But when the overhead dominates the project, this pattern lets you keep speed **and** safety.

---

## Summary

We built apps-platform because we needed a pragmatic answer for small projects:

- Keep the classic multi-account setup for large products.
- Add a safe, low-overhead lane for the long tail.

The result is a multi-tenant App Runner platform where:

- GitLab OIDC binds CI identity to the repo.
- Role chaining limits who can deploy what.
- Tags and DNS scoping enforce tenant boundaries in AWS.
- Service onboarding is a boring MR, not a bespoke IAM exercise.

If you want to explore this approach in your own org, start by picking one small service and implementing the trust chain + tag boundaries end-to-end—then iterate based on what your CI and IaC tooling *actually* requires in practice.
