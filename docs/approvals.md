# Require approval for larger purchases

Suppose purchases up to USD 30 may proceed automatically, USD 30.01–100 require a
manager, and anything above USD 100 must be rejected. Use two allow rules with
separate amount ranges, and attach `approvals` to the second rule.

Complete example: [typed-approvals.yaml](authority/examples/typed-approvals.yaml).
It accepts `recipient`, `currency`, and decimal `amount`, as in [your first policy](getting-started.md).

```yaml
schemaVersion: verdict.authority/v1
id: com.acme.office.approvals
type: custom
version: 1.0.0
config:
  fields:
    recipient: {type: string}
    currency: {type: string}
    amount: {type: decimal}
policy:
  id: purchase-with-approval
  fallback: DENY
  variables:
    officeRecipient: office-shop
  approvers:
    manager:
      algorithm: ED25519
      # Demonstration public key. Replace with your manager's public key.
      publicKey64: "11qYAYKxCrfVS/7TyWQHOg7hcvPapiMlrwIaaPcHURo="
  allow:
    - id: small-purchase
      where:
        - "recipient == officeRecipient && currency == 'USD'"
        - "decimal.gt(amount, '0') && decimal.lte(amount, '30.00')"
    - id: reviewed-purchase
      where:
        - "recipient == officeRecipient && currency == 'USD'"
        - "decimal.gt(amount, '30.00') && decimal.lte(amount, '100.00')"
      approvals:
        threshold: 1
        approvers: [manager]
```

| Input to office-shop           | Result                                        |
|--------------------------------|-----------------------------------------------|
| USD 30.00                      | ALLOW                                         |
| USD 30.01                      | ALLOW_WITH_REQUIREMENTS: manager must approve |
| USD 100.00                     | ALLOW_WITH_REQUIREMENTS                       |
| USD 100.01                     | DENY                                          |
| EUR 20.00 or another recipient | DENY                                          |

`threshold: 1` means one of the listed approvers. For two of three, define three
named approvers under `policy.approvers`, list those names in the rule and set
`threshold: 2`. Names must reference distinct configured keys. Replace the example
public key with your real approver's key.

## What the signing integration does

| Verdict                 | Next action                                              |
|-------------------------|----------------------------------------------------------|
| ALLOW                   | Signing may proceed for the evaluated content            |
| DENY                    | Stop; no signature                                       |
| ALLOW_WITH_REQUIREMENTS | Obtain and verify every required approval before signing |
| Exception               | Stop; fix or reject the invalid input/policy             |

Each returned requirement includes the policy ID, source rule, threshold and eligible
approvers. The application collects and verifies their approvals, then signs the action.

If two allow rules match and both require approval, **both requirements survive**.
Requirements accumulate across matching allow rules. A deny rule wins. For payment requests containing several actions,
requirements from all actions survive; multi-action sources look like `action[1]/reviewed-purchase`.
The integration must satisfy every returned requirement.

## Approval as a fallback

Use this only when otherwise-unmatched actions really should be reviewable:

```yaml
fallback: ALLOW_WITH_REQUIREMENTS
fallbackApprovals:
  threshold: 1
  approvers: [manager]
```

This fragment belongs inside `policy`; `manager` must still be defined in
`policy.approvers`. For a narrow allowlist, keep `fallback: DENY` and attach approvals
to explicit allow rules, as above.
