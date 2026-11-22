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
| `signer-key`     | (none)         | Colon-separated list of PEM files with the RSA public keys allowed to sign the packages. See below.                    |
| `json`           | `false`        | If `true`, the verifier emits machine-readable JSON to stdout.                                                         |
| `eatf-ref`       | `v0.7.4`       | Release tag of `tyche-institute/eatf-verifier` to install. Pin to a release tag for reproducibility.                   |
| `node-version`   | `24`           | Node.js version to install: an LTS line. The verifier needs Node 20 or later.                                          |

`tsa-trust-list` is passed to the verifier as `--tsa-trust-list`.
The verifier compares the issuer name of the timestamp signer's
certificate with the subject names of the supplied roots. This is
an advisory pin, not RFC 5280 path validation. When the input is
unset, no issuer-name comparison is made.

`signer-key` is passed to the verifier as `--signer-key`, once per
file. Each file holds one RSA public key in a single `PUBLIC KEY`
PEM block, as `openssl pkey -pubout` writes it; for a certificate,
take the key with `openssl x509 -in signer.crt -pubkey -noout`. A
package passes only when the signer key embedded in it matches one
of the listed keys; any other package fails with "Signer public key
is not in the caller-supplied trust set". When the input is unset,
each signature is checked against the key embedded in the package
itself: that shows the package is intact, not who signed it. The
input needs `eatf-ref` v0.2.0 or later; v0.1.x refuses the option
and the step exits 2.

## Outputs

| Output      | Meaning                                                                                                 |
|-------------|---------------------------------------------------------------------------------------------------------|
| `passed`    | Number of packages that verified.                                                                       |
| `failed`    | Number of packages that did not verify.                                                                 |
| `exit-code` | Exit code of `eatf-verify`: `0` all verified (or the conformance contract was met), `1` at least one failure or contract mismatch, `2` usage error, unreadable file, or a conformance tree with no vectors. |

The counts come from the verifier's own report, one per package, in
human and JSON output alike. In `conformance` mode `failed` also
counts the expected rejections under `invalid/`: on the reference
vectors of `eatf-verifier` v0.7.4 a met contract reads `passed=5`,
`failed=8`, `exit-code=0`. When the verifier stops with exit code 2,
the counts cover only the packages it checked before it stopped,
usually none.

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

## Example: accept only packages from a known signer

```yaml
      - uses: tyche-institute/verify-aep-action@v1
        with:
          path: ./evidence/
          signer-key: ./keys/release-signer.pem
```

## Example: read the counts

```yaml
      - uses: tyche-institute/verify-aep-action@v1
        id: aep
        continue-on-error: true
        with:
          path: ./evidence/
      - run: echo "verified ${PASSED}, failed ${FAILED}, exit code ${CODE}"
        env:
          PASSED: ${{ steps.aep.outputs.passed }}
          FAILED: ${{ steps.aep.outputs.failed }}
          CODE: ${{ steps.aep.outputs.exit-code }}
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
  exit status and its three outputs, so comments, status checks,
  and summaries stay fully under your job's own annotation strategy.

## Versioning

Tags use semver. `@v1` is a moving tag that follows the latest
v1.x.y release; pin to `@v1.0.1` (or similar) for byte-identical
builds.

Use v1.0.1 or later. v1.0.0 cloned the verifier from
`tyche-institute/eatf`, a repository that is no longer public, so
its clone step fails. From v1.0.1 the Action installs the verifier
from the public
[`tyche-institute/eatf-verifier`](https://github.com/tyche-institute/eatf-verifier).

Releases up to v1.0.1 leave `passed` and `failed` empty and have
no `signer-key` input. GitHub passes over an unknown input with
only a warning, so with those releases packages are verified
without the key pin.

The `eatf-ref` input lets you decouple Action version from
verifier version: e.g., use the latest Action but pin to a
specific `eatf-verifier` release tag (`v0.1.0` or later).

## License

Apache License 2.0. Composite Action: ~1 file of YAML; the
heavy lifting is in `tyche-institute/eatf-verifier`.
