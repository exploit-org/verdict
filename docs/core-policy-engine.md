# Core Policy Engine

Artifact: `org.exploit:verdict`.

## Responsibility

The core module builds, compiles, and evaluates policies over Google CEL.

It does not decode domain objects by itself. Domain-specific modules convert a request into root CEL variables, then the core evaluator applies policy rules.

## What It Can Check

- Any CEL-compatible Java `Map<String, ?>` context.
- Any `Intent` implementation.
- Allow/deny rules with deterministic precedence.
- Policy-level constants through variables.
- Built-in helper functions for decimal, bigint, lists, network, semver, crypto, time, and effects.
- Custom CEL functions registered by the caller.

## Semantics

- Policy id must be non-blank.
- Rule ids must be unique across `allow` and `deny`.
- A rule matches when every `where` expression is `true`.
- A rule does not match when any `unless` expression is `true`.
- Empty `where` and empty `unless` matches unconditionally.
- Policy variables are available as root CEL variables.
- On name collision, policy variables override runtime context variables.
- `DENY` matches override `ALLOW` matches.
- If no rule matches, `policy.fallback()` is returned.
- Result `matches()` includes matched deny rules first, then allow rules.

## Builder API

```java
Policy policy = Policy.allowByDefault("policy-id")
    .allowWhen("public-read", "resource.public")
    .allow("owner", rule -> rule
        .where("subject.id == resource.ownerId")
        .where("action == 'read'")
        .unless("resource.locked"))
    .deny("banned", rule -> rule.where("subject.banned"))
    .build();
```

Raw records are supported:

```java
Policy policy = new Policy(
    "policy-id",
    Verdict.DENY,
    List.of(new Rule("allow-admin", List.of("subject.role == 'admin'"), List.of())),
    List.of()
);
```

## Policy Variables

Policy variables are constants stored on the policy and merged into CEL root variables at evaluation time.

```java
Policy policy = Policy.denyByDefault("access")
    .variable("allowedRoles", List.of("admin", "support"))
    .variable("maxAmount", "100.00")
    .allowWhen("role", "subject.role in allowedRoles")
    .allowWhen("amount", "decimal.lte(request.amount, maxAmount)")
    .build();
```

Rules:

- Variable names must be non-blank.
- Values may be any CEL-compatible Java value.
- `null` values are visible in CEL as `null`.
- Policy variables override runtime context variables with the same name.

## Evaluation Context

Pass context as `Map<String, ?>`.

```java
Map<String, ?> context = Map.of(
    "subject", Map.of(
        "id", "u-1",
        "account", Map.of(
            "plan", "pro",
            "scopes", List.of("admin", "team:core")
        )
    ),
    "resource", Map.of(
        "ownerId", "u-1",
        "metadata", Map.of("region", "eu-west-1")
    ),
    "action", "read"
);
```

Nested map access:

```cel
subject.account.plan == 'pro'
resource['metadata']['region'] == 'eu-west-1'
'plan' in subject.account
```

Missing variables or missing nested keys fail evaluation unless guarded:

```cel
'plan' in subject.account && subject.account.plan == 'pro'
```

## Intents

An intent is a typed context adapter. `Intent.variables()` returns root CEL variables.

```java
PolicyEvaluation result = evaluator.evaluate(policy, intent);
```

Intent modules:

- [X.509 TBSCertificate intent](intents/x509.md)
- [EVM transaction intent](intents/evm.md)
- [Bitcoin transaction intent](intents/bitcoin.md)
- [Typed JSON intent](intents/typed.md)

## Compile With Explicit Types

By default, discovered variables are declared as CEL `dyn`.

Use explicit types when you want compile-time validation:

```java
CompiledPolicy compiled = evaluator.compile(policy, Map.of(
    "age", SimpleType.INT,
    "user", MapType.create(SimpleType.STRING, SimpleType.DYN)
));
```

## Custom Functions

Register custom CEL functions through `PolicyEvaluator.builder()`.

```java
import dev.cel.common.types.SimpleType;
import org.exploit.verdict.cel.VerdictFunction;
```

Unary:

```java
PolicyEvaluator evaluator = PolicyEvaluator.builder()
    .function(VerdictFunction.unary(
        "email.isCorporate",
        SimpleType.BOOL,
        String.class,
        email -> email.endsWith("@corp.test")
    ))
    .build();

Policy policy = Policy.denyByDefault("custom")
    .allowWhen("corp-email", "email.isCorporate(subject.email)")
    .build();
```

Binary:

```java
PolicyEvaluator evaluator = PolicyEvaluator.builder()
    .function(VerdictFunction.binary(
        "network.matches",
        SimpleType.BOOL,
        String.class,
        String.class,
        (ip, cidr) -> cidr.equals("10.0.0.0/24") && ip.startsWith("10.0.0.")
    ))
    .build();
```

Fixed arity:

```java
PolicyEvaluator evaluator = PolicyEvaluator.builder()
    .function(VerdictFunction.fixed(
        "strings.lengthBetween",
        SimpleType.BOOL,
        List.of(SimpleType.DYN, SimpleType.INT, SimpleType.INT),
        List.of(String.class, Long.class, Long.class),
        args -> {
            int length = ((String) args[0]).length();
            return length >= (Long) args[1] && length <= (Long) args[2];
        }
    ))
    .build();
```

CEL `int` arguments arrive as Java `Long`.

## Built-In CEL Functions

See [Built-in CEL functions](cel-functions.md).

## Errors

Compilation errors throw:

```java
org.exploit.verdict.exception.PolicyCompilationException
```

Evaluation errors throw:

```java
org.exploit.verdict.exception.PolicyEvaluationException
```

Intent validation errors throw:

```java
org.exploit.verdict.exception.IntentValidationException
```

Error messages include policy, rule, and expression location.

## Tests

```bash
./gradlew test
```
