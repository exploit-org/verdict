# How to write conditions

Conditions are CEL expressions that return `true` or `false`.
The examples below are fragments to insert into an authority's `policy` section.

## AND: put the checks in one rule

```yaml
allow:
  - id: approved-purchase
    where:
      - "currency == 'USD'"
      - "decimal.lte(amount, '100.00')"
```

Both lines must pass. This is equivalent to one expression:
`currency == 'USD' && decimal.lte(amount, '100.00')`.

## OR: alternatives inside a condition or separate rules

```cel
recipient == 'office-shop' || recipient == 'backup-shop'
```

The shorter version is `recipient in ['office-shop', 'backup-shop']`.

Separate allow rules are also alternatives. This policy fragment is **wrong** if
you intended to require both USD and a limit:

```yaml
allow:
  - id: currency-only
    where: ["currency == 'USD'"]
  - id: amount-only
    where: ["decimal.lte(amount, '100.00')"]
```

It permits a large USD request via the first rule and a small non-USD request via
the second. Put both checks in one rule instead.

## Deny and unless do different jobs

```yaml
allow:
  - id: office-purchase
    where: ["recipient == 'office-shop'"]
    unless: ["urgent"]
deny:
  - id: blocked-currency
    where: ["currency == 'EUR'"]
```

`unless` excludes the request from **this allow rule**; another allow rule can still
permit it. A matching `deny` forbids the action regardless of other allow rules.
Every `where` must be true; any true `unless` prevents that rule from matching.
An empty `where` matches unconditionally. Remove unused placeholder rules.

## Read fields and check optional values

| Need                 | Expression                                             | Meaning                             |
|----------------------|--------------------------------------------------------|-------------------------------------|
| Equality             | `currency == 'USD'`                                    | Exact string match; case matters    |
| Inequality           | `recipient != 'blocked-shop'`                          | Different string                    |
| Boolean              | `urgent` or `!urgent`                                  | True or false                       |
| Nested field         | `customer.country == 'DE'`                             | Field in an object                  |
| Key with punctuation | `labels['cost-center'] == 'office'`                    | Bracket access                      |
| Optional key         | `has(customer.country) && customer.country == 'DE'`    | Missing nested key makes this false |
| Nullable field       | `customer.country != null && customer.country == 'DE'` | Field exists but may be null        |

Declare the root object in the intent schema before using `has()` on its fields.
Typed JSON optional fields without defaults exist as `null`; native modules can omit
optional fields. Each module guide describes its field behavior.

## Lists: one match is different from every item

```cel
items.exists(item, item.sku == 'paper')
items.all(item, item.sku in ['paper', 'pens'])
size(items) > 0 && items.all(item, item.sku in ['paper', 'pens'])
```

The first permits a list containing paper **and anything else**. The second requires
all items to be permitted, but returns true for an empty list. The third also requires
at least one item. `item` is a local name for the element being inspected.

For plain lists, helpers can be easier:

```cel
lists.nonEmpty(roles) && lists.hasOnly(roles, ['reader', 'writer'])
```

`hasOnly` accepts an empty list; combine it with `nonEmpty` to require entries.
AP2/VI [action policies](intents/payments.md) run once per action. Use list checks
for nested collections such as cart items.

## Amounts: choose the correct unit and helper

| Data                          | Example limit  | Comparison                              |
|-------------------------------|----------------|-----------------------------------------|
| Your decimal field in dollars | `"100.00"`     | `decimal.lte(amount, '100.00')`         |
| EVM token base units          | `"1000000"`    | `bigint.lte(tokenAmount, '1000000')`    |
| Bitcoin satoshis              | `"1000"`       | `bigint.lte(fee, '1000')`               |
| AP2/VI concrete payment       | `"100.00"` USD | `payment.amountAtMost('100.00', 'USD')` |

A token with six decimal places has 1,000,000 base units per token. Set limits using
the token's known decimals. Bitcoin has 100,000,000 satoshis per BTC. Payment helpers convert
the quoted major-unit limit to the currency's minor units and check the currency too.

Use quoted constants for exact large integers and money, and `bigint.*` or `decimal.*`
for comparisons. Ordinary string comparisons are alphabetical: `'100' < '20'` is true.

## Where each name comes from

- An input root such as `amount` comes from the intent module, or typed `config.fields`.
- A constant such as `maxAmount` comes from `policy.variables`.
- `merchants` and `methods` in payment policies come from `PaymentIntentConfig`.
- A name such as `item` inside `.all(item, ...)` exists only inside that expression.

Each intent module defines which config entries become CEL variables. A missing
variable is an error. Strict compilation catches unknown roots; nested map keys and
values are checked at evaluation. See [troubleshooting](troubleshooting.md).
