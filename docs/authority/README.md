# Authority Documents

Artifact: `org.exploit:verdict-authority`.

An authority is an immutable policy bundle:

```yaml
schemaVersion: verdict.authority/v1
id: com.acme.prod.evm.treasury
type: evm.transaction
version: 1.2.0

metadata:
  title: Treasury policy

config:
  chainId: 1

policy:
  id: treasury
  fallback: DENY
  allow:
    - id: erc20-transfer
      where:
        - "effect.one(effects, 'erc20.transfer')"
  deny: []
```

## Fields

| Field | Required | Meaning |
| --- | --- | --- |
| `schemaVersion` | yes | Must be `verdict.authority/v1`. |
| `id` | yes | Stable authority id. |
| `type` | yes | Intent type. |
| `version` | yes | Human/release version. |
| `metadata` | no | Human-only labels and description. |
| `config` | no | Type-specific intent config. |
| `policy` | yes | One Verdict policy. |

`requires` is intentionally not part of v1.

## Java

```java
AuthorityRegistry registry = AuthorityRegistry.builder()
    .source("com.acme.prod.evm.treasury", AuthorityFileSource.of(Path.of("authority.yaml")))
    .build();

LoadedAuthority loaded = registry.load("com.acme.prod.evm.treasury");
CompiledAuthority compiled = registry.compile("com.acme.prod.evm.treasury");
```

`AuthorityRegistry` checks that the resolved document id equals the requested id.

## Config Routing

Pass `authority.config()` to the matching intent config:

```java
EvmIntentConfig evm = EvmIntentConfig.fromMap(authority.config());
BitcoinIntentConfig bitcoin = BitcoinIntentConfig.fromMap(authority.config());
TypedIntentConfig typed = TypedIntentConfig.from(authority.config());
```

## Trust

Production trust is based on digest-pinned artifact references.

See [OCI authority loading](oci.md).
