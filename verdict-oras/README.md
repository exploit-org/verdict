# verdict-oras

Artifact: `org.exploit.verdict:oras`. Requires Java 25+.

Read and publish OCI artifacts and list repository tags.

```groovy
implementation 'org.exploit.verdict:oras:0.3.0'
```

## Read an artifact

```java
import org.exploit.verdict.oras.OrasClient;

try (var client = OrasClient.builder("https://registry.example")
        .basicAuth(username, password)
        .build()) {
    var manifest = client.pullManifest("team/policies", tagOrSha256Digest);
    var layer = manifest.layers().getFirst();
    byte[] payload = client.pullBlob("team/policies", layer);
    var page = client.tags("team/policies", 100, null);
    // Pass page.nextCursor() to read the following page when it is non-null.
}
```

The manifest digest is computed from the exact response bytes. Digest references
must match it even when `Docker-Content-Digest` is absent. Blob downloads must match
both the descriptor's SHA-256 digest and size. Optional digest response headers are
also checked. `OciManifest.bytes()` preserves the original manifest bytes.

## Publish an artifact

Upload the config and layer blobs first, then publish the application's OCI manifest:

```java
client.pushBlob(repository, configBytes);
client.pushBlob(repository, sourceBytes);
String digest = client.pushManifest(repository, tag, manifestBytes);
```

`pushBlob` returns the blob's SHA-256 digest and skips existing content.
`pushManifest` accepts an OCI manifest as JSON bytes and a tag or matching digest
as the destination. It transmits those exact bytes and returns their SHA-256 digest.
Publishing to an existing tag can replace its previous value.

## Authentication, downloads and limits

Without `basicAuth`, requests are anonymous. Bearer challenges are supported for
anonymous and authenticated requests. A token realm on the registry's origin is
allowed; add a different origin with `.tokenOrigin("https://auth.example")`.
Credentials are only sent to the configured registry and explicitly trusted token
origins. Token redirects are rejected.

Blob redirects are followed at most three times, only to the registry origin or an
origin added with `.downloadOrigin("https://storage.example")`. Redirected requests
receive no registry credentials, and signed URL queries are preserved byte for byte.
Manifest, upload and token requests do not follow redirects. Upload locations must
stay on the same origin and within the repository's upload path.

HTTPS is required. To use a local plain HTTP registry, configure its `http://` URL
and call `.allowHttp()`. TLS certificate verification remains enabled for HTTPS.

The default timeout is 30 seconds per HTTP exchange; the response and upload body
limit is 2 MiB. Override them with `.timeoutSeconds(...)` and `.maxResponseBytes(...)`.
Jettyx enforces the response limit as bytes arrive, including chunked responses, and
applies the deadline to the complete exchange. Close the client when finished;
closing it cancels outstanding exchanges.

Supported artifacts use SHA-256 blobs and OCI image manifests with raw layers.
Image indexes, archives, subject/referrer workflows and descriptor `urls` are not
supported. JSON must contain one document, with no duplicate keys and strictly typed
descriptors and annotations.
`OrasException.status()` returns the HTTP status, or `-1` when no HTTP status is
available.

Protocol references: [OCI Distribution](https://github.com/opencontainers/distribution-spec/blob/main/spec.md),
[OCI manifest format](https://github.com/opencontainers/image-spec/blob/main/manifest.md),
[token authentication](https://distribution.github.io/distribution/spec/auth/token/).

## Tests

```bash
./gradlew :verdict-oras:test :verdict-authority-oci:test
```
