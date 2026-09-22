# Why did my policy fail?

First distinguish three stages: the module reads the request, Verdict compiles the
policy, then Verdict evaluates it. An error at any stage stops signing.

| Symptom                                   | Likely cause                                                       | What to change                                                                                         |
|-------------------------------------------|--------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Unexpected ALLOW                          | Checks were split across separate allow rules                      | Put checks that must all pass in one `where`                                                           |
| Empty rule permits everything             | Empty `where` matches unconditionally                              | Remove the placeholder rule or add conditions                                                          |
| DENY, `matches()` empty                   | No allow rule matched; fallback is DENY                            | Compare the input with each condition in one allow rule                                                |
| DENY, allow and deny both appear          | A deny rule matched                                                | Deny takes precedence                                                                                  |
| ALLOW_WITH_REQUIREMENTS                   | A matching allow rule or fallback requires approval                | Inspect every `approvalRequirements()` entry                                                           |
| Unknown root at compile time              | Typo or missing declaration in strict schema                       | Use the module's root names; add trusted custom data through its supported path                        |
| No such key / missing field at evaluation | Nested key missing, wrong shape or typo                            | Use `has(...)`, null checks, or fix the input/config                                                   |
| Unknown function / no matching overload   | Typo, wrong argument type, or missing library                      | Check the function signature; register `PaymentFunctions` for payments                                 |
| Wrong money result                        | Dollars compared with cents, or strings compared directly          | Use the unit table in [policy language](policy-language.md#amounts-choose-the-correct-unit-and-helper) |
| Payment identity helper fails             | The helper received a string alias                                 | Use `payment.methodIs(methods.companyCard)`                                                            |
| `merchants.officeSop` fails               | Config has `officeShop`; nested map keys are checked at runtime    | Correct the catalog key                                                                                |
| Payment total check is false              | Mixed currencies, delegation, checkout-only request, or over limit | Check the action kind and currency; unpriced actions fail the total check                              |
| Module rejects request before CEL         | Malformed/unsupported content or missing signing context           | Consult that module's input requirements                                                               |

## Inspect results in Java

```java
var result = evaluator.evaluate(compiledPolicy, intent);
System.out.println(result.decision());
System.out.println(result.matches());
System.out.println(result.approvalRequirements());
```

`matches()` lists the rules that matched. To diagnose a failed condition, evaluate it
in a local test with the same intent variables and library setup. For a multi-action
payment request, inspect `request.evaluationInputs().get(index)`. `variables()` throws
when the request contains several actions.

## Decide whether a missing field should deny or be invalid

If the field is optional and its absence should deny, guard it:

```cel
has(customer.country) && customer.country == 'DE'
```

If the field is required for every request, declare it as required in the typed
config or use the native module's required field.

## Test a boundary before publishing the authority

For each amount limit, check exactly the limit and one smallest unit above it.
For each allowlist, test an allowed value, an unlisted value, and an empty list.
For batch operations, test one valid action plus one forbidden action. The whole
request must be denied. The repository's examples are exercised by Gradle tests.
