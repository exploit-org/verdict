# EVM Transaction Intent

Artifact: `org.exploit:verdict-intent-evm`.

Intent type: `evm.transaction`.

## Purpose

`EvmTransactionIntent` decodes unsigned serialized EVM transactions and exposes transaction fields, decoded calls, and effects.

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

`chainId` comes from trusted config. Typed transactions must match it. Legacy unsigned transactions do not encode chain id.

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
