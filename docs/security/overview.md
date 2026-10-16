# Orbit Elasticsearch: security

The adapter uses the official client's TLS verification defaults, accepts credentials explicitly,
signs continuation cursors, bounds request time and page size, and maps provider failures to
credential-free public errors. These controls do not replace deployment authorization, network
isolation, secret management, or review of Elasticsearch mappings and permissions.

## Credentials and transport

- Load API keys or the Basic Auth tuple from a secret manager; do not embed them in source, URLs,
  examples, logs, or error messages.
- Use HTTPS for remote services. Keep certificate verification enabled and provide a trusted CA
  bundle with `ca_certs` when the service uses a private trust root.
- Grant the identity only the index privileges required for search and point-in-time creation and
  closure. The adapter does not create indices or administer cluster security.
- Configure network policy so only intended application workloads can reach the cluster.

## Cursor protection

The cursor contains an Elasticsearch PIT identifier and sort values. HMAC-SHA256 prevents callers
from changing its contents, and the query fingerprint prevents reusing it with different search
parameters. The HMAC does not encrypt the cursor. Do not log it or expose it to untrusted clients
unless the application explicitly accepts that disclosure. Use 32–64 bytes of unpredictable secret
material from a secret manager and share it only among workers that need to continue the same query.
Changing the secret invalidates outstanding cursors. Cursors also expire when their one-minute PIT
expires.

## Data boundaries

Search text and returned source documents may include confidential or personal information. Apply
authorization before calling the adapter, derive tenant filters from trusted identity, and avoid
logging queries, document payloads, and raw provider responses. This package's search contract does
not impose tenant isolation or collection authorization. Elasticsearch mappings and field access
must be configured to match application policy.

The adapter rejects malformed responses, incomplete shard results, and timed-out searches. It does
not retry requests automatically. Its timeout and 100-hit capability page bound each provider call,
but deployment concurrency and total result processing still need application-level limits.

## Vulnerability disclosure

Follow the repository [security policy](../../SECURITY.md) for private vulnerability reporting; do
not publish unpatched security details in public issues.
