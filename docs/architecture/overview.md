# Orbit Elasticsearch: architecture and boundaries

## Responsibility

`orbit-search-elasticsearch` is the optional Elasticsearch provider for the provider-neutral
`orbit-search` capability. Install both explicitly:

```bash
python -m pip install 'orbit-search' 'orbit-search-elasticsearch[elasticsearch]'
```

```python
from orbit_search_elasticsearch import ElasticsearchSearchStore
from orbit_search import SearchQuery

store = ElasticsearchSearchStore.connect(
    ["https://search.example.internal:9200"],
    api_key=api_key,  # load from a secret manager
    ca_certs="/etc/ssl/certs/search-ca.pem",
    cursor_secret=cursor_secret_from_secret_manager,
)
try:
    page = await store.search("articles", SearchQuery(text="async", fields=("title",)))
finally:
    await store.aclose()
```

The adapter translates the portable text and exact-match filters into Elasticsearch Query DSL. It
uses the official async client, PIT plus `search_after` for consistent bounded pagination, signs
opaque cursors with the configured shared secret, limits pages to 100 hits, rejects partial/time-out results,
sanitizes vendor failures, propagates cancellation, and drains active searches before closing its
owned client. PITs expire after one minute; all workers serving a cursor must share the same high-entropy
secret from a secret manager. Do not log cursors because they contain snapshot identifiers.

TLS verification uses the SDK's secure defaults; configure credentials with API keys or explicit
Basic Auth via a secret manager. There are no default credentials. This adapter does not provision
indexes, ingest documents, manage mappings, or retry searches. Validate server permissions,
version compatibility, and query mappings in the deployment environment before release.

This alpha package supports Python 3.11–3.14. The official client supports async/await and
requires explicit closure; the provider owns these lifecycle details. Current client compatibility
is documented by Elastic: its 8.x Python client supports Elasticsearch 8.x and 9.x (not 10.x).
The package suite passes on Python 3.11–3.14. Its local live integration check passed on Python 3.11
against Elasticsearch 8.17.4 with `elasticsearch` 8.19.3; this does not establish production
topology or a full server/client compatibility matrix.

## Declared dependencies

The following dependency declarations come from the checked-in manifests. Optional groups and development dependencies are called out separately.

### `pyproject.toml`
- `orbit-search>=0.1.0a1,<0.2`
- Optional `elasticsearch` group: `elasticsearch[async]>=8.17,<9`.
- Optional `dev` group: `pytest>=8,<10`, `pytest-asyncio>=0.24,<2`, `ruff>=0.8,<1`, `mypy>=1.13,<2`.

Declared dependencies do not mean that optional providers or services are bundled with this package.

## Implementation layout

Representative implementation files in this checkout:

- `src/orbit_search_elasticsearch/__init__.py`
- `src/orbit_search_elasticsearch/store.py`

## Contract mapping and ownership

The adapter implements `SearchStore.search(collection, query)` from `orbit-search`:

1. It validates the collection alias and accepts only a `SearchQuery` value.
2. It builds an Elasticsearch `multi_match` query and exact filters from the capability fields.
3. It opens or resumes a one-minute point-in-time snapshot and requests at most one extra hit to
   determine whether another bounded page exists.
4. It validates every returned ID, score, source document, timing field, and pagination sort value
   before constructing `SearchPage`.
5. It translates expected provider failures into the capability's safe error types and closes PITs
   when pagination is complete or a request fails.

The adapter owns the async SDK client passed to its constructor. `aclose()` prevents new work, drains
active searches, and closes that client. The capability contract itself does not define lifecycle
methods; applications must retain and close the adapter instance during shutdown.

The cursor format is adapter-private. It contains a PIT ID, the two sort values, and a SHA-256 query
fingerprint, authenticated with HMAC-SHA256. It is not encrypted and should be treated as sensitive.

## Boundary rules

Keep provider SDKs, credentials, transport behavior, PIT management, and Elasticsearch error
translation in this adapter. Keep shared query values and errors in `orbit-search`. Do not add
Elasticsearch DSL or SDK types to the capability package.
