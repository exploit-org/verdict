# Load an authority from an OCI registry

Artifact: `org.exploit.verdict:authority-oci`.

Use OCI loading when your application receives versioned authority files from a
registry. The policy YAML format stays the same. Write and
test it locally first using [the authority guide](README.md).

## Pin the exact artifact

A supported reference has this shape:

```text
oci://registry.acme.io/verdict/office-purchase@sha256:<64 hexadecimal digits>
```

Replace the placeholder with the artifact's actual digest. A tag such as `:latest`
is not accepted: the same tag can point to different content later. Pinning the
digest identifies exact content; your application must still decide which registry,
repository and digest it trusts.

## Load by authority ID

```java
var client = OrasOciArtifactClient.defaults();
var registry = OciAuthorityRegistry.builder(client)
        .authority("com.acme.office.purchase", digestPinnedReference)
        .build();
var loaded = registry.load("com.acme.office.purchase");
var authority = loaded.authority();
```

Here `digestPinnedReference` is a real reference selected by your deployment/config
system. Imports are from
`org.exploit.verdict.authority.oci`.

Now use `authority.config()` to build the matching intent configuration and compile
`authority.policy()` with your evaluator. Select the decoder using `authority.type()`.
Applications using the default
registry compiler can also call `registry.compile(id)`; payment integrations should
use their evaluator with `PaymentFunctions` registered, as shown in the
[payment integration guide](../intents/payments.md#java-integration).

## What the artifact contains

The artifact contains one supported authority document: `authority.json`,
`authority.yaml`, or `authority.yml`. The descriptor digest must match the requested
digest, and the document's `id` must match the registered authority ID.

| Failure                     | Check                                                               |
|-----------------------------|---------------------------------------------------------------------|
| Reference rejected          | Use `oci://`, repository path, and a full SHA-256 digest; omit tags |
| Pull/authentication failure | Registry reachability and client credentials                        |
| Authority ID mismatch       | The YAML `id` and application registration must agree               |
| Document parse failure      | One supported document, valid schema, no duplicate keys             |
| Policy compile failure      | CEL fields/functions and the evaluator's registered libraries       |

The loader retrieves and validates the authority artifact. The application evaluates
requests, collects approvals and signs authorized content.
