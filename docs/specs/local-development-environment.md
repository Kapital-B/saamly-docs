# Saamly local development environment

This specification records the local infrastructure design, runbook, and acceptance checks. For the product phases, see the [parent PRD](../prd/Saamly_PRD_Parent.md).

## Goal and scope

Create `kb/saamly-local`, a sibling of `saamly-service`, `saamly-web`, `saamly-admin`, and `saamly-mobile`. Use the operating model of `yonda/yonda-local`: Docker Compose owns shared infrastructure, and each application repository owns its own Terraform and runtime commands. A fresh checkout should reach a working consumer and admin flow through the real API, with data surviving a normal Compose restart.

This specification records the implemented local stack and its acceptance criteria. Cloud Lambda adapters compile and have focused tests; deployment to a real AWS account has not been exercised here. Image parsing still requires a separately configured vision model.

The local stack was verified on 2026-09-26: Floci started healthy, Terraform created the table, bucket, and queues and reapplied without resource changes, the host API passed `/healthz` and `/readyz`, the host worker consumed an SQS job, a presigned S3 upload worked, and a DynamoDB item survived `docker compose down` and `up`. Both Vite apps built and proxied real API requests. Go tests and local/dev Terraform validation passed. Flutter was not run because the Flutter SDK is unavailable here. Race-enabled Go tests require a C compiler, which is not installed on this host.

| Owner | Runs in Compose | Runs on the host |
| --- | --- | --- |
| `saamly-local` | Floci and LocalCloud Console | `docker compose` and smoke checks |
| `saamly-service` | Nothing | Terraform apply, API, SQS worker, seed commands |
| `saamly-web` | Nothing | Vite on `localhost:35174` |
| `saamly-admin` | Nothing | Vite on `localhost:35175` |
| `saamly-mobile` | Nothing | Flutter on a simulator or device |

The local Terraform target deploys AWS **resources** to Floci. It does not deploy the API or worker as Lambda functions. The API listens on host port `38080`; the worker polls the Floci SQS queue as a separate host process. Web and admin development servers proxy to the host API. Production and dev Terraform retain their cloud Lambda/API Gateway path.

## Starting point addressed by this implementation

- `yonda-local/docker-compose.yml` supplies the reference pattern: Floci, a persistent named volume, and a local console. Its Floci image is pinned to `1.6.0` and uses account `000000000000`.
- `saamly-service/docker-compose.yml` started `floci/floci:latest` with memory storage. The persistent stack now lives in `saamly-local`; the service Compose file remains an isolated CI fixture.
- `saamly-service/infra/environments/local` originally called the cloud `api` module and required Lambda zip files. It now provisions only the data and queue resources used by host processes.
- The service's local DynamoDB mode used to select an in-memory blob store in both `cmd/api` and `cmd/worker`. DynamoDB mode now uses Floci S3 for uploads. Local mail intentionally uses the log mailer.
- The cloud Terraform module defines Lambda functions. `cmd/api-lambda` and `cmd/worker-lambda` now provide Lambda entry points alongside the host commands. A real AWS deployment remains to be exercised.
- `saamly-web` defaults to MSW. Its Vite proxy targets `127.0.0.1:38080`; the admin app has the same proxy and a local admin bypass. Real-service startup must disable each mock explicitly.
- The local environment uses `af-south-1` in service config and Terraform. Keep that region for Saamly even though the Yonda stack uses `us-east-1`.

## Target infrastructure

`saamly-local/docker-compose.yml` contains Floci and LocalCloud Console. It follows the Yonda volume and emulator settings, pins `floci/floci:1.6.0`, and sets `FLOCI_DEFAULT_REGION=af-south-1`, `FLOCI_DEFAULT_ACCOUNT_ID=000000000000`, `FLOCI_STORAGE_MODE=hybrid`, and `FLOCI_STORAGE_PERSISTENT_PATH=/app/data`. A named volume is mounted at `/app/data`. The AWS edge is published on host `127.0.0.1:34576:4566`; the console is published on `127.0.0.1:38011:80`. The Compose health check probes Floci, and the console depends on it. Docker's socket is not mounted because local API and worker execution does not require Floci Lambda execution. PostgreSQL, OpenSearch, and Jaeger are not required by Saamly.

