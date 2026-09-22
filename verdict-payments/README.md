# verdict-payments

Shared payment configuration and CEL functions for AP2 and Mastercard VI. Both
protocol modules include it as a dependency.

- [Write a payment policy](../docs/intents/payments.md): shops, cards, amounts, multiple purchases and delegations.
- [Integrate with your signer](../docs/intents/payments.md#java-integration): normal `PolicyEvaluator` and `Intent`
  APIs.
- Copy an authority: [AP2](../docs/authority/examples/ap2-payment.yaml)
  or [VI](../docs/authority/examples/mcintent-payment.yaml).
