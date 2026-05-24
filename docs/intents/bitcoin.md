# Bitcoin Transaction Intent

Artifact: `org.exploit:verdict-intent-bitcoin`.

Intent type: `bitcoin.transaction`.

## Purpose

`BitcoinSigningIntent` decodes unsigned Bitcoin-like UTXO transactions with BitcoinJ.

The request must include full previous transactions. Raw prevout summaries are not accepted.

## Authority Config

Raw Bitcoin-like transactions do not identify their protocol. Pin it in authority config:

```yaml
type: bitcoin.transaction
config:
  protocol: BTC
```

Known protocols:

- `BTC`
- `LTC`
- `DASH`
- `BCH`

Custom protocol:

```yaml
config:
  protocol:
    id: DOGE
    asset: DOGE
    p2pkhVersion: 30
    p2shVersion: 22
    bech32Hrp: null
    decimals: 8
```

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

- `protocol`, `asset`, `assetDecimals`
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
protocol == 'BTC' &&
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
- `unknown_output_script`
- `negative_fee`
- `unknown_sighash`
- `unsafe_sighash`
- `invalid_signing_input`
- `invalid_bitcoin_intent_config`
