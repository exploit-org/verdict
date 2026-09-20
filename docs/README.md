# Write a Verdict policy

Verdict answers: **may this action be signed or issued?** You describe the rules in
an authority YAML file. Your application supplies the action and handles signing.

**Start with [your first policy](getting-started.md).** It includes a complete YAML
file, a request, and the expected result. No knowledge of CEL or the Java API is assumed.

## Choose the action you want to control

| I want to… | Guide | Ready-to-edit authority |
| --- | --- | --- |
| Validate my own JSON request | [Typed JSON](intents/typed.md) | [Purchase](authority/examples/typed-purchase.yaml) |
| Require human approval above a limit | [Approvals](approvals.md) | [Purchase with approval](authority/examples/typed-approvals.yaml) |
| Control AP2 or Mastercard agent payments | [Payment policies](intents/payments.md) | [AP2](authority/examples/ap2-payment.yaml), [VI](authority/examples/mcintent-payment.yaml) |
| Restrict an EVM token transfer | [EVM](intents/evm.md) | [Token transfer](authority/examples/evm-transfer.yaml) |
| Restrict a Bitcoin payment and its change | [Bitcoin](intents/bitcoin.md) | [Payment with change](authority/examples/bitcoin-transfer.yaml) |
| Check a certificate before issuance | [X.509](intents/x509.md) | [Server certificate](authority/examples/x509-server.yaml) |

## Learn only what you need

- [Policy language](policy-language.md): AND/OR, lists, optional fields, amounts, common mistakes.
- [Function cookbook and reference](cel-functions.md): pick a helper by the question you need to ask.
- [Authority fields](authority/README.md): what goes in `config`, `variables`, and rules.
- [Effects](intents/effects.md): how to check every consequence of a transaction.
- [Troubleshooting](troubleshooting.md): why a rule did not match, compilation errors, missing fields.

## Integrate Verdict into an application

[Core Java API](core-policy-engine.md) · [Intent modules](intents/README.md) ·
[OCI loading](authority/oci.md) · [AP2 adapter](intents/ap2.md) · [VI adapter](intents/mcintent.md).
Java 25+ is required. Use the same Verdict version for core and intent modules.

An **intent** is the request after the relevant module has decoded it. An **effect**
is a described consequence, such as a token transfer. An **authority** contains the
intent configuration and the policy together. The examples show these concepts in use.

Protocol sources: [Google AP2](https://github.com/google-agentic-commerce/AP2) and
[Mastercard Verifiable Intent](https://github.com/agent-intent/verifiable-intent).
