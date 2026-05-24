# OCI Authority Loading

Artifact: `org.exploit:verdict-authority-oci`.

Authorities can be loaded from digest-pinned OCI artifacts.

## Reference Format

Use a digest-pinned reference:

```text
oci://registry.acme.io/verdict/evm-treasury@sha256:...
```

Tags are not accepted as trust anchors.

## Java

```java
OciArtifactClient client = OrasOciArtifactClient.defaults();

OciAuthorityRegistry registry = OciAuthorityRegistry.builder(client)
    .authority(
        "com.acme.prod.evm.treasury",
        "oci://registry.acme.io/verdict/evm-treasury@sha256:..."
    )
    .build();

LoadedAuthority loaded = registry.load("com.acme.prod.evm.treasury");
CompiledAuthority compiled = registry.compile("com.acme.prod.evm.treasury");
```

## Artifact Payload

The OCI artifact should contain one authority document:

- `authority.json`
- `authority.yaml`
- `authority.yml`

The descriptor digest must match the requested digest.
