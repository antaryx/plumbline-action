# Plumbline Scanner Action

Runs the [Plumbline](https://github.com/antaryx/plumbline) Linux hardening
scanner on a GitHub Actions runner and publishes the findings to the repository's
Security tab as SARIF.

The action downloads a pinned release tarball, verifies it against the release
checksums before extracting anything, runs the binary against the runner host,
and hands `plumbline.sarif` to `github/codeql-action/upload-sarif`. It audits
the **host the job runs on** — its kernel parameters, services, SSH
configuration, accounts and file permissions — not the repository checkout. That
makes it useful for auditing self-hosted runners and custom runner images; on a
stock GitHub-hosted runner it reports on that ephemeral VM.

## Example usage

`.github/workflows/security.yml`:

```yaml
name: Security

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: "0 6 * * 1"

jobs:
  plumbline:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write   # required to upload SARIF
    steps:
      - uses: antaryx/plumbline-action@v1
        with:
          version: v2.0.1
          sudo: "true"         # without this, root-owned state reports UNKNOWN
          args: scan --format=sarif --output=plumbline.sarif --fail-on-critical 1
```

No `actions/checkout` step is needed: the scanner reads the host, not the
working tree.

## Inputs

| Input | Default | Description |
|---|---|---|
| `version` | `v2.0.1` | Plumbline release to download. Give the release tag (`v2.0.1`); a bare `2.0.1` is accepted and normalised. Pin it — `latest` is not resolved. |
| `args` | `scan --format=sarif --output=plumbline.sarif --fail-on-critical 1` | Arguments passed to the `plumbline` binary. Keep `--format=sarif --output=plumbline.sarif` if you want the upload step to find a report. |
| `sudo` | `"false"` | Run Plumbline with `sudo` for full access to protected system paths. See [Running as root](#running-as-root). |
| `upload-sarif` | `"true"` | Upload `plumbline.sarif` to the Security tab. Set to `"false"` to run the scan as a plain build gate. |

All inputs are strings. GitHub Actions has no boolean input type for composite
actions, so `sudo` and `upload-sarif` are compared against the literal `"true"`;
any other value is treated as false.

## Release verification

The tarball is verified before it is unpacked:

1. `plumbline_<version>_linux_amd64.tar.gz` and `checksums.txt` are downloaded
   from the same release into a temporary directory outside the workspace.
2. `sha256sum --ignore-missing -c checksums.txt` runs there. `--ignore-missing`
   is needed because `checksums.txt` covers every artifact in the release and
   only one was fetched; `sha256sum` still exits non-zero both on a mismatch and
   when it verified nothing at all, so a `checksums.txt` that does not name the
   archive fails the step rather than passing silently.
3. Only then is the archive extracted, and only the binary is moved into the
   workspace.

This catches a corrupt, truncated or substituted download. It does **not**
establish who produced the release: `checksums.txt` arrives over the same
channel as the tarball, so an attacker able to replace one could replace both.
Every Plumbline release also publishes a keyless cosign signature over
`checksums.txt` (`checksums.txt.sig`, `checksums.txt.pem`) and an SBOM per
artifact. If provenance is in your threat model, use the verified install
sequence in
[`docs/CI-INTEGRATION.md`](https://github.com/antaryx/plumbline/blob/main/docs/CI-INTEGRATION.md),
which runs `cosign verify-blob` before `sha256sum`.

## Running as root

Plumbline reads root-owned state — `/etc/shadow`, auditd rules, much of the
service and container surface. Run unprivileged, those checks report `UNKNOWN`
rather than guessing, and coverage falls accordingly. A hardening report from an
unprivileged scan is not a clean bill of health; it is a partial one.

Set `sudo: "true"` to run the scanner as root. The action then also hands the
report back to the runner user: Plumbline writes `--output` owner-only (`0600`),
and re-applies that mode after opening, so a root-owned report would otherwise
be unreadable to the upload step, which runs unprivileged.

This relies on passwordless `sudo -E`, which GitHub-hosted runners provide. A
self-hosted runner with a restricted sudoers policy may refuse `-E`
("sorry, you are not allowed to preserve the environment"); grant the `SETENV`
tag, or leave `sudo` off and accept the reduced coverage.

## Gating the build

Plumbline's exit codes are a contract, and the action propagates them, so the
step fails when a gate trips.

| Code | Meaning |
|---|---|
| 0 | Completed; all gates satisfied |
| 1 | Usage error — nothing was scanned |
| 2 | Findings at or above `--fail-on`, `--fail-on-critical` or `--fail-on-high` |
| 3 | Posture below `--threshold` |
| 4 | Completed degraded — a collector failed, or coverage below `--min-coverage` |
| 10 / 11 / 70 | Insufficient privileges under `--strict-privileges` / timeout / internal error |

`--fail-on-critical` and `--fail-on-high` take a **count**, not a bare flag:
`--fail-on-critical 1` fails on the first CRITICAL, `0` disables the gate.

Gate on coverage as well as on findings. A run that could not read half the
host and reports no failures is not a clean host:

```yaml
args: scan --format=sarif --output=plumbline.sarif --fail-on high --min-coverage 90
```

The SARIF upload runs even when the scan exits non-zero — a run that fails the
build is the one whose findings most need to reach the Security tab. It is
skipped only when no report was written, so a usage error is reported as itself
rather than as a missing file.

## Limitations

- **`linux/amd64` only.** Releases also carry `linux/arm64`; this action does
  not select it.
- **No signature verification.** Checksums are verified, provenance is not. See
  [Release verification](#release-verification).

## Licence

Apache-2.0, matching Plumbline itself.
