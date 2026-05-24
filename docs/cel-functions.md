# Built-In CEL Functions

These functions are installed by the core `org.exploit:verdict` module.

## Effect Helpers

Inputs: list of maps. Every effect must have `type`.

Use them with intent modules that expose `effects`, for example EVM and Bitcoin.

```cel
effect.any(effects, 'erc20.transfer', {'to': subject.wallet, 'amount': '1000'})
effect.onlyTypes(effects, ['erc20.transfer', 'native.transfer'])
bigint.lte(effect.amount(effects, 'erc20.transfer'), maxTokenAmount)
```

- `effect.types(effects)`: list of effect type strings, in order.
- `effect.has(effects, type)`: at least one effect has `type`.
- `effect.none(effects, type)`: no effect has `type`.
- `effect.one(effects, type)`: exactly one effect has `type`.
- `effect.count(effects, type)`: number of effects with `type`.
- `effect.of(effects, type)`: effects with `type`.
- `effect.onlyTypes(effects, allowedTypes)`: every effect type is in `allowedTypes`.
- `effect.any(effects, type, criteria)`: at least one effect with `type` matches every field in `criteria`.
- `effect.all(effects, type, criteria)`: at least one effect has `type`, and all effects with `type` match `criteria`.
- `effect.values(effects, type, field)`: field values from effects with `type`; missing fields are skipped.
- `effect.amount(effects, type)`: sum of `amount` fields for effects with `type`.
- `effect.sum(effects, type, field)`: sum of a numeric field for effects with `type`.

`effect.amount` and `effect.sum` return `BigInteger`. Missing summed fields on a matching effect fail evaluation.

## Standard CEL

Standard CEL macros are enabled:

```cel
roles.exists(role, role == 'admin')
roles.all(role, role != 'banned')
has(resource.ownerId)
```

Google CEL list extension is enabled:

```cel
[1, 2, 3].reverse()[0] == 3
[1, 2, 3, 4].slice(1, 3) == [2, 3]
```

## Decimal Helpers

Inputs: `BigDecimal`, `BigInteger`, finite Java numbers, CEL `int`/`double`, and decimal strings.

```cel
decimal.eq(decimal.add(balance, fee), '0.30')
decimal.between(amount, '10.00', '100.00')
decimal.round(amount, 2)
```

- `decimal.from(value)`: converts `value` to `BigDecimal`.
- `decimal.compare(left, right)`: returns `-1`, `0`, or `1`.
- `decimal.eq(left, right)`: `left == right`, ignoring scale (`1.0 == 1.00`).
- `decimal.ne(left, right)`: `left != right`.
- `decimal.gt(left, right)`: `left > right`.
- `decimal.gte(left, right)`: `left >= right`.
- `decimal.lt(left, right)`: `left < right`.
- `decimal.lte(left, right)`: `left <= right`.
- `decimal.between(value, min, max)`: inclusive range check.
- `decimal.add(left, right)`: decimal addition.
- `decimal.sub(left, right)`: decimal subtraction.
- `decimal.mul(left, right)`: decimal multiplication.
- `decimal.div(left, right)`: decimal division with `DECIMAL128`; division by zero fails evaluation.
- `decimal.round(value, scale)`: sets decimal scale with `HALF_UP`.
- `decimal.abs(value)`: absolute value.

## BigInt Helpers

Inputs: `BigInteger`, exact integral `BigDecimal`, integral Java numbers, CEL `int`, and integer strings.

```cel
bigint.gt(counter, '9223372036854775808')
bigint.eq(bigint.mod(counter, 10), 9)
```

- `bigint.from(value)`: converts `value` to `BigInteger`; fractional values fail evaluation.
- `bigint.compare(left, right)`: returns `-1`, `0`, or `1`.
- `bigint.eq(left, right)`: `left == right`.
- `bigint.ne(left, right)`: `left != right`.
- `bigint.gt(left, right)`: `left > right`.
- `bigint.gte(left, right)`: `left >= right`.
- `bigint.lt(left, right)`: `left < right`.
- `bigint.lte(left, right)`: `left <= right`.
- `bigint.between(value, min, max)`: inclusive range check.
- `bigint.add(left, right)`: integer addition.
- `bigint.sub(left, right)`: integer subtraction.
- `bigint.mul(left, right)`: integer multiplication.
- `bigint.mod(left, right)`: remainder; modulo by zero fails evaluation.
- `bigint.abs(value)`: absolute value.

## List Helpers

Inputs: Java `Collection`, Java arrays, and CEL lists. Membership helpers compare numeric Java/CEL values by value, not by wrapper type.

```cel
lists.containsAny(subject.account.scopes, ['admin', 'billing'])
lists.hasOnly(roles, allowedRoles)
lists.nonEmpty(lists.without(roles, ['viewer']))
```

