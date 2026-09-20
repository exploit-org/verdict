# AP2 and Mastercard: write a payment policy

Start with one sentence: **“Allow office purchases from this shop, using this card,
up to USD 100 per purchase and USD 150 for the complete request.”** This page turns
that sentence into a policy. The same conditions work for Google AP2 and Mastercard VI.

## What is being checked?

Your integration receives the protocol content, builds an intent, asks Verdict for a
decision, then signs the evaluated content if authorized. Signing, SD-JWT assembly
and cryptographic verification remain in the integration.

A request can contain several actions. **You write the rule for one action; Verdict
runs it for every action.** If one action is denied, the entire request is denied.

| Action kind | What you are authorizing | Fields to check |
| --- | --- | --- |
| `purchase` | A concrete checkout and its payment together | `checkout`, `payment` |
| `payment` | A concrete payment at a payment-only signing stage | `payment` |
| `checkout` | A concrete checkout at a checkout-only signing stage | `checkout` |
| `delegation` | Permission to make future purchases within proposed bounds | `delegation` |

A **checkout** is the order/cart description. A **payment** specifies who receives
money, the method and the amount. A **delegation** grants future authority: a maximum
on a delegation bounds future spending. Start each allow rule with the permitted
action kind.

## The data seen by one purchase rule

Relevant fields produced by the mandate decoder:

```json
{
  "action": {"kind": "purchase", "index": 0},
  "checkout": {
    "vct": "mandate.checkout.1",
    "content": {"merchant": {"id": "merchant-1", "website": "https://office.example"}}
  },
  "payment": {
    "vct": "mandate.payment.1",
    "payee": {"id": "merchant-1", "website": "https://office.example"},
    "payment_instrument": {"id": "card-1", "type": "card"},
    "payment_amount": {"currency": "USD", "amount": 7500}
  },
  "request": {"actionCount": 2, "allAmountsKnown": true, "totals": {"USD": 15000}}
}
```

This action spends USD 75.00. Another action also spends USD 75.00, so the request
total is USD 150.00. `payment` changes when Verdict evaluates the next action;
`request` describes the same complete request each time.

## Copy this authority

```yaml
schemaVersion: verdict.authority/v1
id: com.acme.office.ap2
type: ap2.mandate
version: 2.0.0

config:
  merchants:
    officeShop:
      website: https://office.example
  methods:
    companyCard:
      id: card-1
      type: card

policy:
  id: office-purchases
  fallback: DENY
  variables:
    maxPerPurchase: "100.00"
    maxPerRequest: "150.00"

  allow:
    - id: office-purchase
      where:
        - "action.kind == 'purchase'"
        - "checkout.merchantIs(merchants.officeShop)"
        - "payment.payeeIs(merchants.officeShop)"
        - "payment.methodIs(methods.companyCard)"
        - "payment.amountAtMost(maxPerPurchase, 'USD')"

  deny:
    - id: request-total
      where:
        - "!request.totalAtMost(maxPerRequest, 'USD')"
```

Download [AP2 authority](../authority/examples/ap2-payment.yaml) or
[Mastercard VI authority](../authority/examples/mcintent-payment.yaml).
For VI, the authority uses `type: mcintent.mandate`; the policy expressions are the same.

## Read each part in plain language

| Part | Meaning |
| --- | --- |
| `config.merchants.officeShop` | A name for an exact merchant identity |
| `config.methods.companyCard` | A name for one specific card/method |
| `maxPerPurchase: "100.00"` | A policy constant, in major units (USD dollars here) |
| `action.kind == 'purchase'` | Only approve concrete checkout/payment pairs |
| `checkout.merchantIs(merchants.officeShop)` | The shop in the checkout must match this configured shop |
| `payment.payeeIs(merchants.officeShop)` | The recipient of the money must match this configured shop |
| `payment.methodIs(methods.companyCard)` | Use this particular configured card |
| `payment.amountAtMost(maxPerPurchase, 'USD')` | Current payment is in USD and no more than USD 100 |
| `!request.totalAtMost(maxPerRequest, 'USD')` in `deny` | Deny if the complete request is not priced entirely in USD within USD 150 |

