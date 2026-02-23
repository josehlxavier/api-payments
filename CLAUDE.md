# CLAUDE.md — API Pagamentos (Open Finance Brasil)

## Repository Overview

This repository contains the official OpenAPI specification for the **API de Iniciação de Pagamentos** (Payment Initiation API) of Open Finance Brasil. It is a **specification-only repository** — there is no application code, build system, or runtime. The single source of truth is `swagger.yaml`.

## Repository Structure

```
api-payments/
├── swagger.yaml                        # OpenAPI 3.0.0 specification (main artifact)
├── README.md                           # Minimal description (Portuguese)
└── .github/
    └── workflows/
        ├── trigger-ci.yml              # CI trigger on PR open/sync
        └── trigger-tag-control.yml     # Tag trigger on PR merge
```

## The Specification File

**File**: `swagger.yaml`
**Standard**: OpenAPI 3.0.0
**Current version**: `5.0.0-beta.1` (under `info.version`)
**Language**: Portuguese (Brazilian)

All changes to the API must be made exclusively in this file. The version field at `info.version` must be kept consistent with the changes being proposed.

### API Endpoints

| Method | Path | Operation ID | Security |
|--------|------|-------------|----------|
| `POST` | `/consents` | `paymentsPostConsents` | OAuth2ClientCredentials (`payments`) |
| `GET` | `/consents/{consentId}` | `paymentsGetConsentsConsentId` | OAuth2ClientCredentials (`payments`) |
| `POST` | `/pix/payments` | `paymentsPostPixPayments` | OAuth2AuthorizationCode or NonRedirectAuthorizationCode |
| `GET` | `/pix/payments/{paymentId}` | `paymentsGetPixPaymentsPaymentId` | OAuth2ClientCredentials (`payments`) |
| `PATCH` | `/pix/payments/{paymentId}` | `paymentsPatchPixPaymentsPaymentId` | OAuth2ClientCredentials (`payments`) |

### Security Schemes

- **OAuth2ClientCredentials** — Used for consent creation/query and payment query/cancellation.
- **OAuth2AuthorizationCode** — Used for payment initiation with user redirect flow. Requires scopes: `openid`, `consent:consentId`, `payments`.
- **NonRedirectAuthorizationCode** — Used for payment initiation without redirect (NRP flow). Requires scopes: `openid`, `enrollment:enrollmentId`, `payments`, `nrp-consents`.

### Key Enumerations

**Payment Status (`EnumPaymentStatusType`)**:
| Value | Meaning |
|-------|---------|
| `RCVD` | Received — request accepted, pending validation |
| `ACCP` | Accepted Customer Profile — ready for settlement |
| `ACPD` | Accepted Clearing Processed — submitted to settlement |
| `ACSC` | Accepted Settlement Completed — payment completed |
| `PDNG` | Pending — held for analysis |
| `SCHD` | Scheduled — scheduled successfully |
| `CANC` | Cancelled — cancelled before confirmation |
| `RJCT` | Rejected — rejected by holder or SPI |

**Local Instrument (`EnumLocalInstrument`)**:
| Value | Meaning |
|-------|---------|
| `MANU` | Manual account data entry |
| `DICT` | Manual Pix key entry |
| `QRDN` | Dynamic QR code (non-NFC) |
| `APDN` | Dynamic QR code via NFC |
| `QRES` | Static QR code (non-NFC) |
| `APES` | Static QR code via NFC |
| `INIC` | Pre-agreed creditor (ITP-contracted) |

For scheduled (non-single) payments, only `MANU`, `DICT`, or `QRES` are permitted.

**Payment Type (`EnumPaymentType`)**: `PIX` (only current value)

### Request/Response Format

All request and response bodies use `application/jwt` (JWS-signed payloads) as the content type. Responses for errors use `application/json; charset=utf-8`.

All requests and responses must be signed following the JWS protocol defined in the Open Finance Brasil security guide.

### Standard Headers

Every endpoint requires:
- `Authorization` — Bearer token
- `x-fapi-auth-date`
- `x-fapi-customer-ip-address`
- `x-fapi-interaction-id` — UUID RFC4122 for request/response correlation (must be mirrored by the holder)
- `x-customer-user-agent`
- `x-idempotency-key` (on write operations)

Response header `x-v` indicates the API version implemented by the financial institution.

## CI/CD Workflows

### `trigger-ci.yml` — PR Validation

