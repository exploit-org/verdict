# Verdict Documentation

Verdict evaluates policy over typed signing and issuance contexts.

## Read Order

1. [Getting started](getting-started.md)
2. [Core policy engine](core-policy-engine.md)
3. [Built-in CEL functions](cel-functions.md)
4. [Intent modules](intents/README.md)
5. [Effect semantics](intents/effects.md)
6. [Authority documents](authority/README.md)

## Concepts

| Concept | Meaning |
| --- | --- |
| Policy | Allow/deny rules compiled to CEL. |
| Intent | Typed evaluation context for a request. |
| Effect | Normalized consequence exposed as a map with `type`. |
| Authority | Versioned policy bundle with type-specific config. |
| OCI authority | Digest-pinned authority artifact pulled from an OCI registry. |

## Module Docs

| Module | Docs |
| --- | --- |
| Core engine | [core-policy-engine.md](core-policy-engine.md) |
| CEL helpers | [cel-functions.md](cel-functions.md) |
| X.509 TBSCertificate | [intents/x509.md](intents/x509.md) |
| EVM transactions | [intents/evm.md](intents/evm.md) |
| Bitcoin-like transactions | [intents/bitcoin.md](intents/bitcoin.md) |
| Typed JSON intents | [intents/typed.md](intents/typed.md) |
| Effect semantics | [intents/effects.md](intents/effects.md) |
| Authority documents | [authority/README.md](authority/README.md) |
| OCI authority loading | [authority/oci.md](authority/oci.md) |

## Runtime Baseline

- Java 21+
- Gradle wrapper from the repository
- Google CEL through `dev.cel:cel`