- `lists.isEmpty(value)`: `true` when the list has no elements.
- `lists.nonEmpty(value)`: `true` when the list has at least one element.
- `lists.size(value)`: element count.
- `lists.first(value)`: first element; empty lists fail evaluation.
- `lists.last(value)`: last element; empty lists fail evaluation.
- `lists.contains(values, candidate)`: `true` when any element equals `candidate`.
- `lists.containsAny(values, candidates)`: `true` when at least one candidate exists in `values`.
- `lists.containsAll(values, candidates)`: `true` when every candidate exists in `values`.
- `lists.containsNone(values, candidates)`: `true` when no candidate exists in `values`.
- `lists.hasOnly(values, allowed)`: `true` when every value is present in `allowed`.
- `lists.concat(left, right)`: returns `left + right`.
- `lists.without(values, excluded)`: returns values not present in `excluded`.
- `lists.distinct(values)`: keeps first occurrence of each value.

## Network Helpers

`ip.*` inputs: IP strings, `InetAddress`, and raw `byte[]`. Invalid IP values return `false`.

```cel
ip.isPrivate(request.ip)
cidr.matches(request.ip, '10.0.0.0/8')
cidr.matchesAny(request.ip, allowedCidrs)
```

- `ip.isValid(value)`: `true` when `value` is IPv4 or IPv6.
- `ip.isV4(value)`: `true` for IPv4.
- `ip.isV6(value)`: `true` for IPv6.
- `ip.isPrivate(value)`: `true` for RFC1918 IPv4 and unique-local IPv6.
- `ip.isPublic(value)`: `true` for public addresses; false for invalid, private, loopback, link-local, reserved, multicast, and documentation ranges.
- `ip.isLoopback(value)`: `true` for loopback addresses.
- `ip.isLinkLocal(value)`: `true` for IPv4 `169.254.0.0/16` and IPv6 `fe80::/10`.
- `cidr.matches(ip, cidr)`: `true` when `ip` is inside `cidr`; invalid inputs return `false`.
- `cidr.matchesAny(ip, cidrs)`: `true` when `ip` is inside any CIDR. `cidrs` may be a collection, array, single CIDR string, or comma-separated string.
- `cidr.contains(cidr, ipOrCidr)`: `true` when the first CIDR contains an IP or another CIDR.

## Semver Helpers

Supports SemVer precedence, prerelease ordering, build metadata ignore, optional `v` prefix, and missing minor/patch as `0`.

```cel
semver.gte(app.version, '1.4.0')
semver.lt(app.version, '2.0.0-beta')
semver.between(app.version, '1.4.0', '1.5.0')
```

- `semver.isValid(value)`: `true` when `value` is accepted as a SemVer string.
- `semver.compare(left, right)`: returns `-1`, `0`, or `1`.
- `semver.eq(left, right)`: `left == right`.
- `semver.ne(left, right)`: `left != right`.
- `semver.gt(left, right)`: `left > right`.
- `semver.gte(left, right)`: `left >= right`.
- `semver.lt(left, right)`: `left < right`.
- `semver.lte(left, right)`: `left <= right`.
- `semver.between(value, min, max)`: inclusive range check.

## Crypto Helpers

Byte inputs: `byte[]`, `ByteBuffer`, `UUID`, Java collections, and arrays of byte values. String inputs are UTF-8 unless the function documents normalization.

```cel
crypto.sha256(subject.email) == expectedHash
crypto.digest(payload, 'sha-512') == expectedDigest
crypto.uuidEq(request.id, expectedRequestId)
```

- `crypto.digest(value, algorithm)`: hex digest. Algorithm names are normalized (`sha_256`, `sha-256`, `SHA-256`).
- `crypto.sha256(value)`: SHA-256 hex digest.
- `crypto.sha512(value)`: SHA-512 hex digest.
- `crypto.md5(value)`: MD5 hex digest.
- `crypto.hex(value)`: hex-encodes bytes. If a string is already valid hex with spaces, colons, dashes, or `0x`, returns normalized lowercase hex.
- `crypto.isHex(value)`: `true` when a string is valid even-length hex after normalization.
- `crypto.uuid(value)`: canonical lowercase UUID string; accepts `UUID`, UUID strings, `{uuid}`, and `urn:uuid:...`.
- `crypto.isUuid(value)`: `true` when `value` can be parsed as UUID.
- `crypto.uuidEq(left, right)`: compares UUID values after normalization; invalid values return `false`.

## Time Helpers

Inputs: `Instant`, `Date`, `Calendar`, `OffsetDateTime`, `ZonedDateTime`, `LocalDateTime` as UTC, `LocalDate` as UTC start of day, ISO/RFC-1123 strings, epoch seconds, and epoch millis.

```cel
time.after(time.now(), subject.createdAt)
time.before(request.createdAt, request.expiresAt)
time.ageMinutes(subject.createdAt, time.now()) < 30
```

- `time.now()`: current UTC `Instant`.
- `time.parse(value)`: converts supported input to `Instant`.
- `time.before(left, right)`: `left < right`.
- `time.after(left, right)`: `left > right`.
- `time.between(value, start, end)`: inclusive instant range check.
- `time.durationSeconds(start, end)`: whole seconds from `start` to `end`; may be negative.
- `time.ageSeconds(timestamp, now)`: whole seconds from `timestamp` to `now`.
- `time.ageMinutes(timestamp, now)`: whole minutes from `timestamp` to `now`.
- `time.ageHours(timestamp, now)`: whole hours from `timestamp` to `now`.
- `time.ageDays(timestamp, now)`: whole days from `timestamp` to `now`.

Numeric epochs with absolute value below `10_000_000_000` are seconds; larger numeric epochs are millis.
