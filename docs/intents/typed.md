# Typed JSON Intent

Artifact: `org.exploit:verdict-intent-typed`.

Intent type: `custom`.

## Purpose

`TypedIntent` describes custom JSON request shapes directly in authority config.

Use it when no native domain intent exists.

## Authority Config

```yaml
type: custom
config:
  fields:
    amount:
      type: bigint
    currency:
      type: string
    customer:
      type: object
      fields:
        id:
          type: string
        country:
          type: string
          required: false
          default: UNKNOWN
  effects:
    - type: payment.transfer
      fields:
        asset: "$currency"
        amount: "$amount"
        customerId: "$customer.id"
```

## Java

```java
TypedIntentConfig config = TypedIntentConfig.from(authority.config());
TypedIntent intent = TypedIntent.fromJson(payload, config);
```

## Rules

- Only `application/json` payloads are supported.
- JSON root must be an object.
- Only declared fields become root variables.
- Unknown JSON fields are ignored.
- `effects` is a reserved field name.
- `required` defaults to `true`.
- `nullable` defaults to `false`.
- Optional missing fields without `default` are exposed as `null`.
- Config typos are rejected.

## Types

| Type | Java value |
| --- | --- |
| `string` | `String` |
| `bool` | `Boolean` |
| `int` | `Long` |
| `bigint` | `BigInteger` |
| `decimal` | `BigDecimal` |
| `time` | `Instant` |
| `bytes` | `byte[]` from Base64 |
| `object` | nested map |
| `list` | list |

## Effects

String values starting with `$` are resolved as paths. Use `$$value` for a literal string beginning with `$`.
