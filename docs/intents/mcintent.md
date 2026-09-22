# Mastercard Verifiable Intent

For YAML examples, see [payment policies](payments.md). For request construction,
see the [Java example](payments.md#java-integration).

Artifact: `org.exploit:verdict-mcintent`

Intent type: `mcintent.mandate` (`IntentTypes.MCINTENT_MANDATE`)

`McIntent` implements `Intent` for
the [pinned Verifiable Intent v0.1 draft](https://github.com/agent-intent/verifiable-intent/tree/356c29635f1c44df7de02edb58699ca9f29bece6).
It evaluates a proposed mandate before signing, or mandate content extracted by an
external credential processor, through `PolicyEvaluator`.

## Inputs and cryptography boundary

`fromJson` accepts one complete mandate JSON object. The caller extracts it from
any JWT, SD-JWT or credential envelope and handles signatures, disclosures, trust,
key validation and digest bindings. Unresolved `_sd`, `_sd_alg` and `...` fields
are rejected recursively, including inside extension data.

The supported exact `vct` values and effects are:

| `vct`                     | Effect                        |
|---------------------------|-------------------------------|
| `mandate.payment.1`       | `mcintent.payment.authorize`  |
| `mandate.checkout.1`      | `mcintent.checkout.authorize` |
| `mandate.payment.open.1`  | `mcintent.payment.delegate`   |
| `mandate.checkout.open.1` | `mcintent.checkout.delegate`  |

Closed content can originate in Immediate L2 or Autonomous L3; the caller identifies
and verifies its signing layer. Open content describes a grant of future authority.
It must contain `cnf.jwk.kid` for policy inspection. The caller authenticates the key.

## Action policies and complete signing requests

For signing flows use `McIntent.request(inputs, config)` and
`PolicyEvaluator.evaluate(compiledPolicy, request)`. Each action receives the
same policy; one denied action denies the whole request.

Start with [shared action policies](payments.md) and the complete
[authority example](../authority/examples/mcintent-payment.yaml).
A `PAIRED` request groups concrete pairs as `purchase` and open pairs as `delegation`.
Explicit `PAYMENTS` / `CHECKOUTS` modes accept only closed mandates for signing stages
that carry one side. Pairing compares references; all cryptographic bindings remain
with the credential adapter. The request retains every original input for signing.

The single-mandate APIs below evaluate one decoded mandate. Use `request(...)`
to evaluate all mandates covered by a signature.

## Raw CEL payment example

```java
import org.exploit.verdict.PolicyEvaluator;
import org.exploit.verdict.intent.mcintent.McIntent;
import org.exploit.verdict.model.Policy;

var intent = McIntent.fromJson("""
        {
          "vct": "mandate.payment.1",
          "transaction_id": "digest-established-outside-verdict",
          "payee": {"name": "Shop", "website": "https://shop.example"},
          "payment_amount": {"currency": "USD", "amount": 27999},
          "payment_instrument": {"type": "mastercard.srcDigitalCard", "id": "card-1"}
        }
        """);

var policy = Policy.denyByDefault("mc-payment")
        .allow("approved-shop", rule -> rule.where(
                "effect.onlyTypes(effects, ['mcintent.payment.authorize'])",
                "effect.one(effects, 'mcintent.payment.authorize')",
                "payee.name == 'Shop' && payee.website == 'https://shop.example'",
                "payment_amount.currency == 'USD'",
                "bigint.lte(effect.amount(effects, 'mcintent.payment.authorize'), '30000')"))
        .build();

var evaluation = new PolicyEvaluator().evaluate(policy, intent);
```

`fromJson` accepts Jackson `JsonNode`, `String`, and UTF-8 `byte[]`. Monetary amounts and budget bounds
require integer JSON numbers in ISO-4217 minor units. The reader exposes them as
`BigInteger`; compare them with `bigint.*` functions.

## Checkout example

```java
var intent = McIntent.fromCheckoutJson(mandateContentJson, decodedCheckoutJson);
```

Both arguments accept Jackson `JsonNode`, `String`, or UTF-8 `byte[]`. The caller supplies the complete
decoded content belonging to the mandate's `checkout_jwt` and verifies their binding.
The checkout must be a nonempty JSON object.

The text and byte readers reject duplicate keys and trailing content. For `JsonNode`
input, configure the application's decoder as shown in [Java integration](payments.md#java-integration).

VI v0.1 leaves the checkout schema to the integration. The reader exposes it unchanged
under `checkout`, both at the root and inside the effect. Write conditions using your
integration's field names and amount units.

The optional final-mandate `line_items` list is preserved separately. Entries require
`item.id` and a positive integer `quantity`. The application checks their consistency
with the checkout.

## Variables and effects

Mandate fields appear at the CEL root and under `mandate`. `open` identifies
delegation content. `effects` contains exactly one effect with the complete mandate
fields plus its normalized `type`. All maps and lists are recursively immutable;
integer JSON values are `BigInteger` and decimal extension data is `BigDecimal`.

A payment authorization effect additionally contains:

| Field    | Value                                                                  |
|----------|------------------------------------------------------------------------|
| `amount` | `payment_amount.amount`                                                |
| `asset`  | `payment_amount.currency`                                              |
| `to`     | Complete `payee` object, including `name`, `website` and optional `id` |

The effect preserves the full recipient because VI makes the opaque payee ID optional.
Delegation effects expose `constraints`, `cnf` and the other mandate fields without
a normalized transfer amount. Checkout effects expose the complete `checkout` object.
Policies should explicitly require the expected effect type.

## Constraint content

All eight registered constraint types are structurally validated and exposed:

- `mandate.checkout.allowed_merchants`
- `mandate.checkout.line_items`
- `mandate.payment.allowed_payees`
- `mandate.payment.amount_range`
- `mandate.payment.budget`
- `mandate.payment.recurrence`
- `mandate.payment.agent_recurrence`
- `mandate.payment.reference`

The reader follows VI-specific semantics: merchant identity requires `name` and
`website`; line-item `acceptable_items: []` is a wildcard; amount-range bounds are
optional; `match_mode` accepts `minimum` or `exact`. Recurrence uses ISO 20022 codes
and ISO dates. Agent recurrence additionally permits `ON_DEMAND`, requires an end
date, and requires companion amount-range and budget constraints. Open payment
always requires a reference constraint and a payment instrument.

Open mandates require at least one constraint. Unknown types and wrong-domain
constraints are rejected. Duplicate types are preserved for independent policy
inspection. Unknown fields inside registered constraints are preserved as required
by the specification, including nested extension objects. Write explicit policy
conditions for extensions supported by your integration.

The validation profile rejects unknown mandate fields, empty allowlists, invalid
currency codes, negative amounts, nonpositive quantities/counts, reversed ranges,
reversed dates and duplicate line-requirement IDs. Closed mandates reject `cnf`
and `constraints`.

## Evaluation responsibility

A CEL policy decides whether the proposed delegation is acceptable. For subsequent
payments, the application enforces the grant, verifies the credential chain, handles
receipts and replay, and tracks cumulative or recurring spending. It can supply
trusted state to the policy through evaluation context.

`risk_data`, `prompt_summary`, descriptions, key IDs and digest strings are exposed
as caller-supplied data. Authenticate their source before relying on them.

## Tests

```bash
./gradlew :verdict-mcintent:test
```
