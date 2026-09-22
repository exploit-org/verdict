# Choose an intent module

An intent module turns an actual request into the fields a policy reads. For example,
the EVM module decodes serialized transaction bytes into `chainId`, `call` and
`effects`. Policies read those decoded fields.

| Your input | Artifact | Authority `type` | Guide |
|------------|----------|------------------|-------|
| Unsigned EVM transaction | `evm` | `evm.transaction` | [EVM](evm.md) |
| Unsigned Bitcoin transaction and previous transactions | `bitcoin` | `bitcoin.transaction` | [Bitcoin](bitcoin.md) |
| Unsigned TRON transaction | `tron` | `tron.transaction` | [TRON](tron.md) |
| Unsigned Solana transaction | `solana` | `solana.transaction` | [Solana](solana.md) |
| Unsigned XRP Payment or atomic Batch | `xrp` | `xrp.transaction` | [XRP](xrp.md) |
| DER TBSCertificate body | `x509` | `x509.tbs-certificate` | [X.509](x509.md) |
| Custom JSON | `typed` | `custom` | [Typed JSON](typed.md) |
| AP2 mandate content | `ap2` | `ap2.mandate` | [AP2](ap2.md) |
| Mastercard VI mandate content | `mc-vi` | `mcintent.mandate` | [VI](mcintent.md) |

Artifacts use group `org.exploit.verdict`. `digital-assets` includes the five chain
modules; `agentic-payments` includes AP2 and VI. Use one Verdict version throughout.

## What becomes available to CEL?

Each guide lists its exact root fields. There is no universal `amount`, `recipient`
or `config` variable. For example:

- EVM: `effect.amount(effects, 'erc20.transfer')` is integer token base units.
- Bitcoin: `fee` is integer satoshis; `outputs` contains all outputs.
- Typed JSON: declared field names become roots, with the configured types.
- X.509: use `extensions`, `validity` and other certificate fields; no generic effects list.
- Payment requests: `payment` and `checkout` describe the current action;
  `request` describes the whole request. Verdict checks every action automatically.

Use [effect helpers](effects.md) where the module supplies effects. Use the
[readable payment helpers](payments.md) for AP2/VI action policies.

## Application flow

```text
authority.config + original request
        → matching intent module
        → evaluator.evaluate(compiledPolicy, intent)
        → ALLOW / DENY / ALLOW_WITH_REQUIREMENTS
        → application's approval and signing logic
```

A module can reject malformed or unsupported content before evaluation. Its guide
explains which cases are supported. Typed JSON ignores undeclared fields, so its
schema must describe every input field relevant to the action being authorized.

Byte-based modules support Base64 for transport:
`EvmTransactionIntent.fromBase64`, `BitcoinSigningIntent`'s Base64 builder methods,
and `TbsCertificateIntent.fromDerBase64`. Payment readers take decoded JSON;
the caller handles JWTs and cryptography.