The shop and payee are separate checks because a checkout's seller can differ from
the payment recipient, for example with an intermediary. Configure both identities
if your payment flow needs that. A merchant entry may use `id`, `website`, or both;
all supplied criteria must match. A method needs `id` and may also specify `type`.
A display name is never sufficient identity.

The module exposes the authority's identity catalogs as `merchants` and `methods`.
Pass a catalog entry to each helper, for example `payment.methodIs(methods.companyCard)`.
Accessing a missing catalog key raises an evaluation error.

## Try these cases

All purchases below use the configured shop, payee and method unless stated otherwise.

| Request | Decision | Reason |
| --- | --- | --- |
| One purchase of USD 100 | ALLOW | Meets both limits |
| Two purchases of USD 75 | ALLOW | Each <= 100; together <= 150 |
| Two purchases of USD 80 | DENY | Total 160 |
| One purchase of USD 100.01 | DENY | Individual limit exceeded |
| USD 10 plus EUR 10 | DENY | Request is not entirely in USD |
| One valid purchase plus one using another card | DENY | Every action must pass |
| One valid purchase plus a delegation | DENY | Delegation is not permitted by this rule |
| Payment-only action | DENY | Different signing stage; `kind` is not `purchase` |

The total check returns false for unpriced actions or mixed currencies.
It sums payments within the current request. Daily or monthly limits require
spending history supplied by the application.

## Common changes

These snippets replace or add conditions inside the same allow rule. Keep the other
checks when adding a new condition.

### Permit two shops

Add `backupShop` to `config.merchants`, then replace each shop condition as needed:

```cel
checkout.merchantIs(merchants.officeShop) || checkout.merchantIs(merchants.backupShop)
payment.payeeIs(merchants.officeShop) || payment.payeeIs(merchants.backupShop)
```

These conditions allow the seller and payee to match different catalog entries.
To require the same shop for both, use:

```cel
(checkout.merchantIs(merchants.officeShop) && payment.payeeIs(merchants.officeShop)) ||
(checkout.merchantIs(merchants.backupShop) && payment.payeeIs(merchants.backupShop))
```

### Bound a range or require an exact amount

```cel
payment.amountAtLeast('1.00', 'USD')
payment.amountAtMost('100.00', 'USD')
```

Use both conditions for USD 1–100, inclusive. For an exact charge use
`payment.amountEquals('49.99', 'USD')`. The protocol amount stays in integer minor
units; the helpers convert the quoted limit. Never write `7500` when you mean a
USD 75.00 helper limit: write `'75.00'` and `'USD'`.

### Restrict cart items

For the AP2 UCP checkout shape, add:

```cel
size(checkout.content.line_items) > 0
checkout.content.line_items.all(line, line.item.id in ['paper-a4', 'pens-blue'])
checkout.content.line_items.all(line, bigint.lte(line.quantity, '10'))
```

This permits the listed SKUs with at most ten units per line. Repeated SKUs are
checked separately on each line. Product restrictions come from these conditions;
`officeShop` identifies the merchant.
VI checkout JSON is integration-defined: use these paths only if your VI adapter
supplies this shape. The common merchant helper expects `content.merchant`.

### Require approval

Add an `approvals` block to the allow rule and define its named approvers under
`policy.approvers`, as shown in [the approval guide](../approvals.md). Keep the amount,
shop and method checks. For a request with several actions, every applicable approval
requirement must be satisfied before the integration signs it.

### Approve a payment-only stage

Use `action.kind == 'payment'`, retain the payment identity/method/amount checks, and
omit the checkout condition because no checkout is present. The integration must
select `PaymentRequestMode.PAYMENTS` as part of the signing operation's configuration.
An incomplete paired request must fail validation.

## Grant future spending authority separately

A delegation policy checks the proposed permissions for future purchases.
The application enforces those permissions when later payments occur.
For open checkout/payment pairs, use a rule such as:

```yaml
allow:
  - id: bounded-delegation
    where:
      - "action.kind == 'delegation'"
      - "delegation.merchantsLimitedTo([merchants.officeShop])"
      - "delegation.payeesLimitedTo([merchants.officeShop])"
      - "delegation.methodsLimitedTo([methods.companyCard])"
      - "delegation.maxAmountAtMost('100.00', 'USD')"
      - >-
        delegation.hasOnlyConstraints([
          'checkout.allowed_merchants', 'checkout.line_items',
          'payment.reference', 'payment.allowed_payees',
          'payment.allowed_payment_instruments', 'payment.amount_range'
        ])
```

