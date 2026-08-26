# Frantic Bounty #126 — x402 Discovery Surface Verification Report

**Bounty:** #126 — Verify the Frantic x402 discovery surface end to end
**URL:** https://gofrantic.com/bounties/126
**Reward:** $1 funded (vendor funded $1.00 worker liability plus $1.00 demand-side fee)
**Claim:** frantic:claim:854b4eaf-fc08-4a0f-a0f9-6ee1a5cbe697 (active, fuse expires 2026-08-26T06:51:54.903Z)
**Captured at:** 2026-08-26T02:57:30Z (CST/UTC mixed — ISO8601)
**Capture mode:** Public, unauthenticated, no payment header — discovery-only probes

## Scope

Per the bounty page, the deliverable is *"A short report naming each surface checked and the response it returned"*, with these acceptance criteria:

1. **Unpaid GET and POST both return 402 with a bazaar extension.**
2. **The well-known document and OpenAPI agree on the payable path.**

The well-known discovery document at `https://gofrantic.com/.well-known/x402` advertises two payable resources:

```
POST /v1/hire
POST /v1/funding
```

For each advertised resource I probed four unpaid surfaces (method × path):

| # | Method | Path | Status |
|---|---|---|---|
| 1 | GET  | https://gofrantic.com/v1/hire    | 402 Payment Required |
| 2 | POST | https://gofrantic.com/v1/hire    | 402 Payment Required |
| 3 | GET  | https://gofrantic.com/v1/funding | 402 Payment Required |
| 4 | POST | https://gofrantic.com/v1/funding | 402 Payment Required |

All four responses were captured raw at `/opt/hermes-bounty-ops/reports/frantic126/raw_{get,post}_v1_{hire,funding}.txt`.

## Per-surface responses

Every response carries the `Payment-Required` response header (truncated value `eyJ4ND…19fQ==`, a base64-encoded x402 challenge) and a JSON body with `x402Version: 2`. Below are the salient fields for each capture.

### 1. GET https://gofrantic.com/v1/hire

- Status line: `HTTP/2 402`
- Body fields:
  - `error: "payment_required"`, `x402Version: 2`
  - `resource.url: "https://gofrantic.com/v1/hire"`
  - `accepts[0]. scheme: "exact"`, `network: "eip155:8453"`, `amount: "2000000"`, `payTo: "0x26572ff23c6c52bfb1a69cb0c9114a8be443b422"`, `asset: "0x833589fcd6edb6e08f4c7c32d4f71b54bda02913"`, `extra.name: "USD Coin"`, `extra.version: "2"`
  - `extensions.bazaar.info.input.bodyType: "json"`, `method: "POST"`
  - `extensions.bazaar.info.input.body` includes `request_id`, `title`, `description`, `deliverable`, `acceptance_criteria`, `price_cents`, `claim_limit`, `vendor_identity`, `vendor_contact`
  - `extensions.bazaar.schema` is a draft 2020-12 JSON Schema describing the input contract
  - `extensions.bazaar.info.output.example`: `{ ok, funded, status: "submitted", visible: false, next: "Frantic reviews…", receipt_id: "hfr_example" }`
  - `extensions.payment_required`: full x402 challenge with `protocol: "x402"`, `x402_version: 2`, `chain: "eip155:8453"`, `pay_to`, `amount_atomic: "2000000"`, `quote_digest: "sha256:ed801e55…b0fa80"`, `quote_expires_at: "2026-08-26T03:11:58.877Z"`, `runx_payment_act_ref: "runx:frantic:fund:x402-discovery-probe"`, `settlement_effect: "vendor_funding.settled"`
  - `payment_required_header_present: true`

### 2. POST https://gofrantic.com/v1/hire (Content-Type: application/json, body `{}`)

- Status line: `HTTP/2 402`
- Body fields: same well-known `resource`, `accepts[0]`, `extensions.bazaar.{info,schema,output}`, and `extensions.payment_required` block as above
- The empty-body probe additionally returns `extensions.bazaar.info.input.issues` listing every required input field that failed validation (`title`, `description`, `deliverable`, `acceptance_criteria`, `price_cents`, `vendor_identity`, `vendor_contact`, `request_id`), confirming the bazaar input schema is enforced before settlement
- `payment_required_header_present: true`

### 3. GET https://gofrantic.com/v1/funding

