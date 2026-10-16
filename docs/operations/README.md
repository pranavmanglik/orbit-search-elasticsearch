# Orbit Elasticsearch: operations

This package implements `orbit-search` using Elastic's asynchronous Python client. Install the
capability and adapter extra explicitly:

```bash
python -m pip install 'orbit-search' 'orbit-search-elasticsearch[elasticsearch]'
```

The README contains a complete construction and shutdown example. `ElasticsearchSearchStore.connect`
accepts one or more hosts, an optional API key or Basic Auth tuple, an optional CA bundle, a cursor
signing secret, and a request timeout. No credential is supplied by default. For injected clients,
the store applies its timeout to search and point-in-time requests; the caller remains responsible
for configuring TLS and authentication on that client.

## Request and pagination behavior

Each search uses Elasticsearch `multi_match` over the requested fields and exact `term` filters. A
`None` filter value is translated to a missing-field predicate. Callers must provide mappings with
appropriate field types; this adapter does not manage mappings or reinterpret provider relevance.
Partial shard responses and timed-out queries fail as retryable normalized errors rather than
returning an incomplete page.

Pages use a point-in-time view and `search_after`. The PIT keep-alive is one minute. Continuation
cursors contain the PIT identifier and sort values, authenticated with HMAC-SHA256, and bound to the
index and query parameters. All workers that exchange cursors must use the same 32–64 byte secret.
Rotate that secret only when existing cursors can be discarded. Treat cursors as sensitive: do not
log or expose them in analytics, and expect them to expire with the PIT. Elasticsearch can rotate
the PIT identifier in a search response; the adapter tracks the newest identifier for continuation
and closes that latest snapshot on the final page or an error. PIT close failures are ignored because
Elasticsearch expires snapshots server-side; close-point-in-time calls and searches are bounded by
`request_timeout`. Before signing a continuation cursor, it validates the provider sort tuple as a
finite score and non-negative shard ordinal; malformed pagination metadata fails the current search.

`aclose()` stops new searches, drains active operations, and closes the owned async client. Call it
during application shutdown. Repeated calls after successful shutdown are safe. If client shutdown
fails, a normalized retryable error is raised so cleanup can be retried.

## Live integration test

The ordinary test suite uses a fake client. The opt-in live test creates a uniquely named index,
indexes sample documents, checks filtered PIT pagination, and deletes that index. Set
`ORBIT_ELASTICSEARCH_URL` to a disposable service. Remote endpoints require the explicit
`ORBIT_ELASTICSEARCH_ALLOW_REMOTE_TESTS=1` opt-in. `ORBIT_ELASTICSEARCH_API_KEY` and
`ORBIT_ELASTICSEARCH_CA_CERTS` are read by the live test from the process environment; never place
secrets in shell history or committed files.

Before deployment, validate the Elastic client/server version pair, least-privilege index access,
TLS trust, query mappings, timeout behavior, secret rotation, and PIT pressure under realistic
concurrency. Local fake-client checks do not certify these deployment properties.
