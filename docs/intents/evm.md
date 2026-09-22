# EVM: allow one token transfer

The module decodes an **unsigned EVM transaction** and its configured ABI call.
Policies evaluate the transaction fields and the effects defined by the call's mapping.

Goal: on chain 1, transfer at most 1,000,000 base units of one configured token to
one configured recipient. The policy rejects approvals and other effect types.

## What the policy sees

Relevant fields produced by the decoder:

```json
{
  "chainId": 1,
  "effects": [{
    "type": "erc20.transfer",
    "token": "0x1111111111111111111111111111111111111111",
    "to": "0x2222222222222222222222222222222222222222",
    "amount": "500000"
  }]
}
```

Here `amount` is displayed as a string for readability; the module supplies a
`BigInteger`. For a six-decimal token, this represents 0.5 tokens. Set the limit
using the token's known decimals.

## Complete authority

```yaml
schemaVersion: verdict.authority/v1
id: com.acme.evm.transfer
type: evm.transaction
version: 1.0.0
config:
  chainId: 1
  contracts:
    - standard: erc20
      address: "0x1111111111111111111111111111111111111111"
policy:
  id: one-token-transfer
  fallback: DENY
  variables:
    tokenAddress: "0x1111111111111111111111111111111111111111"
    recipientAddress: "0x2222222222222222222222222222222222222222"
    maxBaseUnits: "1000000"
  allow:
    - id: approved-transfer
      where:
        - "chainId == 1"
        - "effect.onlyTypes(effects, ['erc20.transfer'])"
        - "effect.one(effects, 'erc20.transfer')"
        - "effect.all(effects, 'erc20.transfer', {'token': tokenAddress, 'to': recipientAddress})"
        - "bigint.gt(effect.amount(effects, 'erc20.transfer'), '0')"
        - "bigint.lte(effect.amount(effects, 'erc20.transfer'), maxBaseUnits)"
```

Download [evm-transfer.yaml](../authority/examples/evm-transfer.yaml).
Replace the example token and recipient addresses, then set `maxBaseUnits` in the
token's smallest units. Use lowercase addresses in policy constants because the
module exposes lowercase addresses.

## Read the rule

| Check                                 | Reason                                            |
|---------------------------------------|---------------------------------------------------|
| `chainId == 1`                        | Restrict the transaction's chain                  |
| `onlyTypes(..., ['erc20.transfer'])`  | Reject other described actions, such as approvals |
| `one(..., 'erc20.transfer')`          | Require exactly one transfer                      |
| `all(..., {'token': ..., 'to': ...})` | Check its token and recipient                     |
| `bigint.gt(..., '0')`                 | Exclude zero-value transfers                      |
| `bigint.lte(..., maxBaseUnits)`       | Bound its integer amount                          |

The ERC-20 config decodes `transfer`, `approve` and `transferFrom`. This policy
permits only `transfer`. Effect mappings define how decoded calls are represented;
contract simulation is outside the module.

| Request                                      | Result                 |
|----------------------------------------------|------------------------|
| Correct recipient, 500,000 units             | ALLOW                  |
| Correct recipient, 1,000,001 units           | DENY                   |
| Correct amount, different recipient          | DENY                   |
| `approve` on the configured token            | DENY                   |
| Transfer plus an additional described effect | DENY                   |
| Unknown contract/function                    | Input validation error |

This example restricts token movement. Add fee conditions for your fee policy, for
example `bigint.lte(gasLimit, '100000')` in the same `where`. Fee fields depend on the
transaction type; guard nullable fields or constrain `type` before using them.

## API, custom contracts and available fields

Artifact: `org.exploit:verdict-intent-evm`.

Intent type: `evm.transaction`.

## Purpose

`EvmTransactionIntent` decodes unsigned serialized EVM transactions and exposes transaction fields, decoded calls, and
effects.

Parsing uses Signet and requires Java 25. Supported transaction types are legacy (0), EIP-2930 (1), and EIP-1559 (2).
Signed transactions and types 3/4 are rejected. ABI calls use the ABI codec available through Signet.

Contract calls are whitelist-only. The target contract, function selector, and effect mapping must be configured.

## Authority Config

```yaml
type: evm.transaction
config:
  chainId: 1
  contracts:
    - standard: erc20
      address: "0x1111111111111111111111111111111111111111"
```

Custom contract:

```yaml
config:
  chainId: 1
  contracts:
    - name: vault
      address: "0x4444444444444444444444444444444444444444"
      functions:
        - signature: "withdraw(address,uint256)"
          arguments: ["to", "amount"]
          effects:
            - type: vault.withdraw
              fields:
                vault: "$transaction.to"
                to: "$to"
                amount: "$amount"
```

`chainId` may come from the transaction or trusted config. An encoded chain ID must match the configured value and fit a
positive signed 64-bit integer. Legacy transactions without an encoded chain ID use the configured value, or `null` when
absent.

## Java

```java
EvmIntentConfig config = EvmIntentConfig.fromMap(authority.config());
EvmTransactionIntent intent = EvmTransactionIntent.fromBase64(serializedTransaction64, config);
```

## Built-In Effects

- `native.transfer`
- `erc20.transfer`
- `erc20.approval`
- `erc20.transferFrom`

## Root Fields

- `type`, `chainId`, `nonce`
- `gasPrice`, `gasLimit`, `maxPriorityFeePerGas`, `maxFeePerGas`
- `to`, `value`, `data`, `selector`
- `transaction`
- `call`
- `effects`

`type` is the numeric type encoded as a string: `"0"`, `"1"`, or `"2"`. `gasPrice` is `null` for type 2; dynamic fee
fields are `null` for types 0/1.

Addresses in config use the `0x` prefix and Signet validation (including checksum validation for mixed-case addresses).
Exposed addresses are lowercase. Effect paths must resolve to existing fields.

## Rejections

- `invalid_transaction`
- `transaction_not_unsigned`
- `chain_id_mismatch`
- `chain_id_out_of_range`
- `unsupported_transaction_type`
- `contract_creation_not_whitelisted`
- `empty_call_to_whitelisted_contract`
- `contract_not_whitelisted`
- `missing_function_selector`
- `function_not_whitelisted`
- `calldata_decode_failed`
- `effect_mapping_failed`
- `effect_not_described`
- `payable_value_not_described`
- `invalid_evm_intent_config`
