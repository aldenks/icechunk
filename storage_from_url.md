# `icechunk.storage_from_url` (proposal branch)

> **Status:** unmerged. Lives on branch `storage-from-url` in
> `/home/alden/code/dynamical/icechunk`. It is **not** in any released
> icechunk. To use it you must build this branch as an editable install
> (instructions below).

## What it does

Builds a **read-only** `icechunk.Storage` directly from a single URL, picking
the backend from the URL scheme and parsing the bucket / prefix / options out
of the URL. This is the same scheme-resolution logic that backs
`redirect_storage`, exposed as a standalone entry point so callers that already
know the final location (e.g. a URL pulled from a STAC asset) can skip the HTTP
redirect round-trip.

```python
import icechunk
import xarray as xr

storage = icechunk.storage_from_url(
    "s3://icechunk-public-data/v1/era5_weatherbench2?region=us-east-1"
)
repo = icechunk.Repository.open(storage)
session = repo.readonly_session("main")
ds = xr.open_zarr(session.store, consolidated=False)
```

## API

```python
def storage_from_url(url: str) -> icechunk.Storage: ...
```

There is also a classmethod form: `icechunk.Storage.from_url(url)`. The
top-level `storage_from_url` is the intended public entry point.

- **Parameters** — `url: str`: a fully-qualified storage URL pointing at a
  supported scheme.
- **Returns** — `icechunk.Storage` configured for **anonymous** access.
- **Raises** — `IcechunkError` on an unknown scheme, an unparseable URL, or a
  missing required query parameter.

### Supported schemes

| Scheme | Required URL parts | Example |
| --- | --- | --- |
| `s3://` | host = bucket, path = prefix, **`?region=`** | `s3://my-bucket/my/prefix?region=us-east-1` |
| `gs://` / `gcs://` | host = bucket, path = prefix | `gs://my-bucket/my/prefix` |
| `r2://` | host = bucket, path = prefix, **`?account_id=`** (optional `?region=`) | `r2://my-bucket/my/prefix?account_id=abc123` |
| `tigris://` | host = bucket, path = prefix (optional `?region=`) | `tigris://my-bucket/my/prefix` |
| `http+icechunk://`, `https+icechunk://`, `http+ic://`, `https+ic://` | the `+icechunk`/`+ic` suffix is stripped and the remaining `http(s)://…` URL is used as the store base | `https+icechunk://example.com/repo` |

## Limitations (all inherited from the existing redirect logic)

- **Anonymous access only.** There is currently no `credentials=` parameter; it
  always configures the backend for unsigned/public access.
- **Required query params per scheme.** `s3://` needs `?region=…`; `r2://` needs
  `?account_id=…`. Omitting them raises an error rather than guessing.
- **Error wording.** Some error messages read *"Bad URL for redirect
  Storage…"* because they are shared with the redirect code path, even though
  no redirect is involved here.

## Installing this branch as an editable install in a sibling repo

icechunk's Python package is a maturin/PyO3 project (compiled Rust extension),
so an editable install has to **build the extension**.

1. Make sure this repo is on the right branch:

   ```bash
   git -C /home/alden/code/dynamical/icechunk rev-parse --abbrev-ref HEAD
   # should print: storage-from-url    (if not: git -C <repo> checkout storage-from-url)
   ```

2. Activate the **sibling repo's** virtualenv (the env where you want icechunk
   available), then build + install icechunk into it with maturin:

   ```bash
   source /path/to/sibling-repo/.venv/bin/activate   # the target env

   pip install maturin   # if not already present in that env
   cd /home/alden/code/dynamical/icechunk/icechunk-python
   maturin develop --uv   # builds the Rust extension and installs into the active env
   ```

   - The first build compiles the Rust crate and takes ~3 minutes; later
     rebuilds are faster.
   - `maturin develop` installs into whatever environment is currently active,
     so activating the sibling venv first is the important step.
   - If the env uses `uv`, the equivalent is
     `uv run --active maturin develop --uv` from the `icechunk-python` dir.

   Alternatively, a one-shot PEP 660 editable install:

   ```bash
   uv pip install -e /home/alden/code/dynamical/icechunk/icechunk-python
   # or: pip install -e /home/alden/code/dynamical/icechunk/icechunk-python
   ```

3. Verify it's the right build:

   ```bash
   python -c "import icechunk; print(hasattr(icechunk, 'storage_from_url'))"
   # True
   ```

> **Note on rebuilds:** because the extension is compiled, editing the Rust
> source requires re-running `maturin develop` (or the editable install) to take
> effect. Python-only edits under `icechunk-python/python/` are picked up
> without a rebuild.
