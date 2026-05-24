# Effect Semantics

Effects are normalized consequences exposed as CEL maps.

Policies should approve effects before raw request fields. Raw fields explain the request; effects describe what the request does.

## Shape

Every effect has:

```json
{
  "type": "namespace.action"
}
```

Additional fields depend on the effect type:

```json
{
  "type": "erc20.transfer",
  "token": "0x1111111111111111111111111111111111111111",
  "to": "0x2222222222222222222222222222222222222222",
  "amount": "1000000"
}
```

Amounts are exposed as integer base units unless a module documents otherwise.

## Policy Pattern

Use `effect.*` helpers:

```cel
effect.onlyTypes(effects, ['erc20.transfer']) &&
effect.one(effects, 'erc20.transfer') &&
effect.any(effects, 'erc20.transfer', {
    'token': tokenAddress,
    'to': recipientAddress
}) &&
bigint.lte(effect.amount(effects, 'erc20.transfer'), maxAmount)
```

## Fail Closed

Native modules reject requests when a consequence cannot be described as an allowed effect.

Examples:

- EVM unknown contract call.
- EVM whitelisted function with missing effect mapping.
- Bitcoin output script that cannot be classified.
- Bitcoin missing previous transaction or previous output.

Typed intents only produce effects declared in config.

## Module Effect Types

| Module | Effects |
| --- | --- |
| EVM | `native.transfer`, `erc20.transfer`, `erc20.approval`, `erc20.transferFrom`, custom mapped effects |
| Bitcoin-like | `utxo.spend`, `utxo.output`, `utxo.data`, `utxo.fee` |
| Typed JSON | configured effect types |

See also [Built-in CEL functions](../cel-functions.md#effect-helpers).
