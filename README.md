![](assets/verdict-banner.png)
# Verdict

Java 21+ policy engine over Google CEL.

## Modules

| Artifact | Purpose |
| --- | --- |
| `org.exploit:verdict` | Core policy model, builder, compiler, evaluator, and CEL functions. |
| `org.exploit:verdict-intent-x509` | RFC 5280 `TBSCertificate` intent for pre-issuance checks. |
| `org.exploit:verdict-intent-evm` | Unsigned EVM transaction intent with ABI/effect mapping. |
| `org.exploit:verdict-intent-bitcoin` | Unsigned Bitcoin-like UTXO transaction intent. |
| `org.exploit:verdict-intent-typed` | Declarative JSON typed intent for custom request shapes. |
| `org.exploit:verdict-authority` | Authority document parser, loader, compiler, and registry. |
| `org.exploit:verdict-authority-oci` | Digest-pinned OCI authority loading. |

## Documentation

Start here: [docs/README.md](docs/README.md).

Most-used pages:

- [Getting started](docs/getting-started.md)
- [Core policy engine](docs/core-policy-engine.md)
- [Built-in CEL functions](docs/cel-functions.md)
- [Intent modules](docs/intents/README.md)
- [Effect semantics](docs/intents/effects.md)
- [Authority documents](docs/authority/README.md)
- [Maven Central publishing](docs/release/maven-central.md)

## Build

```bash
./gradlew test
```

## Release

```bash
export CENTRAL_MAVEN_TOKEN='<central-portal-token>'
export SIGNING_KEY="$(gpg --armor --export-secret-keys <KEY_ID>)"
export SIGNING_PASSWORD='<gpg-passphrase>'

./gradlew clean publishAllToMavenCentralPortal -PreleaseVersion=0.1.0
```

The Maven Central bundle is uploaded with `publishingType = USER_MANAGED`; release it manually in Maven Central Portal after validation.
