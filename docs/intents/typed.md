# Typed JSON: define your own request

Use typed intents for an application-specific JSON message, such as an expense,
release request or internal transfer. The authority declares which fields exist and
their types. A policy then reads those fields by name.

If your input is an EVM/Bitcoin transaction or an AP2/VI mandate, use its native
module so the policy checks the decoded protocol content.

## Start with a complete example

Input:

```json
{"recipient": "office-shop", "currency": "USD", "amount": "49.99"}
```

Authority:

```yaml
schemaVersion: verdict.authority/v1
id: com.acme.office.purchase
type: custom
version: 1.0.0
config:
  fields:
    recipient: {type: string}
    currency: {type: string}
    amount: {type: decimal}
    urgent: {type: bool, required: false, default: false}
policy:
  id: office-purchase
  fallback: DENY
  variables:
    officeRecipient: office-shop
    maxAmount: "100.00"
  allow:
    - id: approved-purchase
      where:
        - "recipient == officeRecipient"
        - "currency == 'USD'"
        - "decimal.gt(amount, '0')"
        - "decimal.lte(amount, maxAmount)"
  deny:
    - id: urgent-needs-another-process
      where:
        - "urgent"
```

Download [typed-purchase.yaml](../authority/examples/typed-purchase.yaml).
The request is ALLOW: the module reads `amount` as a decimal and fills in `urgent: false`.
`"100.01"`, another recipient, EUR, or `urgent: true` produces DENY. Missing `amount`
is a validation error. [Your first policy](../getting-started.md) walks through every section.

## Add an optional nested field

This is a fragment to add under `config.fields`:

```yaml
customer:
  type: object
  fields:
    id: {type: string}
    country: {type: string, required: false, default: UNKNOWN}
```

The `customer` object and `customer.id` are required. If `country` is missing it
becomes `UNKNOWN`. A condition can now use `customer.country == 'DE'`.
Without a default, an optional missing field becomes `null`; guard it with
`customer.country != null` before a comparison requiring a concrete value.

## Add a list and check every element

Fragment under `config.fields`:

```yaml
items:
  type: list
  items:
    type: object
    fields:
      sku: {type: string}
      quantity: {type: int}
```

For `{"items":[{"sku":"paper","quantity":2}]}`, add these conditions to your
allow rule alongside its other checks:

```cel
size(items) > 0
items.all(item, item.sku in ['paper', 'pens'] && item.quantity > 0 && item.quantity <= 10)
```

An empty list fails the first condition; an unlisted SKU or invalid quantity fails
the second. Ordinary CEL integer comparisons work here because `quantity` is `int`.

## Map fields to an effect only when useful

You can write a policy directly against the declared fields. If several application
messages share the same consequence, an effect gives them a common shape. For the
purchase example, add this under `config` next to `fields`:

```yaml
effects:
  - type: office.purchase
    fields:
      to: "$recipient"
      amount: "$amount"
      currency: "$currency"
```

The module creates `effects` from validated fields. `$amount` copies the input field;
`USD` without `$` would be a literal. `$$value` means the literal string `$value`.
These effects use **your chosen units**. For this decimal-dollar example, read the
amount with `decimal.*`. The `effect.amount` and `bigint.*` helpers require integers.

Unknown JSON fields are ignored. Declare and constrain every field that affects
the signed action, or use the native decoder for its format.

## Config and API reference

Artifact: `org.exploit:verdict-intent-typed`.

Intent type: `custom`.

## Purpose

`TypedIntent` describes custom JSON request shapes directly in authority config.

Use it when no native domain intent exists.

## Alternative field schema

The fragment below shows integer `bigint` amounts and a nested customer object.
It is a different application schema from the decimal-dollar example above. Define
your amount unit explicitly and use `bigint.*` for its comparisons.

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

## Input validation

JSON must contain one object without duplicate keys or trailing documents. Effect fields cannot override the reserved `type` field. Paths must resolve exactly, without empty components. Defaults and effect literals are copied when configuration is created; later mutation of caller-owned collections or byte arrays does not change the configuration. A null default requires a nullable field.
