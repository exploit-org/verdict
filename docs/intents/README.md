# Intent Modules

Intent modules decode domain requests into root CEL variables and normalized `effects`.

Unknown or partially described consequences are rejected before policy evaluation.

## Modules

| Intent | Type | Artifact | Docs |
| --- | --- | --- | --- |
| X.509 TBSCertificate | `x509.tbs-certificate` | `verdict-intent-x509` | [x509.md](x509.md) |
| EVM transaction | `evm.transaction` | `verdict-intent-evm` | [evm.md](evm.md) |
| Bitcoin-like transaction | `bitcoin.transaction` | `verdict-intent-bitcoin` | [bitcoin.md](bitcoin.md) |
| Typed JSON | `custom` | `verdict-intent-typed` | [typed.md](typed.md) |

Shared effect rules: [effects.md](effects.md).

## Design Rules

- Intent data is flattened into root variables for CEL ergonomics.
- `transaction`, `call`, `inputs`, `outputs`, and similar aliases remain available where useful.
- `effects` is the primary surface for consequences.
- Config comes from trusted authority config, not user payload.
- Unknown contracts, functions, scripts, or effects fail closed.

## Base64 Inputs

TKeeper-style payloads are Base64-first where raw bytes are expected:

- `EvmTransactionIntent.fromBase64(...)`
- `BitcoinSigningIntent.unsignedTransactionBase64(...)`
- `BitcoinSigningIntent.previousTransactionBase64(...)`
- `TbsCertificateIntent.fromDerBase64(...)`

Hex methods remain available for debugging and integrations.