This is a **replacement policy fragment for a delegation-specific authority**. Keep
`fallback: DENY` and the identity catalogs. Remove the purchase example's
`!request.totalAtMost(...)` deny rule: delegations are unpriced and fail that check.

| Function | Question it answers |
| --- | --- |
| `merchantsLimitedTo([...])` | Is there an explicit merchant restriction, with every candidate allowed here? |
| `payeesLimitedTo([...])` | Are all fixed/allowed payment recipients within these definitions? |
| `methodsLimitedTo([...])` | Are all fixed/allowed payment methods within these definitions? |
| `maxAmountAtMost('100.00', 'USD')` | Is there an explicit fixed amount or maximum no greater than USD 100? |
| `hasOnlyConstraints([...])` | Are all constraint types in this list? |

Missing bounds return false. The lists must be nonempty. `hasOnlyConstraints`
checks the types of constraints present; use bound helpers to require specific limits.
To restrict products, also check the values inside `checkout.line_items`.
This fragment limits identities and per-payment amounts. The application handles
cumulative spending, recurrence, fulfillment, replay and delegation-chain verification.

## Java integration

Create an evaluator with the payment functions, compile the authority policy, and
build an intent from all mandate content belonging to the signing request:

```java
import java.util.List;
import org.exploit.verdict.PolicyEvaluator;
import org.exploit.verdict.intent.ap2.Ap2Intent;
import org.exploit.verdict.intent.payment.PaymentRequest;
import org.exploit.verdict.intent.payment.config.PaymentIntentConfig;
import org.exploit.verdict.intent.payment.function.PaymentFunctions;
import org.exploit.verdict.intent.payment.model.MandateInput;

var functions = new PaymentFunctions();
var evaluator = PolicyEvaluator.builder().library(functions, functions).build();
var config = PaymentIntentConfig.fromMap(authority.config());
var policy = evaluator.compileStrict(authority.policy(), PaymentRequest.policySchema());
var request = Ap2Intent.request(List.of(
        MandateInput.checkout(checkoutMandateJson, decodedCheckoutJson),
        MandateInput.of(paymentMandateJson)), config);
var result = evaluator.evaluate(policy, request);
```

`authority` is a loaded [authority document](../authority/README.md).
`checkoutMandateJson` and `paymentMandateJson` are complete mandate JSON objects;
`decodedCheckoutJson` is the decoded checkout content associated with the checkout
mandate. The adapter establishes that association and handles all cryptography.
For Mastercard VI, use `McIntent.request` from
`org.exploit.verdict.intent.mcintent` with the same input and config types.

The default mode expects checkout/payment pairs. Include every mandate being signed;
Verdict groups pairs by protocol references and rejects orphaned or duplicate members.
For an open pair, use `MandateInput.openCheckout(openCheckoutJson, checkoutDisclosureHash)`
plus `MandateInput.of(openPaymentJson)`. The adapter supplies the actual disclosure
hash; Verdict compares the supplied references without computing or verifying hashes.

For a payment-only signing operation, select the mode explicitly:

```java
var request = Ap2Intent.request(
        List.of(MandateInput.of(paymentMandateJson)),
        PaymentRequestMode.PAYMENTS, config);
```

Import `org.exploit.verdict.intent.payment.constant.PaymentRequestMode`.
`CHECKOUTS` similarly accepts closed checkout mandates only. Standalone open mandates
are not supported by these modes.

Reuse the evaluator across authorities and compile once per authority revision.
Pass trusted additional data through `request.withContext(context)`; it is exposed
under the `context` root. Evaluate the returned request through the same API.
For multiple actions, rule matches and approval sources include an `action[i]/`
prefix. `request.actions().get(i).mandateIndices()` identifies their original inputs.
Retain `request.inputs()` and sign exactly the evaluated content after satisfying
any [approval requirements](../approvals.md).

## Where to go next

- [Core API](../core-policy-engine.md): compilation, evaluation and custom functions.
- [AP2 input format](ap2.md) or [VI input format](mcintent.md): protocol-specific reader behavior.
- [Troubleshooting](../troubleshooting.md): missing catalog keys, library registration and failed conditions.