**Trigger**: `pull_request_target` on `opened` or `synchronize` events targeting `main`.

**What it does**:
1. Builds the raw GitHub URLs for both the new (`head`) and old (`base`) `swagger.yaml`.
2. Captures PR metadata (title, description, URL, author).
3. Dispatches a workflow in the external repo `OpenBanking-Brasil/CI-CD-OFB` (workflow: `pr-validation-and-jira.yml`), passing the swagger URLs and PR metadata.

**Secret required**: `REPO_ACCESS_TOKEN` — must have permission to dispatch workflows on the `CI-CD-OFB` repository.

### `trigger-tag-control.yml` — Post-Merge Tagging

**Trigger**: `pull_request_target` on `closed` event, only when the PR is merged.

**What it does**:
1. Checks out the repository.
2. Reads the `version` and `title` fields from `swagger.yaml` using shell commands.
3. Dispatches a workflow in `OpenBanking-Brasil/CI-CD-OFB` (workflow: `tag-control.yml`), passing the repository name, base branch, version, title, and PR number.

**Secret required**: `REPO_ACCESS_TOKEN`.

## Development Workflow

### Branch Naming

Feature branches follow this convention based on the Jira ticket:

```
feat-PSV<ticket-number>
```

Examples from history:
- `feat-PSV399`
- `feat-PSV398`
- `feat-PSV397`

Fix branches use:
```
fix/<short-description>
```

### Commit Message Format

```
[GT de Serviços] PSV-<number> - Payments - v<version>: API Pagamentos – <description>
```

Example:
```
[GT de Serviços] PSV-399 - Payments - v5.0.0-beta.1: API Pagamentos – Proposta para definir validações que devem ser feitas durante autorização de consentimentos via JSR
```

### Pull Request Targets

All PRs target the `main` branch. The CI workflow uses `pull_request_target` (not `pull_request`), which allows the workflow to access repository secrets even for PRs from forks.

### Making Changes

1. Create a feature branch from `main` using the naming convention above.
2. Edit `swagger.yaml` — this is the **only file** that needs to be changed for API specification updates.
3. Ensure the `info.version` field is correct for the change being proposed.
4. Open a PR against `main`. The CI workflow will automatically trigger validation in the external CI/CD pipeline.
5. On merge, the tag-control workflow will automatically read the version from `swagger.yaml` and trigger tagging.

## Key Domain Concepts

- **ITP** (Iniciador de Transação de Pagamento) — Payment Transaction Initiator (the client application).
- **Detentor** (Detentora de Conta) — Account Holder (the bank/financial institution holding the payer's account).
- **DICT** — Diretório de Identificadores de Contas Transacionais do Pix (Pix directory for key resolution).
- **SPI** — Sistema de Pagamentos Instantâneos (Brazil's instant payment system infrastructure).
- **PSP do Recebedor** — Receiving PSP (Payment Service Provider of the payee).
- **JWS** — JSON Web Signature, used for payload signing on all API messages.
- **endToEndId** — End-to-end transaction identifier, passed intact through the payment chain.
- **consentId** — Unique consent identifier, used as an OAuth2 scope value (`consent:consentId`).
- **txId** — Transaction identifier (Pix field `EMV 62-05`), alphanumeric, up to 35 characters.
- **payloadJWS** — The JWS payload from the receiver's PSP QR code, required for `APDN` and `QRDN` instruments.

## Validation Rules Summary

Key constraints enforced at the spec level:
- Consent creation must not expose account holder information (balance, account status).
- DICT lookups must not occur at consent creation, only at payment creation.
- For scheduled payments, the `localInstrument` must be `MANU`, `DICT`, or `QRES`.
- `proxy` field must be filled with the Pix key when `localInstrument` is `INIC`, `DICT`, `APDN`, `APES`, `QRDN`, or `QRES`. Must be absent when `MANU`.
- `transactionIdentification` (txId) must be filled and ≤ 25 characters for `INIC`; must not be filled for `MANU` or `DICT`.
- `payloadJWS` is mandatory when `localInstrument` is `APDN` or `QRDN`.
- Cancellation via `PATCH` is only valid for payments in `SCHD` or `PDNG` status.
- Scheduled payments can be cancelled up to 23:59 (Brasília time) on the day before the scheduled date.
- Multiple approval (multi-alçada): immediate payments must be approved by 23:59 on the request date; scheduled payments by the day before settlement.
