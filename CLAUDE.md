# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## The Vaultomic platform

Vaultomic is a digital time-capsule / memory-vault product. Users create **Vaults** — sealed
collections of photos, videos, voice notes, and letters addressed to a recipient — that unlock and
get delivered on a future date. Vaults can be shared with members, and plans gate storage/vault/
member limits (see `VAULTOMIC PLANS.txt`: Silver=free/1GB/1 vault/2 members, Gold=$7.99/100GB/
unlimited, Platinum=$24.99/1TB/unlimited/priority support).

The platform is split across three sibling repos (normally checked out as siblings):

- **`vaultomic-app`** — the Flutter client (mobile/web) end users install.
- **`vaultomic-sam`** (this repo) — the AWS-managed foundation: Cognito user pools, the shared
  DynamoDB single-table, an S3 bucket, and a Cognito user-provisioning Lambda.
- **`vaultomic-server`** — a separately-deployed (Docker/EC2, not SAM) Express/TypeScript API that
  serves `https://api.vaultomic.com/api/v1`, the actual REST backend `vaultomic-app` talks to. It
  reads the *same* Cognito pools and DynamoDB table this stack provisions, and does its own
  Cloudflare R2 storage — this repo's `StorageBucket`/`secrets.primefaya.com`/
  `public.primefaya.com` S3 buckets are legacy/secondary, not the primary media store.

**Cross-repo invariant**: this stack owns the source of truth for Cognito pool IDs, the DynamoDB
table name/key schema, and GSIs. `vaultomic-app` and `vaultomic-server` both hardcode copies of
these (pool IDs in `vaultomic-app/lib/settings/config.dart` and
`vaultomic-server/node/config/`). Any schema change here (new GSI, renamed pk/sk pattern, new
Cognito attribute) must be mirrored in both other repos.

## What this repo is

An AWS SAM (CloudFormation) template. **Currently an early-stage skeleton**: only Cognito auth +
one DynamoDB table + one S3 bucket + one Lambda trigger are defined. There is no
`AWS::Serverless::Api` / HTTP API / WebSocket API in `template.yaml` yet — despite the README
documenting a pattern for adding HTTP/WebSocket Lambdas (inherited boilerplate from a prior
project, "ReserveXpress" — sample names/URLs in the README referencing `ReserveXpress` are
generic examples, not live resources). Read `template.yaml` directly rather than trusting the
README's resource list, since the README is largely a how-to guide, not current-state
documentation.

> **Security note**: `README.md` contains a hardcoded example Cognito credential
> (`shukybadeer@gmail.com` / a plaintext password) in a `USER_PASSWORD_AUTH` curl example. Treat
> it as a real leaked credential — flag for rotation/redaction, don't reuse it, don't add more
> like it.

## Commands

```bash
./run.sh dev            # sam deploy -t template.yaml --config-env dev --profile default
./run.sh prod           # same, --config-env prod
./run_local.sh          # sam local start-api -t template.yaml --region us-east-1 --profile default
```

Deploys go to stacks `vaultomic-dev` / `vaultomic-prod` (region `us-east-1`, artifacts bucket
`vaultomic-sam-artifacts`, `CAPABILITY_IAM CAPABILITY_AUTO_EXPAND` — see `samconfig.toml`). CI
(`.github/workflows/pipeline.yaml`) auto-deploys on push to `dev_v1.0.0` → dev and `prod_v1.0.0`
→ prod; no test/lint step exists in the pipeline.

## Architecture (`template.yaml`)

- **Runtime**: Node.js 18.x, 128MB, 30s timeout for all functions (`Globals`), attached to two
  shared Lambda layers built from `lambda/layers/npm` and `lambda/layers/custom`.
- **Env vars** injected into every function: `STAGE`, `REGION`, `TABLE_NAME=vaultomic_${StageName}`,
  `S3_STORAGE_BUCKET=vaultomic-storage-${StageName}`, plus cross-account buckets
  `secrets.primefaya.com` / `public.primefaya.com` (Primefaya is the parent dev/company brand).
- **Cognito**: `CognitoUserPool` (`vaultomic-${StageName}`), email-as-username, custom `userName`
  attribute, SES send-from `support@vaultomic.com`. `CognitoUserPoolClient` uses OAuth code/
  implicit flows with callback scheme `comprimefayavaultomicwebviewcallback://` — this is the
  Flutter app's deep-link redirect, confirming this pool is what `vaultomic-app` authenticates
  against.
- **DynamoDB**: single table `vaultomic_${StageName}` (`AWS::DynamoDB::GlobalTable`,
  PAY_PER_REQUEST, TTL on `ttl` attr). Key: `pk`/`sk` (String). **PITR must be enabled manually**
  — it's not set in the template. GSIs:
  - `CognitoIdentifierGSI` (userCognitoId) — resolve internal user by Cognito sub.
  - `UserEmailGSI` (userEmail, KEYS_ONLY) — dedupe/lookup by email.
  - `VaultMembershiplGSI` (vaultMemberUserEmail + pk) — note the typo ("Membershipl") is
    load-bearing; don't silently "fix" it without updating every consumer.
  - `itemOwnershipGSI` (itemOwner + itemContext, ALL).
  - `itemAncestryGSI` (itemAncestor + itemContext, KEYS_ONLY) — query an item plus its
    descendants (hierarchical vault/folder/item structure).
- **S3**: `StorageBucket` = `vaultomic-storage-${StageName}`, private, CORS allows GET/PUT from
  any origin.
- **Lambda**: only `OnUserRegister` exists (`lambda/triggers/cognito/on-user-register/app.js`),
  wired as the Cognito `PostConfirmation` trigger, policy `AmazonDynamoDBFullAccess`. On
  confirmation it parses the Cognito sub/email/`custom:userName`/federated-IdP info and calls the
  shared `registerUser()` lib.

### `lambda/` layout

- `lambda/triggers/cognito/on-user-register/app.js` — the one live handler.
- `lambda/layers/custom/nodejs/node_modules/` — shared internal libs bundled as a layer:
  - `environment/index.js` — entity-type prefixes (`USER`, `PROFILE`, `COGNITO_DATA`, `INFO`,
    `NOTIFICATION_SETTINGS`), default plan `PLAN_SILVER`, default locale `LOCALE_EN`, GSI name
    constants.
  - `dynamodb/dynamo-entities/user/index.js` — `registerUser()` (dedupes via `UserEmailGSI`;
    generates a ULID user id; batch-writes profile/info + cognito-linkage + empty
    notification-settings records) and `peekUserByEmail()`.
  - `dynamodb/dynamo-batch-ops/index.js` — generic batch write/get helpers, 25-item chunks,
    exponential backoff, up to 7 retries.
- `lambda/layers/npm/nodejs/package.json` — layer deps: AWS SDK v3
  (`@aws-sdk/client-dynamodb`, `@aws-sdk/lib-dynamodb`), `ulid`. Compiled via `tsc` from
  `app.ts`.

### Adding a new Lambda

Follow the pattern in `template.yaml`'s existing `OnUserRegister` resource (CodeUri pointing into
`lambda/`, handler `app.lambdaHandler`, attach both shared layers) — the README's HTTP-API/
WebSocket examples are generic SAM patterns to adapt, not something already wired up here.

## Terminology note

Earlier commit history (`c5f8dd1 CapsuleMembershiplGSI`) shows this product was originally called
"Capsule" before renaming to "Vault" — if you see stray `Capsule*` naming anywhere, it's a rename
remnant, not a distinct concept.