Reserve ports `34576`, `38011`, `38080`, `35174`, and `35175`. `saamly-local` should have its own Compose project and named volume, separate from `yonda-local`. Yonda keeps port `4566`, so both stacks can run concurrently. Pin and review image upgrades, and provide `up`, `down`, `status`, and `reset` commands. `down` preserves data; `reset` explicitly removes the named volume and Terraform state after a warning.

## Terraform changes in `saamly-service`

Keep `infra/environments/local` as the single owner of Saamly resources in Floci. Keep its local state outside version control and require dummy credentials plus an explicit endpoint override for **every** AWS provider service it uses. Give local resources the `saamly-local` prefix so they do not collide with other stacks on the same emulator account.

1. Reuse the `data` module for the DynamoDB table (including `GSI1`) and S3 uploads bucket. Verify that Floci supports each configured table and bucket feature. If lifecycle, PITR, encryption, or bucket policy operations are unsupported, gate only those features for the local target and preserve cloud behavior.
2. Extract the capture queue and DLQ from the cloud `api` module into a small reusable messaging module, or provide an equivalent local-only module. Keep the queue URL and ARN as outputs. The local target must create the queue without Lambda, API Gateway, IAM execution roles, EventBridge schedule, or event source mapping. Keep the dev/cloud path functionally equivalent after any refactor.
3. Decide whether the `identity` module's SSM secret is useful in local. If retained, output it as sensitive and feed the same value to both host processes. Local SES domain verification is unnecessary because local email is logged. Do not expose the secret in checked-in files.
4. Remove `module.api` from the local root and remove local requirements for `api_zip_path`, `worker_zip_path`, and `scripts/ensure-lambda-zips.sh`. Keep the Lambda module for cloud environments.
5. Output `dynamodb_table`, `uploads_bucket`, `capture_queue_url`, and a host environment export or generated, ignored env file. Set `AWS_ENDPOINT_URL=http://127.0.0.1:34576`, `AWS_REGION=af-south-1`, dummy credentials, `SAAMLY_ENV=local`, and `SAAMLY_STORE=dynamodb`. Avoid evaluating untrusted Terraform output as arbitrary shell code; if using `env_exports`, treat it as a trusted local artifact or replace it with a narrowly scoped env writer.
6. Make `./scripts/local.sh apply` depend on the external `saamly-local` stack being healthy. It must no longer start the service repository's Compose file or create placeholder Lambda zip files. `tf-local-destroy` should remove only Saamly resources, and reapply must be idempotent.

Use separate Terraform state for this Floci target. A developer with an old local state containing Lambda/API Gateway resources should follow a documented one-time migration: back up the state, destroy using the old configuration while the old emulator is reachable, then initialize and apply the new root. If that cannot be done, reset the isolated Saamly Floci volume and local state together. Never point this migration at a real AWS account.

## Worker execution boundary

Extract worker setup and the `capture.parse` / `core.purge` message handler from `cmd/worker` into reusable Go code. Keep the handler's contract as one message body plus context, returning an error on failure. The local `cmd/worker` remains a long-running process: it long-polls SQS, calls that handler, deletes only successful messages, and lets failed messages become visible again or enter the DLQ. It uses `AWS_ENDPOINT_URL` to reach Floci.

The cloud worker needs a separate Lambda entry point using the Go Lambda runtime. Its event source mapping receives an SQS batch from AWS; the handler calls the same message handler for each record and returns partial batch failures for records that failed. The Lambda handler must not start the poll loop or delete SQS messages itself. AWS owns polling, successful-message deletion, retries, and DLQ delivery in that mode. Package this entry point for `provided.al2023`; point the existing cloud Terraform function at that artifact. Test both entry points against the same message fixtures and verify batch failure behavior.

Likewise, the cloud API needs a Lambda HTTP adapter for API Gateway v2 that invokes the same router as the host HTTP server. The existing `cmd/api` listener can remain the local entry point. This cloud runtime work is a separate deliverable from provisioning local Floci resources; successful `terraform apply` alone does not prove the present Lambda zips can serve requests.

## Host runtime and client wiring

