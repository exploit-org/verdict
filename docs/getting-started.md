# Your first policy

Suppose your application receives a purchase request. You want to allow a payment
to `office-shop`, in USD, above zero and at most USD 100. Urgent purchases must use
a different process and should be rejected here.

This example uses your own JSON format. For Google AP2 or Mastercard VI, use the
[payments guide](intents/payments.md), which reads their mandate formats.

## 1. Start with the request

```json
{"recipient": "office-shop", "currency": "USD", "amount": "49.99", "urgent": false}
```

This schema defines `amount` in dollars as a decimal. Compare it with `decimal.*` helpers.

## 2. Save this authority as authority.yaml

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

The complete file is also available as [typed-purchase.yaml](authority/examples/typed-purchase.yaml).

Read it in three parts:

1. `config.fields` describes the input. Each field is required unless marked otherwise.
   An omitted `urgent` becomes `false`.
2. `policy.variables` contains your constants: the recipient and the limit.
3. `policy.allow` describes a permitted action. **Every line in its `where` must pass.**
   The separate `deny` rule rejects an urgent request even if the allow rule matched.

`fallback: DENY` requires a matching allow rule. The recipient check belongs in that rule.

## 3. Check a few requests mentally

Keep the recipient `office-shop` and currency USD unless the row says otherwise.

| Request | Result | Why |
| --- | --- | --- |
| 49.99, not urgent | ALLOW | All conditions pass |
| 100.00, not urgent | ALLOW | The upper bound is inclusive |
| 100.01 | DENY | Over the limit |
| 0 | DENY | Must be greater than zero |
| 49.99 EUR | DENY | Wrong currency |
| 49.99 to another recipient | DENY | Wrong recipient |
| 49.99, urgent | DENY | A deny rule wins |
| Missing `amount` | Input error | Required input is missing |

Input, compilation and evaluation errors stop signing.

## 4. Change the policy for your needs

- Change `maxAmount` to `"250.00"` to raise the limit.
- Add another `where` line to require another condition.
- Add a **separate allow rule** for an alternative permitted case. Separate rules use OR;
  conditions within one `where` use AND.
- To allow two recipients, use `recipient in ['office-shop', 'backup-shop']` in place
  of the existing recipient condition.
- To require approval for some purchases, follow [approvals](approvals.md).

## 5. Run it from Java

Add `org.exploit:verdict-authority` and `org.exploit:verdict-intent-typed` at the same
version as your Verdict core. The intent modules include core as a transitive dependency.

```java
import java.nio.file.Path;
import java.util.Map;
import dev.cel.common.types.SimpleType;
import org.exploit.verdict.PolicyEvaluator;
import org.exploit.verdict.authority.AuthorityParser;
import org.exploit.verdict.intent.typed.TypedIntent;
import org.exploit.verdict.intent.typed.config.TypedIntentConfig;

var authority = new AuthorityParser().parse(Path.of("authority.yaml"));
var config = TypedIntentConfig.from(authority.config());
var evaluator = new PolicyEvaluator();
var policy = evaluator.compileStrict(authority.policy(), Map.of(
        "recipient", SimpleType.STRING,
        "currency", SimpleType.STRING,
        "amount", SimpleType.DYN,
        "urgent", SimpleType.BOOL,
        "effects", SimpleType.DYN));
var intent = TypedIntent.fromJson("""
        {"recipient":"office-shop","currency":"USD","amount":"49.99","urgent":false}
        """, config);
var result = evaluator.evaluate(policy, intent);
// result.decision() == Verdict.ALLOW
```

Compile once per authority revision; evaluate each new request. The result contains
the decision, matched rules, and any required approvals. Your application signs only
after ALLOW, or after satisfying every requirement of ALLOW_WITH_REQUIREMENTS.

Next: [policy language](policy-language.md), or choose your [intent module](intents/README.md).
