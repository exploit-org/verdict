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

All artifacts use group `org.exploit.verdict`.

| Artifact | Purpose |
|----------|---------|
| `core` | Policy compiler and evaluator. |
| `digital-assets` | EVM, Bitcoin, TRON, Solana, and XRP modules. |
| `evm`, `bitcoin`, `tron`, `solana`, `xrp` | Individual digital asset modules. |
| `agentic-payments` | AP2 and Mastercard VI modules. |
| `ap2`, `mc-vi` | Individual agent payment modules. |
| `payments` | Shared AP2 and VI policy functions. |
| `typed` | Custom JSON intents. |
| `x509` | Certificate issuance intents. |
| `authority` | Authority document parser and compiler. |
| `authority-oci` | OCI authority loader. |

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