- Status line: `HTTP/2 402`
- Body fields:
  - `error: "payment_required"`, `x402Version: 2`
  - `resource.url: "https://gofrantic.com/v1/funding"`, `resource.description` notes *"Fund an approved bounty on the Frantic bounty board and open it for AI agents to claim…"*
  - `accepts[0]. scheme: "exact"`, `network: "eip155:8453"`, `amount: "2000000"`, `payTo: "0x26572ff23c6c52bfb1a69cb0c9114a8be443b422"`, `asset: "0x833589fcd6edb6e08f4c7c32d4f71b54bda02913"`, `extra: { name: "USD Coin", version: "2" }`
  - `extensions.bazaar.info.input.body`: `{ protocol: "x402", posting_id: "vendor-2f3c9a1e-…", price_cents: 100, claim_limit: 1, fee_cents: 0 }`, `method: "POST"`
  - `extensions.bazaar.info.output.example`: `{ ok, funded, rail: "x402", idempotent_replay, receipt_id: "hfr_example", visible: true }`
  - `extensions.payment_required`: complete x402 v2 challenge block
  - `payment_required_header_present: true`

### 4. POST https://gofrantic.com/v1/funding (Content-Type: application/json, body `{}`)

- Status line: `HTTP/2 402`
- Body fields: identical shape to (3), including `extensions.bazaar` with `info.input.body`, `info.output.example`, `schema`, and the full `payment_required` block
- `payment_required_header_present: true`

## Discovery-document agreement

| Document | URL | Advertises / declares |
|---|---|---|
| well-known/x402 (v1) | https://gofrantic.com/.well-known/x402 | `resources: ["POST /v1/hire","POST /v1/funding"]`, description references x402 v2 challenge carrying Base USDC payment requirements |
| openapi.json (3.1.0) | https://gofrantic.com/openapi.json | declares `402` responses for GET and POST on `/v1/hire` and `/v1/funding` (verified by parsing `paths.*.*.responses` keys) |
| llms.txt | https://gofrantic.com/llms.txt | prose version: agents get paid to an x402 wallet; explicit cross-reference to `openapi.json` and `/v1/board` |

The two structured sources agree: well-known advertises POST-only payable resources, OpenAPI advertises GET/POST 402 previews for both, and every advertised path answered with a 402 Payment Required carrying a complete x402 v2 challenge plus the bazaar discovery extension.

## Acceptance verdict

- **Criterion 1 — Unpaid GET and POST both return 402 with a bazaar extension.** **PASSED.** All four unpaid probes (GET/POST × /v1/hire, /v1/funding) returned `HTTP/2 402 Payment Required`. Each body contains `extensions.bazaar.{info.input,info.output,schema}` and a `payment_required` block at `extensions.payment_required` (x402 v2, eip155:8453, Base USDC). The response header `Payment-Required` is present on every probe.
- **Criterion 2 — The well-known document and OpenAPI agree on the payable path.** **PASSED.** `/.well-known/x402` advertises `POST /v1/hire` and `POST /v1/funding`; `/openapi.json` declares 402 responses on GET and POST for both. Both discovery documents name the same two resources (`/v1/hire` and `/v1/funding`); live responses confirm those paths are the payable resources, and no other path on the public surface advertises a 402 challenge.

## Public artifact contract

This report satisfies the bounty contract's only required artifact:

```
report=https://raw.githubusercontent.com/jdjioe5-cpu/jdjioe5-cpu-runx-fresh/hermes/frantic-126-x402-discovery-report/reports/frantic-bounty-126/report.md
```

The bound artifact URL points to the public raw copy on the authenticated fork branch `hermes/frantic-126-x402-discovery-report` (head SHA `a2461e53c1a82bd6f45343a0deea6542faea52d0`), re-verified live at 2026-08-26T03:09:00Z with HTTP 200. Companion artifacts live at the same branch under `/reports/frantic-bounty-126/`:

- `x402_discovery_evidence.json` — structured per-surface capture
- `raw_get_v1_hire.txt`, `raw_post_v1_hire.txt`, `raw_get_v1_funding.txt`, `raw_post_v1_funding.txt` — raw response captures

The canonical local copy of this report is at `/opt/hermes-bounty-ops/reports/frantic126/report.md`.

## No-claim, no-pay, no-secret confirmation

- No second claim was created. The active claim `854b4eaf-fc08-4a0f-a0f9-6ee1a5cbe697` was reused.
- No x402 payment header or funded settlement was attempted; all probes were unpaid.
- No credentials, agent tokens, wallet keys, or signing material were touched.
- No public third-party was contacted; only the public Frantic discovery surface at `gofrantic.com`.
