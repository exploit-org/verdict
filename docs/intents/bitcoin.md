# Bitcoin Transaction Intent

Artifact: `org.exploit:verdict-intent-bitcoin`.

Intent type: `bitcoin.transaction`.

## Purpose

`BitcoinSigningIntent` decodes unsigned Bitcoin transactions with Signet. Java 25+ is required.

The request must include full previous transactions. Raw prevout summaries are not accepted.

## Authority Config

This module supports Bitcoin only. Select the address network explicitly in authority config:

```yaml
type: bitcoin.transaction
config:
  network: MAINNET
```

Supported networks are `MAINNET`, `TESTNET`, `SIGNET`, and `REGTEST` from Signet's
`BitcoinNetwork`. The Java builder defaults to mainnet. Serialized transactions
contain no network identifier; the configured network controls address rendering.

## Breaking changes

- `BitcoinProtocol` and its BTC/LTC/DASH/BCH/custom factories have been removed.
- Replace `protocol: BTC` with `network: MAINNET`; unknown and extra config keys are rejected.
- Replace `BitcoinIntentConfig.btc()` with `BitcoinIntentConfig.mainnet()`.
  For another Bitcoin network, use `new BitcoinIntentConfig(BitcoinNetwork.REGTEST)`.
- Transaction hex must be even-length without separators, optionally prefixed by `0x` or `0X`.
- Script-specific fields are present only for the matching script type. Use a type
  check or CEL `has()` before accessing optional script fields.
- OP_RETURN accepts data pushes only; malformed pushes and executable opcodes after
  OP_RETURN are rejected instead of silently omitted from the described data.

## Java

```java
BitcoinIntentConfig config = BitcoinIntentConfig.fromMap(authority.config());

BitcoinSigningIntent intent = BitcoinSigningIntent.builder()
    .config(config)
    .unsignedTransactionBase64(unsignedTx64)
    .previousTransactionsBase64(previousTx64s)
    .signingInput(0)
    .sighash(BitcoinSighash.all())
    .build();
```

## Effects

- `utxo.spend`
- `utxo.output`
- `utxo.data`
- `utxo.fee`

## Root Fields

- `protocol`, `asset`, `assetDecimals` (always `BTC`, `BTC`, `8`)
- `network`
- `txId`, `wtxId`, `version`, `lockTime`
- `sighash`
- `signing`
- `inputs`
- `outputs`
- `previousTransactions`
- `totalInput`, `totalOutput`, `fee`
- `effects`

## Policy Example

```cel
network == 'MAINNET' &&
signing.single &&
sighash.all &&
!sighash.anyoneCanPay &&
effect.any(effects, 'utxo.output', {
    'address': recipientAddress,
    'amount': '100000'
}) &&
bigint.lte(effect.amount(effects, 'utxo.fee'), maxFee)
```

## Rejections

- `invalid_unsigned_transaction`
- `invalid_previous_transaction`
- `duplicate_previous_transaction`
- `transaction_not_unsigned`
- `coinbase_input`
- `missing_previous_transaction`
- `missing_previous_output`
- `unknown_previous_output_script`
- `unspendable_previous_output`
- `unknown_output_script`
- `negative_fee`
- `unknown_sighash`
- `unsafe_sighash`
- `invalid_signing_input`
- `invalid_bitcoin_intent_config`

## Validation and structure

`BitcoinSigningIntent` is the public request builder. `BitcoinTransactionReader`
handles encoding and Signet parsing, `BitcoinIntentResolver` validates and resolves
previous outputs, and `BitcoinIntentMapper` produces immutable CEL variables and effects.
`ScriptClassifier` describes P2PKH, P2SH, P2WPKH, P2WSH, P2TR, P2PK, bare multisig,
and OP_RETURN scripts using Signet address/hash utilities.

All inputs are resolved even when only a subset is selected for signing. Duplicate
outpoints, coinbase spends, nonempty scriptSig/witness, missing or unspendable
previous outputs, and negative fees are rejected. Only SIGHASH_ALL without extra
bits is accepted; Taproot SIGHASH_DEFAULT is not currently supported. Describing a
script does not prove it can be spent or that its output is still unspent on-chain.
