![](assets/verdict-banner.png)

# Verdict

Verdict checks whether an action may be signed or issued. Write the conditions in
an authority YAML file; your application supplies the intent and handles signing.
Requires Java 25+.

**New to writing policies? [Start with one complete example](docs/getting-started.md).**
It shows the input, every YAML field, and the requests that pass or fail.

For agent payments, go straight to [AP2 and Mastercard payment policies](docs/intents/payments.md).
For other actions, [choose an intent guide](docs/intents/README.md).

## Modules

| Artifact                             | Purpose                                                                                              |
|--------------------------------------|------------------------------------------------------------------------------------------------------|
| `org.exploit:verdict`                | Core policy model, builder, compiler, evaluator, and CEL functions.                                  |
| `org.exploit:verdict-intent-x509`    | RFC 5280 `TBSCertificate` intent for pre-issuance checks.                                            |
| `org.exploit:verdict-intent-evm`     | Unsigned EVM transaction intent with ABI/effect mapping.                                             |
| `org.exploit:verdict-intent-bitcoin` | Unsigned Bitcoin UTXO transaction intent.                                                            |
| `org.exploit:verdict-intent-typed`   | Declarative JSON typed intent for custom request shapes.                                             |
| `org.exploit:verdict-ap2`            | AP2 v0.2 mandate content evaluation, without cryptography.                                           |
| `org.exploit:verdict-mcintent`       | Mastercard Verifiable Intent v0.1 draft mandate content evaluation.                                  |
| `org.exploit:verdict-payments`       | Shared merchant/method config, action policy functions, and whole-request evaluation for AP2 and VI. |
| `org.exploit:verdict-authority`      | Authority document parser, loader, compiler, and registry.                                           |
| `org.exploit:verdict-authority-oci`  | Digest-pinned OCI authority loading.                                                                 |

## Documentation

Start here: [docs/README.md](docs/README.md).

- [Getting started](docs/getting-started.md)
- [Policy language: AND/OR, lists and amounts](docs/policy-language.md)
- [Approval rules](docs/approvals.md)
- [Troubleshooting](docs/troubleshooting.md)
- [Core policy engine](docs/core-policy-engine.md)
- [Built-in CEL functions](docs/cel-functions.md)
- [Intent modules](docs/intents/README.md)
- [Effect semantics](docs/intents/effects.md)
- [Authority documents](docs/authority/README.md)
- [Payment policies: named shops, cards and readable limits](docs/intents/payments.md)

## Running tests

```bash
./gradlew test
```
