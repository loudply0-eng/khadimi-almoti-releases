# Signed stable release procedure

This repository contains the public Windows build and stable-release workflows. The canonical application source remains in the private repository loudply0-eng/khadimi-almoti.

## Required GitHub Actions secrets

- **KHADIMI_SOURCE_READ_KEY**: the existing read-only deploy key used to clone the private application repository.
- **KHADIMI_RELEASE_SIGNING_KEY**: an RSA private key in PEM format, at least 3072 bits. It is used only by the isolated signing job and must never be committed, added to an artifact, or printed in logs.

The build job does not receive the release signing key. It builds the application and uploads only the ZIP package. A fresh Windows runner then downloads that ZIP, computes its SHA-256 and exact byte size, creates the signed manifest, and verifies its own signature before it can optionally publish a release. The GitHub write token is scoped only to the release-publishing step.

## Generate and store the signing key from the Oracle server

Run the following in the Oracle Linux SSH terminal after confirming that gh auth status is logged into the account that owns this repository. The private key is created under /dev/shm (RAM-backed temporary storage), piped directly into the GitHub Actions secret, and removed on exit. Only the public key is printed.

    set -euo pipefail
    command -v openssl >/dev/null
    command -v gh >/dev/null
    test -d /dev/shm && test -w /dev/shm
    gh auth status >/dev/null

    umask 077
    KEY_TMP="$(mktemp -d /dev/shm/khadimi-signing.XXXXXX)"
    cleanup() { rm -rf -- "$KEY_TMP"; }
    trap cleanup EXIT

    openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 -out "$KEY_TMP/private.pem" >/dev/null 2>&1
    openssl pkey -in "$KEY_TMP/private.pem" -pubout -out "$KEY_TMP/public.pem"

    gh secret set KHADIMI_RELEASE_SIGNING_KEY \
      --repo loudply0-eng/khadimi-almoti-releases \
      < "$KEY_TMP/private.pem"

    echo "GitHub Actions signing secret configured. Public key (safe to share):"
    cat "$KEY_TMP/public.pem"

Copy only the displayed public PEM into the private application source. Never paste, upload, or commit private.pem.

## Preview before any public release

Use **Actions → Build and Sign Stable Release → Run workflow**:

- **release_version**: a three-part numeric version, for example 1.0.1.
- **minimum_supported_version**: normally 1.0.0 for the first release.
- **publish_release**: keep **false** for the first run.

A preview run still builds the Windows app, runs the updater security smoke tests, signs latest.json, and verifies the RSA-PSS signature. It does not create a public release. The resulting artifact contains the package, checksum, and signed manifest.

Only set publish_release=true after the production public key has been embedded in the application and the app's feed-check, download, helper, backup, rollback, and data-preservation paths have passed their tests. Publishing attaches these three assets to a GitHub Release tagged v<release_version>:

- Khadimi-Windows-win-x64.zip
- SHA256SUMS.txt
- latest.json

The application can later retrieve the latest stable manifest from:

https://github.com/loudply0-eng/khadimi-almoti-releases/releases/latest/download/latest.json

## Safety boundaries

- Stable releases are manual; the workflow never publishes a public release automatically on a source push.
- The signing private key is never available to the build job.
- The workflow packages only the published application output; it does not clone or include local user data, runtime folders, voice profiles, or AI models.
- A successful signed preview is not proof that the installed application can safely update itself. The update helper and rollback implementation remain separate release gates.
