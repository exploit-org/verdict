# XRP transaction intent

Artifact: `org.exploit.verdict:xrp`. Intent type: `xrp.transaction`.

Pass the hex dump produced by Signet's `XrpUnsignedTransaction.dump()`, or the
unsigned transaction object itself:

```java
var intent = XrpTransactionIntent.fromSerialized(unsignedTransactionHex);
var result = evaluator.evaluate(compiledPolicy, intent);
```

For direct Signet integration, use `XrpTransactionIntent.fromTransaction(unsigned)`.
Signet validates the transaction format and supported XRP operations. Verdict
accepts direct XRP Payments and atomic payment Batches containing 2–8 Payments.
It does not describe issued-currency payments or other XRP transaction types.

## Policy variables

The root and `transaction` object expose `type` (`payment` or `batch`), `account`,
`fee` in drops, `sequence`, `lastLedgerSequence`, `sourceTag`, `flags`, and
`signingPublicKey`. Optional fields are null when absent.

For a direct Payment, `destination`, `amount` in drops, and `destinationTag` are
also available at the root. For a Batch, those three fields are null and
`payments` contains each inner payment's `account`, `destination`, `amount`,
`fee`, `sequence`, `lastLedgerSequence`, `destinationTag`, `sourceTag`, and `flags`.
For a direct Payment, `payments` is empty.

`effects` contains one `native.transfer` for each payment. Each effect has
`asset: xrp`, `from`, `to`, `amount` in drops, and `destinationTag` (nullable).
For a Batch, **every** inner payment has its own effect. Use `effect.onlyTypes`,
`effect.all`, and `effect.amount` to check all destinations and the total amount.
Destination tags can distinguish recipients at a shared address, so check them
when the recipient requires one. The [authority example](../authority/examples/xrp-payment.yaml)
allows one tagged payment to a specified address.

`fee` is the outer transaction fee for a Batch; inner payments have zero fee.
Bound the fee in policy. `lastLedgerSequence` may be absent, so require it in a
policy if the application needs a bounded submission window. The integration
selects the XRP network and checks account state and current fee requirements
before signing; those facts are not encoded in the transaction.