Add one documented host environment setup in `saamly-service`, shared by `go run ./cmd/api` and `go run ./cmd/worker`. The API must use DynamoDB, S3 uploads, and the capture SQS queue in local mode. Change the current `cfg.Env == "local"` blob-store branch so local Floci mode uses S3 while the in-memory `make dev` mode keeps its in-memory blob store. Keep the log mailer for local magic links. Set `SAAMLY_DEV_AUTH_BYPASS=true`, `SAAMLY_SEED_DEV_USER=true`, and `SAAMLY_ENTRA_ALLOWLIST=dev@saamly.local` only in local mode. Keep the session secret stable across API restarts, either from the local identity output or an ignored local env file.

The worker must poll the same queue URL and table as the API. For text and URL capture, it needs no extra infrastructure. Image parsing needs a reachable local OpenAI-compatible model endpoint and a configured vision model; document that as an optional developer dependency, outside Compose. Scheduled purge is not required for the first local stack; expose the existing one-shot `SAAMLY_RUN_PURGE=1` command in the runbook.

For web and admin, run `VITE_API_MOCK=false npm run dev` from their own repositories, with `VITE_DEV_API_PROXY=http://127.0.0.1:38080`. Web uses `VITE_SAAMLY_FLAVOUR=local`; admin uses `VITE_DEV_ADMIN_BYPASS=true` and `VITE_DEV_ADMIN_EMAIL=dev@saamly.local`. Do not write these flags into a production build. The browser accesses the API through each Vite proxy, so normal local browser flows do not need API CORS. For Flutter, document platform-specific API and presigned S3 URLs: `localhost` on a desktop or iOS simulator, the Android emulator's host alias, and a LAN-reachable host address for a physical device. Ensure the signed S3 URL hostname is reachable by that client before claiming mobile upload support.

## Developer runbook to deliver

The final `saamly-local/README.md` should give the following sequence, with commands that match the implemented targets:

```bash
# Terminal 1: persistent infrastructure
cd /path/to/kb/saamly-local
docker compose up -d --wait

# Terminal 2: provision Saamly resources in Floci
cd /path/to/kb/saamly-service
./scripts/local.sh apply

# Terminals 3 and 4: use the same generated local environment
cd /path/to/kb/saamly-service
./scripts/local.sh api
cd /path/to/kb/saamly-service
./scripts/local.sh worker

# Terminal 5: consumer client
cd /path/to/kb/saamly-web
npm ci
VITE_API_MOCK=false npm run dev

# Terminal 6: admin client
cd /path/to/kb/saamly-admin
npm ci
VITE_API_MOCK=false npm run dev
```

The runbook states prerequisites (Docker Compose v2, Terraform, Go, Node, and project dependencies), exact expected health endpoints and URLs, how to read log mail, how to seed taxonomy if automatic loading is unavailable, how to stop processes without deleting data, and how to reset all local state deliberately. The in-memory `make dev` path remains separate from `./scripts/local.sh api`.

## Acceptance checks

1. From a clean checkout, start Compose, apply Terraform, and launch API, worker, web, and admin with the documented commands. No manual AWS resource creation or Lambda packaging is needed.
2. Check the Floci health endpoint, Terraform outputs, and `GET http://127.0.0.1:38080/healthz`. Confirm table, bucket, and queue exist at the configured Floci endpoint and account.
3. Sign in through web's local-user path, create a household or inventory item through the real API, restart API and `docker compose down`/`up`, then verify the data remains. A `down -v` reset is the only normal operation that removes emulator data.
4. Submit a capture job; confirm the host worker consumes it and updates the draft. For an image job, verify the browser PUT goes to Floci S3 and the worker reads the same object. Test with a configured local vision model.
5. Open admin with the local bypass and confirm a real API request succeeds. Confirm both front ends have MSW disabled.
6. Re-run `terraform apply` and check for no changes. Validate local and dev Terraform; ensure the dev/cloud API and worker deployment plan remains intact.
7. Verify a reset and reapply recover a working stack, and that `saamly-local` can be stopped without touching `yonda-local` resources.
8. Before claiming cloud runtime deployment, invoke the API Lambda through API Gateway and test an SQS batch with one failing record. Verify the successful record is acknowledged and the failing record is retried.

## Delivery order

1. Add `saamly-local` Compose, README, health check, and persistent volume.
2. Decouple the service's local Terraform root from cloud Lambda/API Gateway resources; apply and validate it against the new stack.
3. Wire and document host API and worker env, including S3 upload behavior and stable session signing.
4. Document real-service client startup and run the acceptance checks. Record emulator-specific limitations and the tested Floci version in the runbook.
