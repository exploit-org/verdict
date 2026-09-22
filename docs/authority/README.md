# Authority files: config and policy in one document

An authority is the YAML file your application loads to decide which actions it can
sign. Start by copying a complete example for your use case:

| Use case                       | File                                                    |
|--------------------------------|---------------------------------------------------------|
| Custom JSON purchase           | [typed-purchase.yaml](examples/typed-purchase.yaml)     |
| Purchase requiring approval    | [typed-approvals.yaml](examples/typed-approvals.yaml)   |
| EVM token transfer             | [evm-transfer.yaml](examples/evm-transfer.yaml)         |
| Bitcoin payment with change    | [bitcoin-transfer.yaml](examples/bitcoin-transfer.yaml) |
| Certificate names and validity | [x509-server.yaml](examples/x509-server.yaml)           |
| AP2 purchases                  | [ap2-payment.yaml](examples/ap2-payment.yaml)           |
| Mastercard VI purchases        | [mcintent-payment.yaml](examples/mcintent-payment.yaml) |

## What goes where?

```yaml
schemaVersion: verdict.authority/v1
id: com.acme.office.purchase
type: custom
version: 1.0.0
metadata:
  title: Office purchases
config:
  fields:
    amount: {type: decimal}
policy:
  id: positive-small-amount
  fallback: DENY
  variables:
    maxAmount: "100.00"
  allow:
    - id: within-limit
      where:
        - "decimal.gt(amount, '0')"
        - "decimal.lte(amount, maxAmount)"
```

This small structural example checks only a decimal amount. For recipient/currency
checks, use the complete purchase example above.

| Field              | Who uses it                    | What to put there                                             |
|--------------------|--------------------------------|---------------------------------------------------------------|
| `schemaVersion`    | Document parser                | Exactly `verdict.authority/v1`                                |
| `id`               | Authority loader/integration   | Stable authority name; registry lookup must match             |
| `type`             | Integration's intent routing   | For example `custom`, `evm.transaction`, `ap2.mandate`        |
| `version`          | Release/version management     | Your authority revision, such as `1.0.0`                      |
| `metadata`         | People and application tooling | Optional title, labels, descriptions                          |
| `config`           | Matching intent module         | Decoding schema, known contracts or payment identity catalogs |
| `policy.variables` | CEL evaluation                 | Constants you reference by name in conditions                 |
| `policy.allow`     | Policy evaluation              | Alternative permitted cases                                   |
| `policy.deny`      | Policy evaluation              | Conditions that forbid the action even if an allow matched    |
| `policy.approvers` | Approval requirements          | Named approver keys; see [approvals](../approvals.md)         |

Config defines decoding and identity lookup. Policy rules authorize actions.
An EVM contract entry enables decoding its calls; a payment shop entry defines an
identity for rules to match. Each module specifies the fields it exposes to CEL.

The parser rejects unknown document fields, duplicate YAML/JSON keys, and trailing
documents. Supported fields are listed above. `version` labels an authority revision;
manage Verdict dependencies separately.

## Load and evaluate locally

Artifact: `org.exploit:verdict-authority`. JSON and YAML are supported.

```java
var authority = new AuthorityParser().parse(Path.of("authority.yaml"));
var config = TypedIntentConfig.from(authority.config());
var intent = TypedIntent.fromJson(payloadJson, config);
var evaluator = new PolicyEvaluator();
var compiled = evaluator.compile(authority.policy());
var result = evaluator.evaluate(compiled, intent);
```

This example explicitly chooses the typed module. Select the intent module matching
`authority.type()`, validate its configuration, and compile the policy with the
module's input schema. Full imports and a strict example are
in [getting started](../getting-started.md#5-run-it-from-java).

## Load by an application-controlled authority ID

```java
var registry = AuthorityRegistry.builder()
        .source("com.acme.office.purchase", AuthorityFileSource.of(Path.of("authority.yaml")))
        .build();
var loaded = registry.load("com.acme.office.purchase");
```

The registry checks the document ID. `loaded.authority()` is the document;
`loaded.artifact()` contains the loaded artifact information. The registry also has
`compile(id)` for policies using its configured evaluator/compiler.

For payments, register `PaymentFunctions` with the evaluator and use
`PaymentRequest.policySchema()` with `compileStrict`. See the
[payment Java example](../intents/payments.md#java-integration).

Load the authority reference and config from trusted application state. For registry-distributed files, see
[digest-pinned OCI loading](oci.md).
