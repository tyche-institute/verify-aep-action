# verify-aep-action

A GitHub composite Action that wraps the [EATF offline
verifier](https://github.com/tyche-institute/eatf-verifier) so that
downstream CI pipelines can assert their `.aep` evidence packages
verify cleanly before they merge.

```yaml
- uses: tyche-institute/verify-aep-action@v1
  with:
    path: ./evidence/   # default '.'  — walks recursively for *.aep
```

By default the Action fails the workflow step if any package
fails verification. The verifier makes no network calls and needs
no API keys — verification happens in the runner's filesystem.

## Inputs

| Input            | Default        | Meaning                                                                                                                |
|------------------|----------------|------------------------------------------------------------------------------------------------------------------------|
| `path`           | `.`            | Path to a single `.aep` file or a directory to walk.                                                                   |
| `mode`           | `batch`        | One of `single`, `batch`, `conformance`.                                                                               |
| `tsa-trust-list` | (none)         | Colon-separated list of PEM files for the verifier's advisory issuer-name pin on the RFC 3161 timestamp signer. See below.|
| `json`           | `false`        | If `true`, the verifier emits machine-readable JSON to stdout.                                                         |
| `eatf-ref`       | `v0.7.4`       | Release tag of `tyche-institute/eatf-verifier` to install. Pin to a release tag for reproducibility.                   |
| `node-version`   | `20`           | Node.js version to install (Node 20 or later required).                                                                |

`tsa-trust-list` is passed to the verifier as `--tsa-trust-list`.
The verifier compares the issuer name of the timestamp signer's
certificate with the subject names of the supplied roots. This is
an advisory pin, not RFC 5280 path validation. When the input is
unset, no issuer-name comparison is made.

## Modes

- **`single`** — `path` must be a `.aep` file. Verifies one
  package; step exits non-zero on failure.
- **`batch`** (default) — `path` is a directory walked recursively.
  Every `.aep` found is verified. Step exits non-zero if any
  package fails.
- **`conformance`** — `path` is a test-vectors tree with
  `valid/<vector>/package.aep` and `invalid/<vector>/package.aep`
  subdirectories. Every package under `valid/` must verify and
  every package under `invalid/` must be rejected; step exits
  non-zero on any contract mismatch, and when the tree holds no
  `.aep` vectors.

## Example: assert every package in a release artifact verifies

```yaml
name: Verify evidence

on:
  pull_request:
    paths:
      - "evidence/**.aep"

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: tyche-institute/verify-aep-action@v1
        with:
          path: ./evidence/
```

## Example: assert a single produced package

```yaml
      - uses: tyche-institute/verify-aep-action@v1
        with:
          mode: single
          path: ./build/attestation.aep
```

## Example: run conformance against a custom vector set

```yaml
      - uses: tyche-institute/verify-aep-action@v1
        with:
          mode: conformance
          path: ./my-vectors/
```

## Design guarantees

- **Verify-only by design.** The Action asserts evidence; pair it
  with the `eatf-sign` CLI in the same workflow to produce `.aep`
  packages from CI.
- **Fully local verification.** All verification happens on the
  runner's filesystem; the verifier never contacts `api.eatf.eu`
  or any other service. Only the setup steps use the network:
  they install Node.js, clone `tyche-institute/eatf-verifier`
  from GitHub and install its npm dependencies.
- **Side-effect-free.** The Action reports through the step's
  exit status alone, so comments, status checks, and summaries
  stay fully under your job's own annotation strategy.

## Versioning

Tags use semver. `@v1` is a moving tag that follows the latest
v1.x.y release; pin to `@v1.0.1` (or similar) for byte-identical
builds.

Use v1.0.1 or later. v1.0.0 cloned the verifier from
`tyche-institute/eatf`, a repository that is no longer public, so
its clone step fails. From v1.0.1 the Action installs the verifier
from the public
[`tyche-institute/eatf-verifier`](https://github.com/tyche-institute/eatf-verifier).

The `eatf-ref` input lets you decouple Action version from
verifier version: e.g., use the latest Action but pin to a
specific `eatf-verifier` release tag (`v0.1.0` or later).

## License

Apache License 2.0. Composite Action: ~1 file of YAML; the
heavy lifting is in `tyche-institute/eatf-verifier`.
