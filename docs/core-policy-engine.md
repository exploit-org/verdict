# Core engine: integrate a policy evaluator

Artifact: `org.exploit.verdict:core`. Java 25+.

If you are writing YAML, start with [your first policy](getting-started.md).
This page explains the Java calls your application makes and their exact behavior.

## The three calls you need

```java
var evaluator = new PolicyEvaluator();                 // application lifetime
var compiled = evaluator.compileStrict(policy, schema); // once per policy revision
var result = evaluator.evaluate(compiled, intent);      // once per request
```

`policy` comes from a builder or `authority.policy()`. `schema` declares external
root variable names and CEL types. `intent` is built by the matching domain module
with trusted `authority.config()`. No signing takes place in these calls.

| Result                  | Application behavior                                          |
|-------------------------|---------------------------------------------------------------|
| ALLOW                   | May proceed with the evaluated action                         |
| DENY                    | Stop                                                          |
| ALLOW_WITH_REQUIREMENTS | Satisfy every returned approval requirement before proceeding |
| Exception               | Stop and handle the invalid policy/input/evaluation           |

Handle all three verdicts explicitly. See [approvals](approvals.md) for a complete example.

## A complete map example

For application-defined data you can evaluate a Java map directly:

```java
import java.util.Map;
import dev.cel.common.types.SimpleType;
import org.exploit.verdict.PolicyEvaluator;
import org.exploit.verdict.model.Policy;

var policy = Policy.denyByDefault("read-access")
        .allow("active-admin", rule -> rule
                .where("role == 'admin'")
                .unless("suspended"))
        .build();
var evaluator = new PolicyEvaluator();
var compiled = evaluator.compileStrict(policy, Map.of(
        "role", SimpleType.STRING,
        "suspended", SimpleType.BOOL));
var result = evaluator.evaluate(compiled, Map.of("role", "admin", "suspended", false));
// ALLOW; matches contains active-admin.
```

Change `suspended` to true or `role` to `viewer`: DENY. Omit a required root from the
map: evaluation error. The caller supplies every required field.
For protocol-specific signing formats, use the matching native intent module to
decode the request.

## How one action is decided

1. A rule matches when every `where` is true and no `unless` is true.
2. A matching deny rule produces DENY, even when allow rules also match.
3. Otherwise, matching allow rules permit the action and collect their approvals.
4. With no matching allow or deny, use `fallback` and its `fallbackApprovals`, if any.

Empty `where`/`unless` lists mean an unconditional match. Rule IDs must be nonempty
and unique across allow and deny. The policy ID and fallback are required.
Matching deny rules appear before matching allow rules in `matches()`.
The evaluator may encounter errors while evaluating other rules even if a deny
has already matched. Guard optional fields within each rule that reads them.

Separate allow rules use OR. Conditions within one `where` use AND. See
[AND/OR examples](policy-language.md) before splitting a policy into rules.

## Constants versus input

```java
var policy = Policy.denyByDefault("roles")
        .variable("allowedRoles", java.util.List.of("admin", "support"))
        .allowWhen("allowed-role", "role in allowedRoles")
        .build();
```

Policy variables are declared automatically. Names cannot be blank. Null values
are visible as CEL `null`. With explicit external declarations, collisions with
policy variable names fail compilation. At runtime, policy variables override
same-named context entries. Keep constants and request roots distinct.

## compile versus compileStrict

| Call                            | Behavior                                                        |
|---------------------------------|-----------------------------------------------------------------|
| `compile(policy)`               | Discovers external roots and declares them as dynamic           |
| `compile(policy, schema)`       | Discovers roots and uses supplied types where available         |
| `compileStrict(policy, schema)` | Only roots in the schema and policy constants may be referenced |
| `compileStrict(policy)`         | Only policy constants may be referenced                         |

Use strict compilation when the intent builder defines the input schema. A dynamic
map root still has runtime-defined keys: `merchants.officeSop` can compile while
failing at evaluation because the actual key is `officeShop`. Nested map keys and
input values are checked during evaluation.

## Intents with one or several actions

An ordinary `Intent` implements `variables()`. Its default `evaluationInputs()`
returns a list containing that map. Single-action results contain unprefixed rule IDs
and approval sources.

Payment requests implement the same interface but expose one input map per action.
`evaluate(compiled, intent)` evaluates every input. Any DENY denies the whole request;
otherwise every approval requirement survives. An empty input list is invalid.
An evaluation error aborts the operation.
For multiple inputs, matched IDs and approval sources carry `action[i]/` prefixes.

Always pass a multi-action intent itself to `evaluate`. `PaymentRequest.variables()`
throws for multiple actions to prevent accidental evaluation of only one of them.
Map evaluation represents one action. Multi-action evaluation uses the intent's
`evaluationInputs()` method.

## Register functions once, then reuse the evaluator

Core helpers are enabled by default. The builder supports custom functions and CEL
libraries. For [payment helpers](intents/payments.md):

```java
var payments = new PaymentFunctions();
var evaluator = PolicyEvaluator.builder()
        .standardLibraries(true)
        .library(payments, payments)
        .build();
```

The first library argument installs compiler declarations; the second installs runtime
implementations. `PaymentFunctions` supplies both. It holds no current authority,
config or request state, so this evaluator can serve different authorities.
Compile policies with `compileStrict` and evaluate requests with `evaluate`.

For a small custom predicate:

```java
var evaluator = PolicyEvaluator.builder()
        .function(VerdictFunction.unary(
                "email.isCorporate", SimpleType.BOOL, String.class,
                email -> email.endsWith("@corp.test")))
        .build();
// In CEL: email.isCorporate(subject.email)
```

Import `org.exploit.verdict.cel.VerdictFunction`. Binary and fixed-arity factories
are also available; CEL `int` arguments arrive as Java `Long`.
Pass current request data through the evaluation input. Keep shared functions stateless.

## Diagnose failures

- `IntentValidationException`: input could not be decoded/validated by its module.
- `PolicyCompilationException`: invalid policy structure or CEL expression.
- `PolicyEvaluationException`: a CEL expression could not be evaluated.

Configuration and argument factories may also throw `IllegalArgumentException` for
invalid values. Handle construction errors before proceeding to signing.
See [troubleshooting](troubleshooting.md) and the [function reference](cel-functions.md).
