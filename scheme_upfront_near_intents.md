# Scheme: `upfront` on NEAR Intents (1Click Swap)

## Summary

This document specifies the `1click-swap` asset transfer method of the [`upfront`](./scheme_upfront.md) scheme, using the [NEAR Intents 1Click Swap API](https://docs.near-intents.org/integration/distribution-channels/1click-api/about-1click-api) as the settlement backend. This method facilitates cross-chain payments where a client pays a specified amount of a source asset on any [supported origin chain](https://docs.near-intents.org/resources/chain-support), and the resource server (merchant) receives an exact amount of a destination asset on any supported destination chain, with the NEAR Intents solver network executing the cross-chain swap in between.

`1click-swap` belongs to the **payment-proof (client-settled)** family of `upfront` asset transfer methods: the client pays first by broadcasting a deposit transaction on the origin chain (paying any network fee), and presents the transaction hash as a payment proof. It extends the basic payment-proof model with an **asynchronous settlement backend**: between the client's payment and finality, the NEAR Intents solver network executes a cross-chain swap. The facilitator's settlement step therefore awaits finality of an already-client-initiated payment — it does not move funds itself.

Consistent with the defining property of `upfront`, **payment is final before the protected route handler executes**. In this method, finality is defined as **destination delivery**: the route handler runs only after the merchant has received the destination asset (see [Finality Definition](#finality-definition)). This is a strictly stronger guarantee than transfer-broadcast finality.

---

## Protocol Flow

```
┌────────┐          ┌───────────────┐          ┌────────────────┐       ┌──────────────┐
│ Client │          │Resource Server│          │  Facilitator   │       │ 1Click Swap  │
│(Buyer) │          │  (Merchant)   │          │(x402 + 1Click) │       │     API      │
└───┬────┘          └──────┬────────┘          └───────┬────────┘       └──────┬───────┘
    │                      │                           │                      │
    │  1. GET /resource    │                           │                      │
    │─────────────────────>│                           │                      │
    │                      │                           │                      │
    │                      │  2. Request quote         │                      │
    │                      │──────────────────────────>│                      │
    │                      │                           │  3. POST /v0/quote   │
    │                      │                           │     (dry: false)     │
    │                      │                           │─────────────────────>│
    │                      │                           │  quote response with │
    │                      │                           │  depositAddress,     │
    │                      │                           │  maxAmountIn...      │
    │                      │                           │<─────────────────────│
    │                      │  PaymentRequirements      │                      │
    │                      │  (payTo=depositAddress)   │                      │
    │                      │<──────────────────────────│                      │
    │                      │                           │                      │
    │  4. 402 Payment      │                           │                      │
    │     Required         │                           │                      │
    │  (payTo = deposit    │                           │                      │
    │   address, extra =   │                           │                      │
    │   quote metadata)    │                           │                      │
    │<─────────────────────│                           │                      │
    │                      │                           │                      │
    │  5. Client sends     │                           │                      │
    │     deposit TX on    │                           │                      │
    │     origin chain     │                           │                      │
    │     to payTo address │                           │                      │
    │     (client-settled  │                           │                      │
    │      payment)        │                           │                      │
    │  ════════════════════╪═══════════════════════════╪══(on-chain TX)═══════│
    │                      │                           │                      │
    │  6. GET /resource    │                           │                      │
    │  X-PAYMENT: {payload │                           │                      │
    │   with txHash =      │                           │                      │
    │   payment proof}     │                           │                      │
    │─────────────────────>│                           │                      │
    │                      │                           │                      │
    │                      │  7. POST /verify          │                      │
    │                      │──────────────────────────>│                      │
    │                      │                           │ Validate proof:      │
    │                      │                           │ txHash, deposit-     │
    │                      │                           │ Address matches,     │
    │                      │                           │ amount, freshness,   │
    │                      │                           │ single-use           │
    │                      │  VerifyResponse (valid)   │                      │
    │                      │<──────────────────────────│                      │
    │                      │                           │                      │
    │                      │  8. POST /settle          │                      │
    │                      │  (BEFORE route handler)   │                      │
    │                      │──────────────────────────>│                      │
    │                      │                           │ 9. POST              │
    │                      │                           │   /v0/deposit/submit │
    │                      │                           │─────────────────────>│
    │                      │                           │      OK              │
    │                      │                           │<─────────────────────│
    │                      │                           │                      │
    │                      │                           │ 10. Poll GET         │
    │                      │                           │   /v0/status         │
    │                      │                           │─────────────────────>│
    │                      │                           │  status: SUCCESS     │
    │                      │                           │  (destination        │
    │                      │                           │   delivery = final)  │
    │                      │                           │<─────────────────────│
    │                      │                           │                      │
    │                      │  SettlementResponse       │                      │
    │                      │  (success,                │                      │
    │                      │ destinationChainTxHashes) │                      │
    │                      │<──────────────────────────│                      │
    │                      │                           │                      │
    │                      │ 11. Execute protected     │                      │
    │                      │     route handler         │                      │
    │                      │     (only after settle    │                      │
    │                      │      succeeds)            │                      │
    │                      │                           │                      │
    │  12. 200 OK          │                           │                      │
    │  + resource body     │                           │                      │
    │  + X-PAYMENT-RESPONSE│                           │                      │
    │<─────────────────────│                           │                      │
```

### Step-by-step

1. **Client → Resource Server**: Client makes an HTTP request (e.g., `GET /resource`) without payment headers.

2. **Resource Server → Facilitator**: The resource server's x402 middleware invokes the facilitator to construct `PaymentRequirements`. The facilitator calls the 1Click API `quote` endpoint with `dry: false` and the merchant's swap configuration (origin asset, destination asset, amount, recipient address, refund policy, fees).

3. **Facilitator → 1Click API**: `POST /v0/quote` with `dry: false` and `swapType: EXACT_OUTPUT` returns the full quote including a unique `depositAddress`, `depositMemo` (if applicable), `maxAmountIn`, `amountOut`, `deadline`, and `timeEstimate`.

4. **Resource Server → Client**: The resource server responds `402 Payment Required` with the `PaymentRequirements` object. Critically, `payTo` is set to the 1Click `depositAddress`, and `network` is the CAIP-2 identifier of the origin chain where that address lives. The `extra` field embeds all quote metadata the client needs: the minimum deposit amount, deposit memo (if any), the quote deadline, and the destination-leg details.

5. **Client sends deposit (client-settled payment)**: The client constructs and submits a native transaction on the origin chain, transferring the required `amount` to the `payTo` address (plus `depositMemo` if required, e.g., for Stellar). This transaction **is** the payment — funds irrevocably leave the client's wallet at this point. Clients opt in to this risk allocation per the `upfront` scheme definition.

6. **Client → Resource Server**: The client resends the original request with the `X-PAYMENT` header containing a `PaymentPayload` whose `txHash` is the **payment proof**.

7. **Resource Server → Facilitator `/verify`**: The facilitator validates the proof: structural consistency (`depositAddress` matches `payTo`, memo and network match), quote freshness, on-chain or backend-confirmed deposit of at least `minAmountIn` to the deposit address, and single-use (the proof has not been consumed before). See [Verification](#verification-post-verify).

8. **Resource Server → Facilitator `/settle`**: Upon successful verification, the resource server calls `/settle`. Per the `upfront` scheme, this MUST happen **before** the protected route handler is invoked.

9. **Facilitator → 1Click API**: The facilitator calls `POST /v0/deposit/submit` with the `txHash` and `depositAddress` to notify the 1Click service and speed up processing.

10. **Facilitator polls status**: The facilitator polls `GET /v0/status?depositAddress=<addr>[&depositMemo=<memo>]` until a terminal status: `SUCCESS`, `FAILED`, `REFUNDED`, or `INCOMPLETE_DEPOSIT`. `SUCCESS` means the destination asset has been delivered to the merchant — this is the finality boundary of this method.

11. **Route handler executes**: Only after a successful `SettlementResponse` does the resource server execute the protected route handler. If settlement fails, the server MUST NOT execute the resource and returns `402`; the settlement backend automatically refunds the client to `refundTo` (see [Refunds](#refunds)).

12. **Resource Server → Client**: On `SUCCESS`, the resource server responds `200 OK` with the requested resource and the `X-PAYMENT-RESPONSE` header containing the `SettlementResponse`.

---

## PaymentRequirements for `upfront` / `1click-swap`

```jsonc
{
  "scheme": "upfront",
  "network": "eip155:42161",               // CAIP-2 of the ORIGIN chain — where payTo lives and where the payment proof is anchored
  "amount": "1005000",                     // Deposit amount the client must send (in smallest unit of origin asset)
  "asset": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831",
                                            // Origin asset the client pays WITH (USDC on Arbitrum)
  "payTo": "0x76b4c56085ED136a8744D52bE956396624a730E8",
                                            // 1Click deposit address (on the origin chain)
  "maxTimeoutSeconds": 300,
  "extra": {
    "assetTransferMethod": "1click-swap",
    "settlementBackend": "near-intents",   // Discloses the backend executing the swap leg
    "depositMemo": null,                   // Required if present in quote (e.g. Stellar); null otherwise
    "minAmountIn": "1000000",              // Minimum accepted deposit (from 1Click quote)
    "destinationAsset": "nep141:17208628f84f5d6ad33f0da3bbbeb27ffcb398eac501a31bd6ad2011e36133a1",  // What the merchant receives
    "destinationNetwork": "near:mainnet",  // Where the merchant receives it
    "amountOut": "1000000",                // Guaranteed output to merchant (EXACT_OUTPUT)
    "slippageTolerance": 100,              // Basis points (100 = 1%)
    "deadline": "2026-06-25T15:10:00Z",    // Quote expiry (from 1Click)
    "timeEstimate": 120,                   // Estimated swap time in seconds
    "refundTo": "0x2527D02599Ba641c19FEa793cD0F9a6e8457C317" // Refund address on the origin chain (set by client registration or pre-flight)
  }
}
```

and the full `paymentRequirements` object:

```json
{
  "x402Version": 2,
  "resource": {
    "url": "https://api.example.com/premium-data",
    "description": "Cross-chain premium market data access",
    "mimeType": "application/json"
  },
  "accepted": {
    "scheme": "upfront",
    "network": "eip155:42161",
    "amount": "1005000",
    "asset": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831",
    "payTo": "0x76b4c56085ED136a8744D52bE956396624a730E8",
    "maxTimeoutSeconds": 300,
    "extra": {
      "assetTransferMethod": "1click-swap",
      "settlementBackend": "near-intents",
      "depositMemo": null,
      "minAmountIn": "1000000",
      "destinationAsset": "nep141:17208628f84f5d6ad33f0da3bbbeb27ffcb398eac501a31bd6ad2011e36133a1",
      "destinationNetwork": "near:mainnet",
      "amountOut": "1000000",
      "slippageTolerance": 100,
      "deadline": "2026-06-25T15:10:00Z",
      "timeEstimate": 120,
      "refundTo": "0x2527D02599Ba641c19FEa793cD0F9a6e8457C317"
    }
  }
}
```

### Mapping to Standard x402 Fields

As is native to the payment-proof family, `amount`, `asset`, `payTo`, and `network` describe **the payment the client makes** — the proof attests to exactly these values. The destination leg of the swap (what the merchant receives) is carried in `extra`.

| x402 Field | 1Click Source | Semantics in This Method |
|---|---|---|
| `scheme` | — | Always `"upfront"`. |
| `network` | Request `originAsset` chain | CAIP-2 identifier of the **origin chain**: the chain where `payTo` lives, where the client broadcasts the deposit, and where the payment proof (`txHash`) is anchored. Standard SDK wallet selection by `network` applies unchanged. |
| `amount` | `quote.maxAmountIn` | The **deposit amount** the client must send on the origin chain. |
| `asset` | Request `originAsset` | The **origin asset** the client pays with (see [Asset Identifier Convention](#asset-identifier-convention)). |
| `payTo` | `quote.depositAddress` | The **1Click deposit address** on the origin chain. The client sends tokens here. |
| `maxTimeoutSeconds` | `deadline` − now + `timeEstimate` + buffer | Max time the resource server will wait for full settlement (destination delivery). |

### Asset Identifier Convention

| Origin chain type | `asset` format | Example |
|---|---|---|
| EVM chains | ERC-20 contract address | `0xaf88d065e77c8cC2239327C5EDb3A432268e5831` (USDC, Arbitrum) |
| Solana | SPL token mint address | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` (USDC) |
| Native assets without a contract address | CAIP-19 asset identifier | `bip122:000000000019d6689c085ae165831e93/slip44:0` (BTC) |
| NEAR | NEP-141 contract account ID | `wrap.near` |

Implementations MUST NOT use informal short identifiers (e.g., `"arb"`, `"BTC"`) in the `asset` field.

### Extra Field Descriptions

| Field | Type | Required | Description |
|---|---|---|---|
| `assetTransferMethod` | string | Yes | Always `"1click-swap"`. Discriminates this method's verification path within `upfront`. |
| `settlementBackend` | string | Yes | Always `"near-intents"`. Discloses the backend that executes the swap leg between client payment and finality. |
| `depositMemo` | string \| null | Yes | Deposit memo if required by the origin chain (e.g., Stellar). `null` if not needed. |
| `minAmountIn` | string | Yes | Minimum deposit amount accepted by the 1Click quote. Deposits below this are refunded. |
| `destinationAsset` | string | Yes | Asset the merchant receives, as a Defuse asset identifier or destination-chain contract address. |
| `destinationNetwork` | string | Yes | CAIP-2 identifier of the chain where the merchant receives `destinationAsset`. |
| `amountOut` | string | Yes | Guaranteed output amount to the merchant (in smallest unit of destination asset), enforced via `EXACT_OUTPUT`. |
| `slippageTolerance` | integer | Yes | Slippage tolerance in basis points. |
| `deadline` | string (ISO 8601) | Yes | Quote expiry timestamp from 1Click. Client MUST deposit before this time. |
| `timeEstimate` | integer | Yes | Estimated swap completion time in seconds (from 1Click). |
| `refundTo` | string | Yes | Address on the origin chain where funds are automatically refunded if the swap fails or excess exists. |

---

## PaymentPayload `payload` Field

```jsonc
  "payload": {
    "txHash": "0x9bcff372aee89b648c922b850573b22387c31d693079f5e37cd255814e2d615a",
    "depositAddress": "0x76b4c56085ED136a8744D52bE956396624a730E8",
    "depositMemo": null,
    "clientAddress": "0x2527D02599Ba641c19FEa793cD0F9a6e8457C317"
  }
```

and the full `PaymentPayload` object:

```json
{
  "x402Version": 2,
  "scheme": "upfront",
  "network": "eip155:42161",
  "payload": {
    "txHash": "0x9bcff372aee89b648c922b850573b22387c31d693079f5e37cd255814e2d615a",
    "depositAddress": "0x76b4c56085ED136a8744D52bE956396624a730E8",
    "depositMemo": null,
    "clientAddress": "0x2527D02599Ba641c19FEa793cD0F9a6e8457C317"
  }
}
```

### Field Descriptions

| Field | Type | Required | Description |
|---|---|---|---|
| `network` (envelope) | string | Yes | MUST equal `paymentRequirements.network`. Identifies the chain the `txHash` belongs to. |
| `payload.txHash` | string | Yes | Transaction hash of the client's deposit on the origin chain. This is the **payment proof** of the payment-proof family — the network-native settlement reference the verifier checks against the requirements. |
| `payload.depositAddress` | string | Yes | Must match `payTo` from `PaymentRequirements`. Included for self-contained verification. |
| `payload.depositMemo` | string \| null | Conditional | Must match `extra.depositMemo` from requirements. Required if non-null. |
| `payload.clientAddress` | string | Yes | The client's address on the origin chain (sender of the deposit TX). Used for audit, on-chain verification, and as the `payer` in responses. |

---

## Facilitator Behavior

### Quote Generation (402 Construction Time)

When the resource server needs to construct a `402 Payment Required` response, it invokes the facilitator with the merchant's swap configuration. The facilitator:

1. Calls `POST {apiBaseUrl}/v0/quote` with `dry: false`:
   ```jsonc
   {
     "dry": false,
     "originAsset": "<configured origin asset>",
     "destinationAsset": "<merchant's destination asset>",
     "amount": "<merchant's desired output amount>",
     "swapType": "EXACT_OUTPUT",
     "slippageTolerance": 100,
     "depositType": "ORIGIN_CHAIN",
     "recipientType": "DESTINATION_CHAIN",
     "refundTo": "<configured refund address>",
     "recipient": "<merchant wallet>",
     "deadline": "<now + configured TTL>",
     "referral": "<x402-1click>",
     "appFees": [...]
   }
   ```
2. Validates the quote response has a `depositAddress` and `maxAmountIn`.
3. Constructs the `PaymentRequirements` with `payTo = quote.depositAddress`, `network = <origin chain CAIP-2>`, and embeds all quote metadata in `extra`.

> **Note**: Because the quote generates a time-limited deposit address, the resource server SHOULD cache the `PaymentRequirements` for the duration of the quote's `deadline` and serve the same `depositAddress` for repeated 402 responses to the same resource, regenerating only when the deadline expires. Initial price discovery MAY use `dry: true` quotes, which do not allocate a deposit address.

### Verification (`POST /verify`)

In the payment-proof family, proof verification **is** the security model — there is no later settlement transaction to act as a backstop. The verifier confirms the proof matches the requirement on amount, recipient, freshness, and single-use. When the facilitator receives a `PaymentPayload`:

1. **Validate structural consistency** (MUST):
   - `payload.depositAddress` MUST equal `paymentRequirements.payTo`.
   - `payload.depositMemo` MUST equal `paymentRequirements.extra.depositMemo`.
   - The payload envelope `network` MUST equal `paymentRequirements.network`.
   - `payload.txHash` MUST be a non-empty, well-formed transaction hash for the declared chain.

2. **Check quote freshness** (MUST):
   - Current time MUST be before `paymentRequirements.extra.deadline`.
   - The `depositAddress` MUST correspond to an active, non-expired, non-settled quote in the facilitator's state.

3. **Verify the payment proof** (MUST):
   - The facilitator MUST confirm that `payload.txHash` is a confirmed transaction on `network` transferring ≥ `paymentRequirements.extra.minAmountIn` of `asset` to the `depositAddress` (with `depositMemo`, if applicable).
   - This confirmation MAY be performed by querying the origin chain's RPC directly, or by querying `GET /v0/status` and observing that the 1Click backend has detected the deposit (e.g., `KNOWN_DEPOSIT_TX` / `PROCESSING`). Either path satisfies the requirement; relying solely on client-asserted payload fields does not.

4. **Enforce single-use** (MUST):
   - `payload.txHash` MUST NOT have been consumed by a previous verification on this facilitator (see [Replay Prevention](#replay-prevention)).

5. **Return `VerifyResponse`**:
   ```jsonc
   {
     "isValid": true,
     "invalidReason": null,
     "payer": "0x2527D02599Ba641c19FEa793cD0F9a6e8457C317"
   }
   ```

### Settlement (`POST /settle`, pre-execution)

The resource server MUST call `/settle` **before** invoking the protected route handler, and MUST NOT execute the resource unless settlement succeeds. Because the client has already initiated the payment, the facilitator's settlement step does not move funds — it **awaits finality** of the client-initiated payment through the settlement backend:

```
verify(payload, requirements)         # facilitator: proof verification
  → on failure: return 402, do not execute resource
settle(payload, requirements)         # facilitator: await destination delivery via 1Click
  → on failure: return 402, do not execute resource (client is auto-refunded)
execute(protected resource)           # only after settle succeeds
return response + X-PAYMENT-RESPONSE
```

Concretely, the facilitator:

1. **Notifies 1Click** by calling `POST {apiBaseUrl}/v0/deposit/submit`:
   ```jsonc
   {
     "txHash": "<payload.txHash>",
     "depositAddress": "<payload.depositAddress>"
   }
   ```
   This is optional, as 1Click detects deposits automatically, but accelerates processing.

2. **Polls for terminal status** via `GET {apiBaseUrl}/v0/status?depositAddress=<addr>[&depositMemo=<memo>]` at 3–5 second intervals until terminal or `maxTimeoutSeconds` is exceeded.

3. **Returns `SettlementResponse`**:

   **On `SUCCESS`:**
   ```jsonc
   {
     "success": true,
     "network": "eip155:42161",
     "transaction": "<destinationChainTxHashes from 1Click status>",
     "payer": "<payload.clientAddress>",
     "extensions": {
       "depositAddress": "<depositAddress>",
       "originTxHash": "<payload.txHash>",
       "nearTxHashes": ["6XqqDwoa...", "EVcgKukw..."],
       "amountInFormatted": "1.005",
       "amountOutFormatted": "1.00",
       "status": "SUCCESS"
     }
   }
   ```

   **On failure (`FAILED` / `REFUNDED` / `INCOMPLETE_DEPOSIT` / timeout):**
   ```jsonc
   {
     "success": false,
     "network": "eip155:42161",
     "error": "<error_code>",
     "extensions": {
       "depositAddress": "<depositAddress>",
       "status": "<terminal status from 1Click>",
       "refundTo": "<extra.refundTo>"
     }
   }
   ```

### Finality Definition

Methods in the payment-proof family must state what "payment is final before the route handler executes" means. `1click-swap` has two candidate finality boundaries:

1. **Origin-chain deposit confirmation** — the client's deposit transaction has reached sufficient confirmation depth on the origin chain.
2. **Destination delivery** — the swap has completed and the merchant has received `amountOut` of `destinationAsset` (1Click terminal status `SUCCESS`).

This method defines finality as **(2) destination delivery**. The route handler MUST NOT run until the 1Click status is `SUCCESS`. This gives the resource server a guarantee strictly stronger than transfer-broadcast finality: when the handler runs, the merchant already holds the destination funds.

Per-origin-chain confirmation-depth policy (boundary 1) is **owned by the settlement backend**: the 1Click backend enforces chain-appropriate confirmation depth before executing a swap, including on chains with probabilistic finality (e.g., Bitcoin block confirmations, Solana commitment levels). Facilitators MUST NOT treat a deposit observed below the backend's confirmation threshold as final; in practice this is automatic, since `SUCCESS` cannot be reached before the backend confirms the deposit.

### `GET /supported`

Facilitators implementing this method MUST advertise it via `GET /supported`. Each supported origin chain appears as its own `kinds` entry:

```jsonc
{
  "kinds": [
    {
      "x402Version": 2,
      "scheme": "upfront",
      "network": "eip155:42161",
      "extra": { "assetTransferMethods": ["1click-swap"] }
    },
    {
      "x402Version": 2,
      "scheme": "upfront",
      "network": "eip155:1",
      "extra": { "assetTransferMethods": ["1click-swap"] }
    },
    {
      "x402Version": 2,
      "scheme": "upfront",
      "network": "bip122:000000000019d6689c085ae165831e93",
      "extra": { "assetTransferMethods": ["1click-swap"] }
    }
  ],
  "signers": {}
}
```

The `signers` map is empty for this method: the facilitator never signs or submits value-moving transactions; the only signer is the client's origin-chain wallet, and the payment proof replaces facilitator-side signing entirely.

### Facilitator State

The facilitator MUST maintain transient state for active quotes, keyed by `depositAddress`:

| Key | Stored At | Description |
|---|---|---|
| `depositAddress` | Quote time | The unique deposit address |
| `depositMemo` | Quote time | Memo, if applicable |
| `QuoteResponse` | Quote time | Full 1Click quote response |
| `paymentRequirements` | Quote time | The PaymentRequirements served to the client |
| `deadline` | Quote time | Quote expiry |
| `txHashes` | Verify time | Set of consumed deposit TX hashes for this address (proof tracking) |
| `clientAddress` | Verify time | Client's origin chain address |
| `settlementStatus` | Settle time | Last observed 1Click status |

State SHOULD be garbage-collected after `deadline` + `maxTimeoutSeconds` + grace period, or after terminal status. Consumed `txHashes` SHOULD be retained for the full GC window even after terminal status, to enforce single-use across the facilitator's lifetime of the quote.

---

## Core Scheme Properties Compliance

How this method satisfies the four properties every `upfront` implementation MUST enforce:

| # | `upfront` Core Property | Enforcement in `1click-swap` |
|---|---|---|
| 1 | **Settle-before-execute ordering** | The route handler runs only after 1Click terminal `SUCCESS` (destination delivery). On any verification or settlement failure, the server MUST NOT execute the resource. |
| 2 | **Exact amount** | `swapType: EXACT_OUTPUT` guarantees the merchant receives exactly `extra.amountOut` of `destinationAsset`. On the client side, deposits MUST be ≥ `extra.minAmountIn`; excess above the quoted requirement is automatically refunded to `refundTo` (the documented dust/excess convention for this method). |
| 3 | **Recipient binding** | The `depositAddress` ↔ `recipient` pairing is fixed in the 1Click quote at quote time and is immutable thereafter: funds arriving at the deposit address can only be swapped to the quoted recipient or refunded to `refundTo`. This binding is **backend-enforced** rather than enforced by an on-chain contract — see [Trust Model](#trust-model). Neither the facilitator nor the resource server can instruct the backend to redirect funds post-quote. |
| 4 | **Single-use / replay protection** | The unique `depositAddress` acts as a natural per-payment nonce, and the verifier additionally tracks consumed `txHash` proofs and rejects reuse. See [Replay Prevention](#replay-prevention). |

---

## Additional Considerations

### Replay Prevention

- Each quote generates a **unique `depositAddress`** which serves as a natural nonce.
- A given `depositAddress` can only be used for one swap — the 1Click backend rejects duplicate deposits.
- The facilitator MUST reject payloads where `depositAddress` does not correspond to an active, non-expired, non-settled quote in its state.
- The facilitator MUST additionally track consumed payment proofs: each `(depositAddress, txHash)` pair MUST be accepted at most once. A payload presenting a `txHash` already consumed — for the same or a different resource — MUST be rejected with `invalid_upfront_1click_proof_reused`.
- **Multiple deposits to one address**: the 1Click backend aggregates deposits to the same address (e.g., a top-up after an initial under-deposit). The facilitator MAY accept a payload whose `txHash` is any one of the aggregated deposit transactions, provided the aggregate meets `minAmountIn`; each individual `txHash` is still single-use.

### Authorization Scope

A deposit to a 1Click deposit address authorizes **exactly one swap with the immutable parameters fixed at quote time**: the quoted `destinationAsset`, `amountOut`, `recipient`, `slippageTolerance`, and `refundTo`. Funds arriving at the address cannot be used for any other purpose — they are either swapped per the quote or refunded to `refundTo`. There is no standing approval, no reusable allowance, and no authority for the facilitator or resource server to alter the swap parameters after the quote is issued.

### Settlement Atomicity

The swap leg is executed atomically on the NEAR Intents settlement layer (`token_diff` on the Verifier contract), but the end-to-end flow spans three phases. The possible partial-completion states and their resolution:

| State | Meaning | Resolution |
|---|---|---|
| Deposit confirmed, swap not yet executed | Funds at deposit address, solvers executing | Proceeds to `SUCCESS` or refund; route handler does not run yet |
| `INCOMPLETE_DEPOSIT` | Aggregate deposit < `minAmountIn` at deadline | Automatic refund to `refundTo`; `402` returned |
| `FAILED` | Swap could not be executed (e.g., liquidity, slippage breach) | Automatic refund to `refundTo`; `402` returned |
| `REFUNDED` | Terminal refund state | Client made whole on origin chain; `402` returned |
| Settlement-layer deduction succeeded, destination delivery pending | Intra-backend state during cross-chain delivery | Not observable as terminal; status remains non-terminal until delivery completes or the backend refunds |
| `SUCCESS` | Destination delivery complete | Route handler executes; `200` returned |

The resource server only ever observes terminal states; intermediate states never trigger resource execution. There is no terminal state in which the client has paid, the merchant has not been paid, and no refund is issued.

### Deposit Address Validity Window

- The `deadline` field from the 1Click quote defines when the deposit address becomes inactive.
- The `maxTimeoutSeconds` in `PaymentRequirements` SHOULD be set to: `(deadline - now) + timeEstimate + buffer`.
- Clients MUST submit their deposit transaction **before** the `deadline`. Deposits after the deadline may be refunded instead of swapped.

### Amount Validation

- The facilitator MUST verify (on-chain or via 1Click status) that the deposited amount is ≥ `extra.minAmountIn` (see [Verification](#verification-post-verify)).
- Deposits below `minAmountIn` result in the 1Click status `INCOMPLETE_DEPOSIT` and are refunded.
- Deposits above `minAmountIn` are processed normally and excess is refunded to `refundTo`.

### Deposit Address Authenticity

- The facilitator MUST only serve deposit addresses obtained from authenticated calls to the 1Click API (using a valid JWT).
- The facilitator MUST NOT relay deposit addresses from untrusted sources.
- Clients interacting with an untrusted resource server bear the risk of sending funds to a malicious address — this is the same trust model as any payment gateway integration. Because the client pays first under `upfront`, clients SHOULD apply heightened scrutiny to the resource server's identity before depositing.

### Latency and Long-Running Settlement

Because settlement (destination delivery) sits in the critical path before the route handler, the synchronous HTTP 402 flow blocks for the swap duration: typically under `timeEstimate` (~2 minutes for EVM/Solana origins) but potentially much longer for slow-finality origins (e.g., Bitcoin). Resource servers SHOULD:

- Calibrate `maxTimeoutSeconds` per origin chain (e.g., 300s for EVM origins, 3600s for Bitcoin).
- For settlements expected to exceed reasonable synchronous HTTP windows, return `202 Accepted` with a retry/polling location instead of holding the connection, or support webhook notification of settlement completion.
- Surface `extra.timeEstimate` to clients so they can set appropriate request timeouts.

---

## Refunds

The `upfront` scheme defines refunds as out of protocol: servers MAY offer them, and clients MUST NOT assume a refund path exists. The `1click-swap` method **exceeds this baseline**: refunds for failed, incomplete, or expired swaps are issued **automatically by the settlement backend** to `extra.refundTo` on the origin chain, with no action required from the resource server, facilitator, or merchant. This materially reduces the delivery risk the client accepts when opting into `upfront`:

- If the swap cannot complete (`FAILED`, `INCOMPLETE_DEPOSIT`, deadline missed), the deposit is refunded to `refundTo`.
- Excess deposited above the quoted requirement is refunded to `refundTo`.
- The refund covers the deposited origin asset; origin-chain network fees paid by the client are not recoverable.

Note that automatic refunds cover **settlement failure** (the payment leg). Failure of the resource itself after successful settlement — i.e., the route handler errors after the merchant has been paid — remains out of protocol, exactly as for all other `upfront` methods.

---

## Error Codes

In addition to the standard x402 error codes and the common `upfront` codes (`upfront_settlement_failed`, `upfront_unsupported_asset_transfer_method`):

| Code | Description |
|---|---|
| `invalid_upfront_1click_quote_expired` | The `deadline` has passed; the deposit address is no longer active. |
| `invalid_upfront_1click_deposit_not_found` | The payment proof could not be confirmed: the 1Click status API does not recognize the deposit, and on-chain verification found no qualifying transfer. |
| `invalid_upfront_1click_incomplete_deposit` | Deposited amount is below `minAmountIn`. |
| `invalid_upfront_1click_deposit_address_mismatch` | `payload.depositAddress` does not match `paymentRequirements.payTo`. |
| `invalid_upfront_1click_network_mismatch` | The payload envelope `network` does not match `paymentRequirements.network`. |
| `invalid_upfront_1click_memo_mismatch` | `payload.depositMemo` does not match `paymentRequirements.extra.depositMemo`. |
| `invalid_upfront_1click_proof_reused` | The `txHash` payment proof has already been consumed. |
| `upfront_1click_swap_failed` | 1Click swap reached `FAILED` terminal status; deposit automatically refunded. |
| `upfront_1click_swap_refunded` | Swap could not be completed; funds refunded to `refundTo`. |
| `upfront_1click_settlement_timeout` | Swap did not reach terminal status within `maxTimeoutSeconds`. |

---

## References

- [`upfront` scheme definition](./scheme_upfront.md)
- [1Click API Reference](https://docs.near-intents.org/near-intents/integration/distribution-channels/1click-api)
- [NEAR Intents Supported Chains](https://docs.near-intents.org/resources/chain-support)
- [CAIP-2 Chain ID Specification](https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-2.md)
- [CAIP-19 Asset Type and Asset ID Specification](https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-19.md)

## Appendix

### 1Click API Endpoint Mapping

| x402 Operation | 1Click API Endpoint | When Called |
|---|---|---|
| Construct PaymentRequirements | `POST /v0/quote` (`dry: false`) | Resource server builds 402 response |
| Price discovery (no deposit address) | `POST /v0/quote` (`dry: true`) | Optional, before committing to a wet quote |
| Token/asset discovery | `GET /v0/tokens` | Initial configuration |
| Proof verification (deposit detection) | `GET /v0/status` | Facilitator `/verify` |
| Deposit notification | `POST /v0/deposit/submit` | Facilitator `/settle` |
| Status polling to finality | `GET /v0/status` | Facilitator `/settle` — poll until terminal |

### Trust Model

| Relationship | Trust Required | Comparable To |
|---|---|---|
| Client → 1Click deposit address | Client trusts the settlement backend to either complete the swap or refund | User trusting Stripe/PayPal with a payment |
| Resource server → Facilitator | Standard x402 trust model | Same as other `upfront` methods |

Recipient binding (Core Property 3) in this method is **backend-enforced** rather than contract-enforced: the immutability of the `depositAddress` ↔ `recipient` pairing is a guarantee of the 1Click backend and the NEAR Intents solver network, not of an on-chain authorization signed by the client. For the duration of the swap, the backend custodies the deposited funds. The risk allocation is consistent with the `upfront` scheme's design — the client pays first and opts into delivery risk — and is mitigated by the automatic refund mechanism: every failure path terminates in either destination delivery or a refund to `refundTo`. Clients and autonomous agents applying per-method risk policies can discriminate on `extra.assetTransferMethod` and `extra.settlementBackend`.

### Multi-Origin-Chain Support

A resource server MAY advertise multiple `PaymentRequirements` in its 402 response, each a standard `upfront` entry on a different origin `network` with its own `depositAddress`. Because `network` carries the origin chain, client SDKs select the wallet/signer exactly as for any other multi-network `accepts` array — no method-specific routing logic is required:

```jsonc
{
  "x402Version": 2,
  "paymentRequirements": [
    {
      "scheme": "upfront",
      "network": "eip155:42161",
      "amount": "1005000",
      "asset": "0xaf88d065e77c8cC2239327C5EDb3A432268e5831",   // USDC on Arbitrum
      "payTo": "0x76b4c560...",                     // Arbitrum deposit address
      "maxTimeoutSeconds": 300,
      "extra": { "assetTransferMethod": "1click-swap", /* ... */ }
    },
    {
      "scheme": "upfront",
      "network": "eip155:1",
      "amount": "1005000",
      "asset": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",   // USDC on Ethereum
      "payTo": "0xA1B2C3D4...",                     // Ethereum deposit address
      "maxTimeoutSeconds": 600,
      "extra": { "assetTransferMethod": "1click-swap", /* ... */ }
    },
    {
      "scheme": "upfront",
      "network": "bip122:000000000019d6689c085ae165831e93",
      "amount": "38000",
      "asset": "bip122:000000000019d6689c085ae165831e93/slip44:0",  // Native BTC (CAIP-19)
      "payTo": "bc1qxy2kgd...",                    // Bitcoin deposit address
      "maxTimeoutSeconds": 3600,
      "extra": { "assetTransferMethod": "1click-swap", /* ... */ }
    }
  ]
}
```

Each entry requires a separate `POST /v0/quote` (`dry: false`) call. Resource servers SHOULD limit the number of concurrent wet quotes to manage deposit address TTL overhead — a practical pattern is one wet quote for the primary origin chain plus `dry: true` price previews for alternatives, upgrading to a wet quote when the client signals a preference.

### Deposit Address TTL Management

The 1Click deposit address is valid until the `deadline` (typically configurable, defaults to ~1 hour). The resource server's x402 middleware should:

- Generate quotes on-demand when a client first hits the 402-protected endpoint.
- Cache the `PaymentRequirements` (keyed by resource URL or session) for the remaining TTL.
- Regenerate when the cached quote expires or after successful settlement.

### `refundTo` Configuration

The `refundTo` address in the 1Click quote determines where failed swaps and excess deposits are refunded. It must be fixed at quote time, before the client's address is necessarily known. There are two strategies:

- **Client-provided (RECOMMENDED)**: The resource server collects the client's refund address before issuing the wet quote — via a registration step, or a pre-flight mechanism in which the client's initial unauthenticated request carries its origin-chain address (e.g., in a request header), allowing the quote to set `refundTo = clientAddress`.
- **Facilitator-default**: The facilitator uses a default refund address (e.g., a custodial address that the facilitator manages, later disbursing refunds to clients). This reintroduces custodial trust in the refund path and SHOULD only be used where pre-flight collection is impractical.

> **Recommendation**: For production deployments, use the client-provided strategy so that every failure path refunds directly to the client's own origin-chain wallet with no intermediary.

### SDK Integration Notes

Client-side handling of this method is deliberately close to a plain on-chain transfer:

1. **Selection**: Filter the `accepts` array for `scheme: "upfront"` entries whose `network` matches an available wallet and whose `extra.assetTransferMethod` is supported (`"1click-swap"`). Standard network-based wallet selection applies; no secondary routing field is needed.
2. **Risk policy**: Per the `upfront` scheme, clients SHOULD prefer a scheme with post-execution settlement when one is offered for the same resource, and MAY apply method-specific policies keyed on `extra.assetTransferMethod` / `extra.settlementBackend`.
3. **Payment**: Construct a native transfer of `amount` of `asset` to `payTo` (attaching `extra.depositMemo` where non-null), submit it before `extra.deadline`, and record the transaction hash.
4. **Proof presentation**: Retry the request with the `PaymentPayload` carrying `txHash` as the payment proof. No off-chain signature is constructed; the client's only signature is the deposit transaction itself.
5. **Timeouts**: Set the HTTP client timeout from `maxTimeoutSeconds`, and expect settlement latency on the order of `extra.timeEstimate`.

---

## Version History

| Version | Date       | Changes       | Authors                 |
| ------- | ---------- | ------------- | ----------------------- |
| v1.0    | 2026-06-12 | Initial draft | @IkerAlus               |
