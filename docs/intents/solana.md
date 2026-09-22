# Solana transaction intent

Artifact: `org.exploit.verdict:solana`. Intent type: `solana.transaction`.

Pass the Base64 dump produced by Signet's `SolanaUnsignedTransaction.dump()`, or
the unsigned transaction object itself:

```java
var intent = SolanaTransactionIntent.fromSerialized(unsignedTransactionBase64);
var result = evaluator.evaluate(compiledPolicy, intent);
```

For direct Signet integration, use `SolanaTransactionIntent.fromTransaction(unsigned)`.
The Signet codec validates the wire transaction before Verdict interprets its
instructions. See the [SOL authority example](../authority/examples/solana-transfer.yaml).
Its addresses are illustrative; use your actual payer and recipient.

## Policy variables

The root and `transaction` object expose `version` (`legacy` or `v0`), `feePayer`,
`recentBlockhash`, `requiredSignatures`, `computeUnitLimit`, `computeUnitPrice`,
and `instructions`. The compute unit limit is null when the message does not set
one; the price is zero when not set. Price is in micro-lamports per compute unit.
Each decoded instruction includes `type`, `programId`, `accounts` with signer and
writable flags, and `dataHex`. Context-only fee estimates are excluded because
they are not signed into the transaction message.

`effects` contains one item per supported value-changing instruction:

| Effect | Fields |
| --- | --- |
| `native.transfer` | `asset: sol`, `from`, `to`, `amount` in lamports |
| `spl.transfer` | `program`, `sourceAccount`, `mint`, `destinationAccount`, `authority`, `amount` in token base units, `decimals` |
| `spl.ata.create` | `payer`, `account`, `owner`, `mint` |

An SPL `destinationAccount` is a token account, not necessarily the owner's
wallet address. If the message creates an associated token account, its `owner`
is visible in a separate `spl.ata.create` effect. Account creation may charge
rent, whose actual amount requires network state and is not present in the message.
Check every effect with `effect.onlyTypes` and the relevant count and amount rules.
The integration must check network fees and rent before signing. The message does
not identify a Solana cluster, so the integration must also select the intended one.

Verdict accepts System Program SOL transfers, classic SPL Token Program
`transferChecked`, idempotent associated token account creation, memo, and
compute unit limit/price instructions. It rejects other programs and actions,
including Token-2022, durable nonce operations, and v0 address lookup tables.
These need chain-specific handling before their effects can be trusted.
