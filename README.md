# Ace Data Cloud Facilitator Backend

![Deploy Status](../../actions/workflows/facilitator-deploy.yaml/badge.svg)

Facilitator Backend is an Ace Data Cloud service dedicated to running the **X402 payment facilitator**. It validates X402 authorizations, prevents replay attempts, and finalizes settlements on-chain by submitting `transferWithAuthorization` transactions. The service is deployed publicly at **https://facilitator.acedata.cloud**, operating as part of the Ace Data Cloud platform (**https://platform.acedata.cloud**).

## About X402

X402 is a programmable payments specification that allows a payer to sign a typed authorization (`TransferWithAuthorization`) instead of broadcasting a transaction directly. The facilitator receives the signed payload, verifies it under strict guardrails, and then settles the authorization on behalf of the payer. Key X402 guarantees:

- **Typed signing (EIP-712)** – the payload binds destination, amount, validity window, and nonce into a single signature.
- **Replay protection** – authorizations include a nonce; the facilitator stores it and rejects duplicates.
- **Programmable limits** – the requirements object can impose destination, asset, amount caps, and timeout policies.
- **Separation of concerns** – the facilitator handles chain submission so the originating application does not need custody of a hot wallet.

Ace Data Cloud relies on X402 to enable seamless settlement for digital asset purchases while preserving strong security boundaries between client applications and on-chain infrastructure.

## Capabilities

- **Authorization verification** (`POST /x402/verify`)
  - Validates payload structure and signature.
  - Confirms capped amounts, validity window, and recipient data.
  - Persists the authorization to prevent replays.
- **Settlement execution** (`POST /x402/settle`)
  - Re-validates the stored authorization.
  - Submits `transferWithAuthorization` using the configured signer key.
  - Waits for the receipt, records the transaction hash, and marks the authorization as settled.
- **Blockchain integration**
  - Works with any RPC endpoint provided through `X402_RPC_URL`.
  - Supports fixed `gasPrice` or EIP-1559 fee parameters.
- **Operations**
  - `/` and `/healthz` endpoints return JSON payloads for L7/LB readiness checks.
  - GitHub Actions workflow `facilitator-deploy` builds, pushes, and deploys the service whenever changes land on `main`.

## Configuration

All settings are driven by environment variables (see the provided `.env`). Variables prefixed with `X402_` directly control the settlement engine.

| Variable                                                                                       | Description                                         | Required | Default                      |
| ---------------------------------------------------------------------------------------------- | --------------------------------------------------- | -------- | ---------------------------- |
| `APP_ENV`                                                                                      | Runtime mode (`local`, `production`, …)             | No       | `local`                      |
| `APP_SECRET_KEY`                                                                               | Application secret key                              | Yes      | —                            |
| `PGSQL_HOST`, `PGSQL_PORT`,<br>`PGSQL_USER`, `PGSQL_PASSWORD`,<br>`PGSQL_DATABASE_FACILITATOR` | PostgreSQL connection details                       | Yes      | —                            |
| `X402_RPC_URL`                                                                                 | RPC endpoint used to submit settlement transactions | Yes      | —                            |
| `X402_SIGNER_PRIVATE_KEY`                                                                      | Facilitator signer private key                      | Yes      | —                            |
| `X402_SIGNER_ADDRESS`                                                                          | Explicit signer address (optional if derivable)     | No       | derived                      |
| `X402_GAS_LIMIT`                                                                               | Gas limit applied to settlement calls               | No       | `250000`                     |
| `X402_TX_TIMEOUT_SECONDS`                                                                      | Receipt wait timeout (seconds)                      | No       | `120`                        |
| `X402_MAX_FEE_PER_GAS_WEI`                                                                     | Maximum fee per gas (EIP-1559)                      | No       | `0` (fallback to `gasPrice`) |
| `X402_MAX_PRIORITY_FEE_PER_GAS_WEI`                                                            | Priority fee (EIP-1559)                             | No       | `0`                          |

> The facilitator trusts the caller to supply `pay_to`, `asset`, and `network` in the payload. Upstream services should maintain whitelists or policy checks to ensure only approved destinations/assets are processed.

## Local development

1. Install dependencies
   ```bash
   pip install -r <(poetry export -f requirements.txt --without-hashes)
   # or
   poetry install
   ```
2. Apply migrations
   ```bash
   python manage.py migrate
   ```
3. Start the server
   ```bash
   python manage.py runserver 0.0.0.0:8008
   ```

## Containers & Kubernetes

- **Docker** – `docker-compose build && docker-compose up` creates a local stack running `uvicorn core.asgi:application --host 0.0.0.0 --port 8000`.
- **Kubernetes** – `deploy/production/deployment.yaml` and `deploy/production/service.yaml` describe the production workload; `deploy/run.sh` applies them with the current build number. The GitHub Actions workflow (`.github/workflows/facilitator-deploy.yaml`) automates image build/push and deploy to the cluster.

## API reference

```http
POST /x402/verify
Content-Type: application/json

{
  "paymentPayload": { ... },
  "paymentRequirements": { ... }
}
```

Response:

```json
{ "isValid": true, "invalidReason": null, "payer": "0x..." }
```

```http
POST /x402/settle
```

Success:

```json
{
  "success": true,
  "transaction": "0xabc123...",
  "network": "base",
  "payer": "0x..."
}
```

Failures return `success: false` with `errorReason` describing validation issues, replays, RPC timeouts, or on-chain reverts.

## Repository layout

```
FacilitatorBackend/
├── core/          # Settings, URL routing, health endpoint
├── x402f/         # X402 business logic (models, views, tests)
├── deploy/        # Deployment manifests & scripts
├── Dockerfile
├── docker-compose.yaml
├── pyproject.toml
└── README.md
```

## Status & contact

- Public endpoint: **https://facilitator.acedata.cloud**
- Parent platform: **https://platform.acedata.cloud**
- Reach the team: [https://x.com/acedatacloud](https://x.com/acedatacloud)

For internal coordination, continue to follow Ace Data Cloud’s change-management and security processes when updating the facilitator or onboarding new callers.
