# uv for CircleCI

[CircleCI Orb](https://circleci.com/docs/orb-intro) commands to install a pinned
[uv](https://docs.astral.sh/uv/) release and to install uv projects from their
`uv.lock`.

The orb carries the SHA-256 of the uv versions it vouches for and defaults to
one of them (0.10.2), so the version every pipeline runs by default changes in
one reviewed place: a release of the orb. A pipeline can ask for another
version; it then gives the checksum itself, and the orb refuses a release
published less than 7 days before, the same window the dependencies get.

## Commands

| Command | What it does |
|---|---|
| `install` | Downloads uv for the job's architecture and libc (x86_64 or aarch64, glibc or musl), checks it against its SHA-256, installs it into `~/.cobli-uv/bin` and puts it first on the `PATH` of the following steps. Fails if the `uv` those steps would run is not that binary, at that version. |
| `sync` | Runs `install`, restores the uv cache keyed on `uv.lock`, runs `uv sync --locked` in `app-dir` and saves the cache. |

`sync` always passes `--locked`: it refuses a `uv.lock` that no longer matches
`pyproject.toml` instead of resolving again, and uv checks every artifact
against the hash in the lock. Extra arguments go in `args`, for example
`--no-default-groups --group test` or `--no-dev`.

Both commands set `UV_PYTHON_DOWNLOADS=never` by default: uv uses the job
image's Python and fails if it does not satisfy the project, instead of
downloading one. Set `python-downloads` to change it.

## Parameters of `install` (and `sync`)

| Parameter | Default | |
|---|---|---|
| `version` | `0.10.2` | uv version |
| `sha256` | empty | SHA-256 of `uv-<target>.tar.gz` for the job's platform, from the release's `.sha256` file. Required for a version the orb does not carry; for one it carries, it must match |
| `min-release-age-days` | `7` | For a version the orb does not carry: refuse a release published less than this many days before, measured by the `Last-Modified` of its archive. `0` skips the check |
| `python-downloads` | `never` | `UV_PYTHON_DOWNLOADS` for the following steps |

A version the orb carries uses the orb's checksum and skips the age check: it
was checked when it entered the orb. Using another version:

```yaml
- uv/sync:
    version: 0.12.18
    # https://github.com/astral-sh/uv/releases/download/0.12.18/uv-x86_64-unknown-linux-gnu.tar.gz.sha256
    sha256: 89eadd7c76fc063887959510d5ba0ab1264dfd5f1143b925ddb73021a40acf16
```

The checksum is for one platform: a job that runs on aarch64 or on Alpine
(musl) needs the checksum of that archive.

## Why not the uv already in the image

CI images can ship a uv of their own. `cimg/python` installs the latest uv at
the time the image was built, into `~/.local/bin`, ahead of `/usr/local/bin` on
the `PATH`: on `cimg/python:3.11`, a uv installed and checksummed by the job
into `/usr/local/bin` was not the one that ran. `circleci/python`'s
`install-packages` with `pkg-manager: uv` runs whatever uv the image has. This
orb installs its own binary, checks that it is the one on the `PATH`, and
`sync` calls it by its full path.

## Usage

See the [examples](src/orb.yaml).

```yaml
version: 2.1
orbs:
  uv: cobli/uv@x.x.x
jobs:
  test:
    docker:
      - image: cimg/python:3.11
    steps:
      - checkout
      - uv/sync:
          app-dir: ./helm-charts
      - run:
          working_directory: ./helm-charts
          command: .venv/bin/pytest
```

On images without bash (Alpine), later steps do not source `$BASH_ENV`, so
`uv` is not on their `PATH`; call `~/.cobli-uv/bin/uv` there. `sync` already
does.

## Adding a uv version to the orb, or changing the default

The window that applies to dependencies applies to the tool that resolves them:
add a uv release only once it was published at least 7 days before.

```bash
v=0.10.2
gh api "repos/astral-sh/uv/releases/tags/$v" --jq .published_at
for t in x86_64-unknown-linux-gnu aarch64-unknown-linux-gnu \
         x86_64-unknown-linux-musl aarch64-unknown-linux-musl; do
  echo "$t $(curl -fsSL "https://github.com/astral-sh/uv/releases/download/$v/uv-$t.tar.gz.sha256" | cut -d' ' -f1)"
done
```

Add the four `version:target` lines to the checksum table of the `install`
step in `src/orb.yaml`; to change the default, also change the `version`
default of `install` and `sync` (and the `0.10.2` in this README). Then:

```bash
circleci orb validate src/orb.yaml
circleci orb publish src/orb.yaml cobli/uv@X.Y.Z
```

Changing the default uv version is a minor release of the orb; only adding a
version to the table is a patch release.

## Limits

- Linux only, on x86_64 and aarch64.
- `uv sync` still builds sdists in an isolated environment it resolves at that
  moment: those build dependencies are not in `uv.lock` and have no hash. Pin
  them with `build-constraint-dependencies` in the project when an old sdist
  needs it.
