# Getting Started

## Install

Use the artifacts you need:

```groovy
implementation 'org.exploit:verdict:0.1.0'
implementation 'org.exploit:verdict-intent-evm:0.1.0'
implementation 'org.exploit:verdict-authority:0.1.0'
```

Verdict requires Java 21+.

## Evaluate A Map

```java
Policy policy = Policy.denyByDefault("access")
    .allow("admin", rule -> rule
        .where("subject.role == 'admin'")
        .unless("subject.suspended"))
    .denyWhen("blocked-country", "request.country in blockedCountries")
    .build();

PolicyEvaluator evaluator = new PolicyEvaluator();
CompiledPolicy compiled = evaluator.compile(policy);

PolicyEvaluation result = evaluator.evaluate(compiled, Map.of(
    "subject", Map.of(
        "role", "admin",
        "suspended", false
    ),
    "request", Map.of("country", "DE"),
    "blockedCountries", List.of("IR", "KP")
));

result.decision(); // ALLOW
```

## Evaluate An Intent

```java
EvmIntentConfig config = EvmIntentConfig.fromMap(authority.config());
EvmTransactionIntent intent = EvmTransactionIntent.fromBase64(serializedTransaction64, config);

PolicyEvaluation result = evaluator.evaluate(compiledAuthority.policy(), intent);
```

## Common Flow

1. Load an authority by id.
2. Build the intent config from `authority.config()`.
3. Decode the request into an intent.
4. Evaluate the authority policy over the intent.
5. Sign or issue only if the verdict is `ALLOW`.

## Effects

Policies should normally approve effects, not raw request fields:

```cel
effect.one(effects, 'erc20.transfer') &&
effect.any(effects, 'erc20.transfer', {
    'to': subject.wallet,
    'amount': '1000'
})
```

See [Built-in CEL functions](cel-functions.md).
