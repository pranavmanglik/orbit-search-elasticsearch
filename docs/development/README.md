# Orbit Elasticsearch: development

The package declares Python `>=3.11,<3.15` support. CI configuration and actual hosted results are
the evidence for which interpreter versions were exercised; a version range in the manifest alone
is not a test result.

## Local checks

Install the adapter, capability, development tools, and official async client in the Orbit workspace:

```bash
python -m pip install -e ../orbit-search
python -m pip install -e '.[dev,elasticsearch]'
pytest
ruff check src tests
ruff format --check src tests
mypy
```

The regular tests use a fake async client and do not need Elasticsearch. The live test is skipped
unless `ORBIT_ELASTICSEARCH_URL` is set. Use a disposable local service for it; remote access needs
the explicit `ORBIT_ELASTICSEARCH_ALLOW_REMOTE_TESTS=1` opt-in. The test creates and deletes one
randomly named index. Review environment values before running it and use credentials with access
limited to that disposable test cluster.

The current local quality matrix passes on CPython 3.11, 3.12, 3.13, and 3.14: 7 fake-client tests
pass on each interpreter, and the opt-in live Elasticsearch test is skipped when no endpoint is
configured. Ruff lint and format checks, strict mypy, and wheel/sdist builds also pass for each
version. These are local checks, not hosted CI evidence.

## Contribution guidance

Add tests for query translation, provider response validation, exception normalization, request
timeouts, cancellation, PIT cleanup, and lifecycle changes. Keep fake-client tests deterministic;
add live checks only for behavior that the SDK or a server version must confirm. When behavior
changes, update the package README and the specific architecture, operations, or security guide that
describes it. Report client/server versions for live checks and keep deployment validation distinct
from local unit-test results.
