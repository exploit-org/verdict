# EVM Transaction Intent

Artifact: `org.exploit:verdict-intent-evm`.

Intent type: `evm.transaction`.

## Purpose

`EvmTransactionIntent` decodes unsigned serialized EVM transactions and exposes transaction fields, decoded calls, and effects.

Parsing uses Signet and requires Java 25. Supported transaction types are legacy (0), EIP-2930 (1), and EIP-1559 (2). Signed transactions and types 3/4 are rejected. ABI calls use the ABI codec available through Signet.

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

`chainId` may come from the transaction or trusted config. An encoded chain ID must match the configured value and fit a positive signed 64-bit integer. Legacy transactions without an encoded chain ID use the configured value, or `null` when absent.

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

`type` is the numeric type encoded as a string: `"0"`, `"1"`, or `"2"`. `gasPrice` is `null` for type 2; dynamic fee fields are `null` for types 0/1.

Addresses in config use the `0x` prefix and Signet validation (including checksum validation for mixed-case addresses). Exposed addresses are lowercase. Effect paths must resolve to existing fields.

## Policy Example

```cel
chainId == 1 &&
effect.one(effects, 'erc20.transfer') &&
effect.any(effects, 'erc20.transfer', {
    'token': tokenAddress,
    'to': recipientAddress
}) &&
bigint.lte(effect.amount(effects, 'erc20.transfer'), maxAmount)
```

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
