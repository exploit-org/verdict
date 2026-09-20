# Effects: check what the transaction does

An effect is a map describing one consequence of a decoded request. For example,
EVM calldata for `transfer(address,uint256)` becomes:

```json
{
  "type": "erc20.transfer",
  "token": "0x1111111111111111111111111111111111111111",
  "to": "0x2222222222222222222222222222222222222222",
  "amount": "1000000"
}
```

The amount above is displayed as a string; the native module supplies an integer
value. `type` names the consequence. Other fields depend on that type.
The decoder produces effects using the mappings in its trusted config.
Contract execution and simulation are outside this process.

## Check every effect

Suppose `effects` contains a permitted token transfer **and** an unwanted approval.
`effect.any(effects, 'erc20.transfer', {'to': recipientAddress})` is still true.
It checks for one matching transfer. Use `onlyTypes` to reject unwanted effect types.

For exactly one transfer and no other described action, use all these conditions:

```cel
effect.onlyTypes(effects, ['erc20.transfer'])
effect.one(effects, 'erc20.transfer')
effect.all(effects, 'erc20.transfer', {'token': tokenAddress, 'to': recipientAddress})
bigint.gt(effect.amount(effects, 'erc20.transfer'), '0')
bigint.lte(effect.amount(effects, 'erc20.transfer'), maxBaseUnits)
```

These are separate `where` lines in the same rule. The complete
[EVM authority](../authority/examples/evm-transfer.yaml) defines all constants.

## Pick the helper that asks your question

| Need | Helper | Empty/no matching effects |
| --- | --- | --- |
| Some effect of this type exists | `effect.has(effects, type)` | false |
| Exactly one exists | `effect.one(effects, type)` | false |
| No effect has this type | `effect.none(effects, type)` | true |
| No unlisted effect types | `effect.onlyTypes(effects, types)` | true for empty list |
| At least one matches these fields | `effect.any(effects, type, criteria)` | false |
| Every effect of that type matches | `effect.all(effects, type, criteria)` | false if no matching type |
| Integer amount sum for one type | `effect.amount(effects, type)` | zero |

`effect.all` differs from ordinary CEL `list.all`: it requires a matching effect.
Combine it with `onlyTypes` to restrict the other types.
Check recipients, currencies and assets before summing amounts.
The integer sum helpers are unsuitable for custom effects whose amounts are fractional.

## Effects supplied by each module

| Module | Effect types | Units |
| --- | --- | --- |
| EVM | `native.transfer`, `erc20.transfer`, `erc20.approval`, `erc20.transferFrom`, configured custom types | Wei or token base units, depending on type |
| Bitcoin | `utxo.spend`, `utxo.output`, `utxo.data`, `utxo.fee` | Satoshis for amounts |
| Typed JSON | Your configured types | Your schema's chosen units |
| AP2 raw single mandate | `ap2.payment.authorize`, `ap2.checkout.authorize`, `ap2.payment.delegate`, `ap2.checkout.delegate` | Concrete amounts in currency minor units |
| VI raw single mandate | Same suffixes with `mcintent.` prefix | Concrete payment amounts in currency minor units |

AP2/VI request policies use `payment`, `checkout` and `delegation` helpers.
Delegations expose future spending constraints; their limits remain in those constraints. See [payments](payments.md).

## Decoding can fail before a policy runs

Examples include an unknown EVM contract/function or missing effect mapping, an
unsupported Bitcoin script, or a missing previous transaction. The integration stops
on that error. For typed intents, only configured effects are produced; unconfigured
fields are ignored by effect generation.

Full signatures: [effect function reference](../cel-functions.md#effect-helpers).
