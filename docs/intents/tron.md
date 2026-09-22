# TRON transaction intent

Artifact: `org.exploit.verdict:tron`. Intent type: `tron.transaction`.

Verdict accepts an unsigned transaction dump in the JSON form produced by Signet's
`TronUnsignedTransaction.dump()`, or the `TronUnsignedTransaction` object itself.
The Signet codec verifies the transaction ID and the serialized raw data before
Verdict maps its contracts to effects.

```java
var config = TronIntentConfig.fromMap(authority.config());
var intent = TronTransactionIntent.fromSerialized(unsignedTransactionJson, config);
var result = evaluator.evaluate(compiledPolicy, intent);
```

For direct Signet integration, call `TronTransactionIntent.fromTransaction(unsigned, config)`.
An empty config permits native TRX transfers. To permit TRC20 operations, list each
trusted token contract in `config.contracts` with `standard: trc20` and its Base58
TRON address. See the [authority example](../authority/examples/tron-transfer.yaml).
The example addresses illustrate policy wiring; replace them with your token
contract and recipient.

## Policy variables

The root and `transaction` object expose `hash`, `timestamp`, `expiration`,
`feeLimit`, `refBlockBytes`, `refBlockHash`, `memoHex`, and `contracts`. Each
contract has its protocol `type`, `permissionId`, `provider`, `contractName`,
and `owner`. Transfers add `to` and `amount`; TRC20 calls add `contract`,
`data`, `selector`, and `function`. Amounts use integer base units.

`effects` contains one effect per contract:

| Effect | Fields |
| --- | --- |
| `native.transfer` | `asset: trx`, `from`, `to`, `amount` in Sun |
| `trc20.transfer` | `token`, `from`, `to`, `amount` in token base units |
| `trc20.approval` | `token`, `owner`, `spender`, `amount` |
| `trc20.transferFrom` | `token`, `caller`, `from`, `to`, `amount` |

Addresses are canonical Base58 TRON addresses. For a transaction with multiple
contracts, check **every** effect with `effect.onlyTypes` and the relevant count
and amount checks. The [effect guide](effects.md) explains these helpers.

Verdict rejects signed transactions, unknown contract types, unlisted token
contracts, unsupported TRC20 functions, nonzero contract call or token values,
and raw data containing scripts or authorities. Contract execution is not
simulated; token contract behavior remains part of the trusted whitelist.
The transaction does not identify a TRON network, so the integration must select
the intended network when it signs and broadcasts.
