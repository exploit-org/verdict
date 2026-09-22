# Bitcoin: pay one recipient and return change

Use this module for an unsigned Bitcoin transaction plus its **full previous
transactions**. Previous transactions let Verdict resolve every spent output and
calculate the fee. Supply the complete previous transactions for every input.

Goal: pay exactly 100,000 satoshis (0.001 BTC), permit at most one change output to
a specified address, and cap the fee at 1,000 satoshis.

## What the policy sees

Relevant fields from an illustrative decoded transaction:

```json
{
  "network": "MAINNET",
  "signing": {"single": true},
  "sighash": {"all": true, "anyoneCanPay": false},
  "outputs": [
    {"address": "RECIPIENT", "amount": "100000"},
    {"address": "CHANGE", "amount": "49000"}
  ],
  "fee": "1000"
}
```

`RECIPIENT` and `CHANGE` abbreviate the configured addresses. The actual intent also
includes every input and effect; amounts are `BigInteger` satoshis, shown as strings
here. The module produces this view from transaction bytes.

## Complete authority

```yaml
schemaVersion: verdict.authority/v1
id: com.acme.bitcoin.transfer
type: bitcoin.transaction
version: 1.0.0
config:
  network: MAINNET
policy:
  id: bitcoin-payment-with-change
  fallback: DENY
  variables:
    # Example addresses. Replace both with addresses you control or intend to pay.
    recipientAddress: "1BoatSLRHtKNngkdXEeobR76b53LETtpyT"
    changeAddress: "1KFHE7w8BhaENAswwryaoccDb6qcT6DbYY"
    recipientSatoshis: "100000"
    maxFeeSatoshis: "1000"
  allow:
    - id: approved-payment
      where:
        - "network == 'MAINNET'"
        - "signing.single && sighash.all && !sighash.anyoneCanPay"
        - "effect.onlyTypes(effects, ['utxo.spend', 'utxo.output', 'utxo.fee'])"
        - "size(outputs) >= 1 && size(outputs) <= 2"
        - "outputs.filter(o, o.address == recipientAddress).size() == 1"
        - "outputs.all(o, (o.address == recipientAddress && bigint.eq(o.amount, recipientSatoshis)) || o.address == changeAddress)"
        - "bigint.lte(fee, maxFeeSatoshis)"
```

Download [bitcoin-transfer.yaml](../authority/examples/bitcoin-transfer.yaml).
Replace both example addresses before using it. `changeAddress` must be an address
your signing integration has verified belongs to the intended wallet.

## Why every output is checked

`outputs.filter(...).size() == 1` requires exactly one recipient output.
`outputs.all(...)` makes every output either that exact payment or change to the
configured address. The size bound permits one or two outputs. Together they stop
an extra output to an attacker.

| Request                                                       | Result                 |
|---------------------------------------------------------------|------------------------|
| 100,000 to recipient, change to configured address, fee 1,000 | ALLOW                  |
| 100,000 to recipient, no change, fee 900                      | ALLOW                  |
| Correct payment plus output to another address                | DENY                   |
| Recipient gets 100,001                                        | DENY                   |
| Fee 1,001                                                     | DENY                   |
| OP_RETURN data output                                         | DENY under this policy |
| Missing previous transaction                                  | Input validation error |

The policy checks the complete transaction even when `signing.single` selects one
input to sign. Add `inputs`/`prevout` conditions to restrict which wallet's inputs can
be spent. The fee limit is an absolute amount in satoshis.

## API, input requirements and field reference

Artifact: `org.exploit:verdict-intent-bitcoin`.

Intent type: `bitcoin.transaction`.

## Purpose

`BitcoinSigningIntent` decodes unsigned Bitcoin transactions with Signet. Java 25+ is required.

The request must include full previous transactions.

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

## Input requirements

- Set `network` explicitly in authority config; unknown config keys are rejected.
- Java callers can use `BitcoinIntentConfig.mainnet()` or select another network with
  `new BitcoinIntentConfig(BitcoinNetwork.REGTEST)`.
- Transaction hex must be even-length without separators, optionally prefixed by `0x` or `0X`.
- Script-specific fields are present only for the matching script type. Use a type
  check or CEL `has()` before accessing optional script fields.
- OP_RETURN accepts data pushes only; malformed pushes and executable opcodes after
  OP_RETURN are rejected.

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

## Supported scripts and validation

The module describes P2PKH, P2SH, P2WPKH, P2WSH, P2TR, P2PK, bare multisig,
and OP_RETURN scripts.

All inputs are resolved even when only a subset is selected for signing. Duplicate
outpoints, coinbase spends, nonempty scriptSig/witness, missing or unspendable
previous outputs, and negative fees are rejected. Only SIGHASH_ALL without extra
bits is accepted. Taproot SIGHASH_DEFAULT is unsupported. The caller checks
spendability and the current on-chain status of each output.
