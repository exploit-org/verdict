# AP2 Mandate Intent

For YAML examples, see [payment policies](payments.md). For request construction,
see the [Java example](payments.md#java-integration).

Artifact: `org.exploit:verdict-ap2`

Intent type: `ap2.mandate` (`IntentTypes.AP2_MANDATE`)

`Ap2Intent` evaluates mandate content against local policy before signing, or after
an external component has verified a received mandate. It implements `Intent` and
works with `PolicyEvaluator`, including approval requirements.

The supported format is AP2 v0.2 at
the [pinned upstream commit](https://github.com/google-agentic-commerce/AP2/tree/e1ea56db72a6385bce3e5c1112b3a56ce60acb43).
The four exact `vct` values are:

| `vct`                     | Effect                   |
|---------------------------|--------------------------|
| `mandate.payment.1`       | `ap2.payment.authorize`  |
| `mandate.checkout.1`      | `ap2.checkout.authorize` |
| `mandate.payment.open.1`  | `ap2.payment.delegate`   |
| `mandate.checkout.open.1` | `ap2.checkout.delegate`  |

## Evaluation boundary

`fromJson` accepts one complete mandate JSON object. The caller extracts it from
any JWT, SD-JWT or credential envelope and handles signatures, disclosures, key
validation, trust, audience/nonce checks and digest bindings. Unresolved `_sd`,
`_sd_alg` and `...` disclosure fields are rejected recursively.

For a closed checkout, supply the decoded payload associated with the mandate's
`checkout_jwt`. The caller verifies this association before evaluation.
`cnf`, `checkout_jwt`, `checkout_hash`, `transaction_id` and reference digests are
exposed as supplied data for policy conditions.

Open-mandate policies inspect the proposed delegation constraints. The application
enforces them on later payments and handles delegation chains, fulfillment,
cumulative budgets, recurrence and replay. Add expiration conditions to the policy
when needed; the reader validates timestamp format and ordering.

## Action policies and complete signing requests

For signing flows use `Ap2Intent.request(inputs, config)` and
`PolicyEvaluator.evaluate(compiledPolicy, request)`. Each action receives the
same policy; one denied action denies the whole request.

Start with [shared action policies](payments.md) and the complete
[authority example](../authority/examples/ap2-payment.yaml).
A `PAIRED` request groups concrete pairs as `purchase` and open pairs as `delegation`.
Explicit `PAYMENTS` / `CHECKOUTS` modes accept only closed mandates for signing stages
that carry one side. Pairing compares references; all cryptographic bindings remain
with the credential adapter. The request retains every original input for signing.

The single-mandate APIs below evaluate one decoded mandate. Use `request(...)`
to evaluate all mandates covered by a signature.

## Raw CEL payment example

```java
import org.exploit.verdict.PolicyEvaluator;
import org.exploit.verdict.intent.ap2.Ap2Intent;
import org.exploit.verdict.model.Policy;

var intent = Ap2Intent.fromJson("""
        {
          "vct": "mandate.payment.1",
          "transaction_id": "digest-established-outside-verdict",
          "payee": {"id": "merchant-1", "name": "Shop"},
          "payment_amount": {"amount": 19900, "currency": "USD"},
          "payment_instrument": {"id": "card-1", "type": "card"}
        }
        """);

var policy = Policy.denyByDefault("ap2-payment")
        .allow("small-usd-payment", rule -> rule.where(
                "effect.onlyTypes(effects, ['ap2.payment.authorize'])",
                "effect.one(effects, 'ap2.payment.authorize')",
                "effect.any(effects, 'ap2.payment.authorize', {'asset': 'USD', 'to': 'merchant-1'})",
                "bigint.lte(effect.amount(effects, 'ap2.payment.authorize'), '20000')"))
        .build();

var evaluation = new PolicyEvaluator().evaluate(policy, intent);
```

Both `fromJson` and `fromCheckoutJson` accept Jackson `JsonNode` objects directly.
Overloads accept `String` or UTF-8 `byte[]` when the input has not yet been decoded.

## Closed checkout

```java
var intent = Ap2Intent.fromCheckoutJson(mandateContentJson, decodedCheckoutJson);
```

The supported checkout profile is the UCP shape distributed with the pinned AP2 SDK:
`id`, `merchant`, `line_items`, `status`, `currency`, `totals`, and `links` are required.
Each line contains `id`, `item` (`id`, `title`, `price`), positive `quantity`, and `totals`.
The checkout must have exactly one `subtotal` and one `total`. A total category may not
repeat. Optional fields supported by the reader remain available in `checkout`.
The declared final total is exposed without reconstructing pricing, discounts or tax.
Policies should check status, line items and pricing as appropriate to their use case.

This reader supports the UCP checkout profile described above and rejects unknown
checkout fields and extensions.

## Variables and effects

Mandate fields appear at the CEL root and under `mandate`. `open` distinguishes
delegation from authorization of a concrete action. A closed checkout additionally
exposes `checkout`. All returned maps and lists are recursively immutable.

`effects` always has exactly one entry containing the complete mandate fields plus
its normalized effect `type`. Authorization effects also expose:

| Field      | Payment                   | Checkout                                       |
|------------|---------------------------|------------------------------------------------|
| `amount`   | `payment_amount.amount`   | Checkout `totals` entry with `type == 'total'` |
| `asset`    | `payment_amount.currency` | Checkout `currency`                            |
| `to`       | `payee.id`                | Checkout `merchant.id`                         |
| `checkout` | Absent                    | Complete decoded checkout                      |

Payment amounts and checkout prices are integer ISO-4217 minor units: `19900 USD`
means USD 199.00. Integer JSON values use `BigInteger`, including quantities and Unix
timestamps; use `bigint.*` helpers in CEL. Decimal metadata and `payment.budget.max`
use `BigDecimal`. The upstream budget schema defines `max` as a number with
unspecified units; Verdict preserves its numeric value. Agree on the unit in your
integration before writing budget conditions.

Delegation effects contain `cnf`, the full `constraints` list and any preset fields.
Their limits remain in the constraints; the normalized `amount` field is absent.
For concrete payments, require `ap2.payment.authorize` in the rule.

## Validation profile

The text and byte readers reject malformed JSON, duplicate keys and trailing content.
For `JsonNode` input, configure the application's decoder as shown in
[Java integration](payments.md#java-integration). All overloads reject unsupported `vct` versions,
unknown mandate fields and constraint types, missing required fields, nulls in typed
fields, negative payment amounts, fractional minor-unit amounts and unknown currency
codes. Dates require ISO-8601 date-times with an offset; creation/expiration values
are nonnegative integers and, when both exist, `exp` must exceed `iat`.

All eight payment constraint types and both checkout constraint types in the pinned
specification are structurally supported. Open payment requires `payment.reference`;
open checkout requires `checkout.line_items`. Allow lists and acceptable item lists
must be nonempty; quantities and recurrence counts must be positive; range bounds
must be ordered; budgets require recurrence. Unknown fields inside these structures
are rejected. `cnf` is required to be a nonempty object and remains opaque.

The policy evaluates whether the combined constraints grant acceptable authority.
`risk_data` and checkout messages are preserved as caller-supplied informational JSON.

## Tests

```bash
./gradlew :verdict-ap2:test
```
