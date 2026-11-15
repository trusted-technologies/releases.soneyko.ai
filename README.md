# releases.soneyko.ai

The central source of ready-to-install Soneyko software releases. Source code lives in each project's own repository; this document defines the mandatory contract for building, storing, publishing, and updating releases.

Public files are served over HTTPS at `https://releases.soneyko.ai`. Artifacts are stored in Cloudflare R2. The S3 endpoint is used only for publishing and administrative verification; clients download from the public custom domain. Container images are outside the scope of this contract.

> This README is a specification and configuration example. It does not create a bucket, DNS records, secrets, GitHub Actions, or completed builds. Each application's build script must be implemented in its source repository.

## 1. Mandatory release rules

- Every project has a permanent, unique slug, such as `node-agent`. Use lowercase ASCII letters, digits, and single hyphens; the first and last characters must be letters or digits. Renaming a GitHub repository does not change its project slug.
- **GitHub Release titles must be exactly `Build 1`, `Build 2`, `Build 3`, and so on.** Tags must be `build-1`, `build-2`, `build-3`. Storage paths use the same tag.
- Build numbers are allocated per project, not organization-wide. An additional semantic version may appear in the description and manifest, but it does not replace `Build N`.
- One permanent `release.yml` workflow owns the project's numbering. In the example below, `N = github.run_number`. Failed runs consume a number: gaps are allowed, but reusing a number is prohibited.
- Re-running jobs (`Re-run`) does not create a new Build. Start a new workflow run for a new build. Do not delete/recreate the workflow or change its counter scheme without preserving monotonically increasing numbering.
- Each Build is bound to an exact, full Git commit SHA. Builds from different source commits must never share a number.
- Publish a release only after **the entire required matrix** has been built and verified successfully. A missing architecture or OS version is a release failure, not a silently skipped target.
- All files under `builds/build-N/` are immutable. Never overwrite an existing archive, manifest, or signature. Publish corrections as a new Build.
- Update `stable` only after every file has been uploaded, read back, and verified, and the GitHub Release has been created.
- Never include secrets, `.env` files, private keys, machine configurations, or user data in release artifacts.

## 2. Required distributions and architectures

Architectures are listed **separately for each version**. Do not generate a Cartesian product of every version and every architecture.

- **AlmaLinux**
  - 10: `amd64`, `arm64`.
  - 9: `amd64`, `arm64`.
  - 8: `amd64`, `arm64`.
- **Amazon Linux**
  - 2023: `amd64`, `arm64`.
  - 2: `amd64`, `arm64`.
- **CentOS / CentOS Stream**
  - 10: `amd64`, `arm64`.
  - 9: `amd64`, `arm64`.
  - 8: `amd64`, `arm64`.
  - 7: `amd64`.
- **Debian**
  - 13 Trixie: `amd64`, `arm64`.
  - 12 Bookworm: `amd64`, `arm64`.
  - 11 Bullseye: `amd64`.
  - 10 Buster: `amd64`, `i386`.
- **openSUSE Leap**
  - 16: `amd64`, `arm64`.
  - 15: `amd64`, `arm64`.
- **Oracle Linux**
  - 10: `amd64`, `arm64`.
  - 9: `amd64`, `arm64`.
  - 8: `amd64`, `arm64`.
  - 7: `amd64`.
- **Raspberry Pi OS**
  - 13 Trixie: `arm64`, `armhf`.
  - 12 Bookworm: `arm64`, `armhf`.
  - 11 Bullseye: `arm64`, `armhf`.
- **Red Hat Enterprise Linux**
  - 10: `amd64`, `arm64`.
  - 9: `amd64`, `arm64`.
  - 8: `amd64`, `arm64`.
  - 7: `amd64`.
- **Rocky Linux**
  - 10: `amd64`, `arm64`.
  - 9: `amd64`, `arm64`.
  - 8: `amd64`, `arm64`.
- **SUSE Linux Enterprise Server**
  - 16: `amd64`, `arm64`.
  - 15 SP4 and later service packs in the 15 series: `amd64`, `arm64`.
  - 12 SP4 and later service packs in the 12 series: `amd64`.
- **Ubuntu**
  - 26.04 Resolute: `amd64`, `arm64`.
  - 24.04 Noble: `amd64`, `arm64`.
  - 22.04 Jammy: `amd64`, `arm64`.
  - 20.04 Focal: `amd64`, `arm64`.
  - 18.04 Bionic: `amd64`, `i386`.
  - 16.04 Xenial: `amd64`, `i386`.


Distribution identifiers: `almalinux`, `amazonlinux`, `centos`, `debian`, `opensuse-leap`, `oraclelinux`, `raspios`, `rhel`, `rocky`, `sles`, `ubuntu`. For SLES, the `15-sp4` and `12-sp4` paths identify the minimum supported service pack; compatibility with later service packs must be tested separately.

Canonical architecture names: `amd64` = x86-64, `arm64` = AArch64, `i386` = 32-bit x86, and `armhf` = 32-bit ARM hard-float. Document minimum CPU/ISA and ABI requirements for each application and verify them on the target systems. `armhf` does not automatically mean compatibility with every ARM generation; `amd64` does not permit building with `-march=native`.

Including an old OS in the matrix is an application compatibility requirement, not a promise of security updates for that OS. EOL systems require pinned, archived build environments. Do not silently replace an old OS with a newer one or weaken verification requirements.

Windows, macOS, Alpine/musl, and other targets may be added through a separate extension of this contract; they do not replace the required Linux matrix above.

## 3. Artifact formats and storage layout

Every OS/version/architecture combination must have a `.tar.gz` archive containing a ready-to-run executable. Users must not need to compile anything on the destination machine. Additional `.deb` and `.rpm` packages are allowed. Merely storing packages in a bucket does not make it a complete APT/YUM repository: that requires separate, signed repository metadata.

```text
<project>/
  builds/
    build-1/
      manifest.json
      manifest.json.minisig
      SHA256SUMS
      linux/
        debian/12/amd64/node-agent-build-1-linux-debian-12-amd64.tar.gz
        debian/12/arm64/node-agent-build-1-linux-debian-12-arm64.tar.gz
        ubuntu/24.04/amd64/node-agent-build-1-linux-ubuntu-24.04-amd64.tar.gz
        ...remaining required targets...
    build-2/
      ...
  channels/
    stable.json
```

Example full URL:

```text
https://releases.soneyko.ai/node-agent/builds/build-1/linux/debian/12/amd64/node-agent-build-1-linux-debian-12-amd64.tar.gz
```

Archive contents:

```text
bin/node-agent
README.md
LICENSE
licenses/                  # licenses for bundled third-party components
systemd/node-agent.service # if the application is a systemd service
config/node-agent.example.yaml
```

`bin/<project>` and the application's license are required. Include other files as needed. Executables retain their executable bit. Archives must not contain absolute paths, `..` components, device nodes, setuid/setgid files, or links escaping the extraction directory. Extraction must not adopt root ownership from archive metadata.

Keep releases until they are explicitly retired; automated cleanup must not remove the previous working version. Unused, incomplete Builds may be removed through a separate, verified cleanup procedure, never through `sync --delete` in the release workflow.

## 4. Building and verifying compatibility

- Build in an isolated environment for the target distribution, or use a correctly pinned sysroot/toolchain. A binary built on a recent Ubuntu release must not be assumed compatible with CentOS 7 or Ubuntu 16.04.
- For glibc builds, check the minimum version, dynamic loader, and external libraries. Static binaries are acceptable if the application and its licenses permit them.
- Identical bytes may be reused across targets only after compatibility has actually been verified for each target. Every target still needs its own manifest entry and storage path.
- Each target requires an ELF format/architecture check, a `--version` invocation or equivalent self-test, and a functional smoke test. Cross-compiling without execution does not establish verified support.
- Test foreign architectures on native runners or in a correctly configured QEMU environment. Installing QEMU and provisioning target environments are responsibilities of the project's build script/infrastructure.
- Pin compiler/toolchain versions, sysroots, and container digests in the project. Commercial images/repositories require legitimate access; substituting a compatible distribution without verification is not acceptable.
- Run Node-based Actions on a modern runner. Launch the old target environment inside the build tool, not as a job container that cannot execute the Action itself.

## 5. Manifests, signatures, and channels

`manifest.json` contains the complete target list, Build number, project slug, commit SHA, UTC publication timestamp, archive sizes, and SHA-256 hashes. Sign **the exact manifest bytes**, not JSON reconstructed through serialization. A separate `SHA256SUMS` file is a convenience for manual checks; authoritative hashes are contained in the signed manifest.

Abbreviated structure example; the actual manifest must contain the entire matrix:

```json
{
  "schema_version": 1,
  "project": "node-agent",
  "build": 1,
  "release": "Build 1",
  "tag": "build-1",
  "source_commit": "0123456789abcdef0123456789abcdef01234567",
  "published_at": "2026-11-14T00:00:00Z",
  "artifacts": [
    {
      "os": "linux",
      "distribution": "debian",
      "version": "12",
      "arch": "amd64",
      "path": "node-agent/builds/build-1/linux/debian/12/amd64/node-agent-build-1-linux-debian-12-amd64.tar.gz",
      "size": 12345678,
      "sha256": "0000000000000000000000000000000000000000000000000000000000000000"
    }
  ]
}
```

These values are illustrative, not metadata for an existing release.

`channels/stable.json` is a small, atomically replaced pointer:

```json
{
  "project": "node-agent",
  "build": 1,
  "manifest": "node-agent/builds/build-1/manifest.json",
  "signature": "node-agent/builds/build-1/manifest.json.minisig"
}
```

The pointer is a discovery mechanism, **not a trusted source of code**. Clients must verify the manifest signature, project, Build, and selected target. Provision the public key in advance through a trusted channel; do not download it alongside the binary from the same mirror. Replacing the pointer must not allow installation of another project or rollback below a previously trusted Build number.

Protecting new installations against prolonged delivery of old but validly signed releases requires a bootstrap minimum Build and/or a protocol with expiring signed metadata, such as TUF. A monotonically increasing local counter alone does not protect a new machine against freeze attacks.

## 6. Cloudflare R2 and caching

1. Create a release bucket and connect the custom domain `releases.soneyko.ai`. A DNS record alone does not connect the bucket; add the domain in the R2 settings.
2. Choose public reads through the custom domain or a separate client authorization mechanism. The example below assumes public artifacts, not private distribution.
3. Publish through `https://<ACCOUNT_ID>.r2.cloudflarestorage.com`, using region `auto`. Object keys have no leading `/` and do not include the bucket name.
4. For `*/builds/*`, use `Cache-Control: public, max-age=31536000, immutable`.
5. For `*/channels/*`, use `Cache-Control: no-store` and a **Bypass cache** Cache Rule. This rule must take precedence over a general Cache Everything rule.
6. Do not cache publication errors/404s for long periods. If a URL was requested through the CDN before publication, invalidate the negative cache or wait for it to expire.
7. The public domain must serve binaries without HTML challenges, interactive authentication, or content transformations. Verify Content-Type, length, and SHA-256 by downloading from the actual domain.
8. Do not use `r2.dev` as the production URL. Scope the token to the relevant bucket; do not grant account-wide administrative privileges.

Sharing one bucket write key across repositories **does not isolate them by project prefix**: a compromised workflow can modify another project's objects. For independent trust boundaries, use separate buckets/credentials or a central publisher that validates the project and allowed prefix. The example below is intended only for trusted publishing repositories.

## 7. GitHub Organization variables and secrets

**Yes: organization settings are available to workflows in authorized repositories.** They do not need to be copied into every repository. They are not OS environment variables: Actions accesses them through `vars.NAME` and `secrets.NAME`.

In the organization, open **Settings → Secrets and variables → Actions → Variables / Secrets**. Under Repository access, choose `Selected repositories` and explicitly list trusted repositories. `All repositories` is acceptable for common, non-secret settings; do not grant write/signing secrets to every repository without reviewing its trust boundary.

### Organization Variables: non-secret settings

- `R2_ACCOUNT_ID`: Cloudflare account ID.
- `R2_BUCKET`: release bucket.
- `RELEASES_BASE_URL`: `https://releases.soneyko.ai`.
- `RELEASE_SIGNING_PUBLIC_KEY`: minisign public key, used for CI verification and documenting bootstrap configuration. Clients must obtain an independently trusted copy.

### Organization Secrets: trusted publishers only

- `R2_ACCESS_KEY_ID`: S3 access key authorized for the relevant bucket.
- `R2_SECRET_ACCESS_KEY`: secret portion of the S3 key.
- `RELEASE_SIGNING_KEY_B64`: base64-encoded, **unencrypted** minisign private key. Base64 is encoding, not encryption; protection comes from GitHub Secrets and restricted job access.

Generate the signing key outside CI using `minisign -G -W`. Never commit the private key or upload it to R2. Separate project keys are preferable to one organization-wide key when trust boundaries differ. A repository/environment secret with the same name can override the organization secret.

### Repository Variable

- `RELEASE_PROJECT`: the application's permanent slug, such as `node-agent`. Do not set one shared slug for the entire organization.

### Important access rules

- Organization-level variables and secrets have repository access policies. Private repositories on GitHub Free cannot access organization-level secrets/variables; use a suitable plan or repository-level settings.
- For identically named settings, the narrower scope generally takes precedence: environment → repository → organization. Environment values are available only to jobs using that environment; do not rely on them while planning the matrix before a job starts.
- Fork pull requests must not receive release secrets. Do not use `pull_request_target` to execute an untrusted build script with publishing credentials.
- Reusable workflows receive secrets explicitly or through `secrets: inherit` where permitted. An organization secret's existence does not imply automatic inheritance by a called workflow.
- GitHub creates `GITHUB_TOKEN` for the current repository. Creating a Release requires `contents: write` in the publish job; an organization-wide PAT is unnecessary. This token does not grant arbitrary access to other private repositories.
- Do not pass publishing keys to build jobs. Run the publish job only from the protected default branch. Protect `.github/workflows/` and `ci/` with branch protection/CODEOWNERS: anyone able to modify the trusted publisher may potentially use its secrets.
- For stronger protection, assign the publish job a GitHub Environment with required reviewers. Fully automated release without manual approval must be an explicit decision, not a bypass of review.

## 8. GitHub Actions example

Place the following files **in the application's source repository**. They are included here as documentation, not as an enabled workflow in this repository.

### `ci/targets.py`: a single machine-readable target matrix

```python
import json

TARGETS = {
    "almalinux": {"10": "amd64 arm64", "9": "amd64 arm64", "8": "amd64 arm64"},
    "amazonlinux": {"2023": "amd64 arm64", "2": "amd64 arm64"},
    "centos": {"10": "amd64 arm64", "9": "amd64 arm64", "8": "amd64 arm64", "7": "amd64"},
    "debian": {"13": "amd64 arm64", "12": "amd64 arm64", "11": "amd64", "10": "amd64 i386"},
    "opensuse-leap": {"16": "amd64 arm64", "15": "amd64 arm64"},
    "oraclelinux": {"10": "amd64 arm64", "9": "amd64 arm64", "8": "amd64 arm64", "7": "amd64"},
    "raspios": {"13": "arm64 armhf", "12": "arm64 armhf", "11": "arm64 armhf"},
    "rhel": {"10": "amd64 arm64", "9": "amd64 arm64", "8": "amd64 arm64", "7": "amd64"},
    "rocky": {"10": "amd64 arm64", "9": "amd64 arm64", "8": "amd64 arm64"},
    "sles": {"16": "amd64 arm64", "15-sp4": "amd64 arm64", "12-sp4": "amd64"},
    "ubuntu": {"26.04": "amd64 arm64", "24.04": "amd64 arm64", "22.04": "amd64 arm64", "20.04": "amd64 arm64", "18.04": "amd64 i386", "16.04": "amd64 i386"},
}

if __name__ == "__main__":
    print(json.dumps({"include": [
        {"distro": distro, "version": version, "arch": arch}
        for distro, versions in TARGETS.items()
        for version, arches in versions.items()
        for arch in arches.split()
    ]}, separators=(",", ":")))
```

### `ci/build-target.sh` contract

Implement and test this **project-specific** script for the application's language and dependencies. There is no universal command that builds every program.

```text
bash ci/build-target.sh <distro> <version> <arch> <build-number> <project-slug>
```

The script must build the program from the current checkout, test it in the target environment, and produce exactly one archive:

```text
out/<distro>/<version>/<arch>/<project>-build-<N>-linux-<distro>-<version>-<arch>.tar.gz
```

For example:

```bash
bash ci/build-target.sh debian 12 amd64 1 node-agent
# out/debian/12/amd64/node-agent-build-1-linux-debian-12-amd64.tar.gz
```

Exit 0 is allowed only after tests and smoke checks succeed. Map toolchain architecture names explicitly: for example, Go uses `386`, while the release path uses `i386`. For cgo/C/C++/Rust, changing an architecture variable is not enough: the target OS's sysroot, loader, and libraries are required.

### `.github/workflows/release.yml`

```yaml
name: Release
on:
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: release-${{ github.repository }}
  cancel-in-progress: false

jobs:
  plan:
    if: github.ref == format('refs/heads/{0}', github.event.repository.default_branch)
    runs-on: ubuntu-24.04
    outputs:
      matrix: ${{ steps.targets.outputs.matrix }}
    steps:
      - uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262 # v4
      - id: targets
        shell: bash
        run: echo "matrix=$(python3 ci/targets.py)" >> "$GITHUB_OUTPUT"

  build:
    needs: plan
    runs-on: ubuntu-24.04
    strategy:
      fail-fast: false
      max-parallel: 8
      matrix: ${{ fromJSON(needs.plan.outputs.matrix) }}
    env:
      PROJECT: ${{ vars.RELEASE_PROJECT }}
      BUILD: ${{ github.run_number }}
      DISTRO: ${{ matrix.distro }}
      VERSION: ${{ matrix.version }}
      ARCH: ${{ matrix.arch }}
    steps:
      - uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262 # v4
      - name: Build and test target
        shell: bash
        run: |
          set -euo pipefail
          [[ "$PROJECT" =~ ^[a-z0-9]+(-[a-z0-9]+)*$ ]]
          bash ci/build-target.sh "$DISTRO" "$VERSION" "$ARCH" "$BUILD" "$PROJECT"
          test -s "out/$DISTRO/$VERSION/$ARCH/$PROJECT-build-$BUILD-linux-$DISTRO-$VERSION-$ARCH.tar.gz"
      - uses: actions/upload-artifact@ea165f8d65b6e75b540449e92b4886f43607fa02 # v4
        with:
          name: target-${{ matrix.distro }}-${{ matrix.version }}-${{ matrix.arch }}
          path: out/
          if-no-files-found: error
          retention-days: 7

  publish:
    needs: [plan, build]
    runs-on: ubuntu-24.04
    permissions:
      contents: write
    env:
      PROJECT: ${{ vars.RELEASE_PROJECT }}
      BUILD: ${{ github.run_number }}
      R2_ACCOUNT_ID: ${{ vars.R2_ACCOUNT_ID }}
      R2_BUCKET: ${{ vars.R2_BUCKET }}
      RELEASES_BASE_URL: ${{ vars.RELEASES_BASE_URL }}
      RELEASE_SIGNING_PUBLIC_KEY: ${{ vars.RELEASE_SIGNING_PUBLIC_KEY }}
      AWS_ACCESS_KEY_ID: ${{ secrets.R2_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.R2_SECRET_ACCESS_KEY }}
      AWS_REQUEST_CHECKSUM_CALCULATION: when_required
      AWS_RESPONSE_CHECKSUM_VALIDATION: when_required
      GH_TOKEN: ${{ github.token }}
    steps:
      - uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262 # v4
      - uses: actions/download-artifact@d3f86a106a0bac45b974a628896c90dbdf5c8093 # v4
        with:
          pattern: target-*
          path: out/
          merge-multiple: true
      - name: Install publishing tools
        run: |
          sudo apt-get update
          sudo apt-get install -y minisign
          python3 -m venv "$RUNNER_TEMP/publisher"
          "$RUNNER_TEMP/publisher/bin/pip" install boto3
      - name: Publish immutable Build and promote stable
        env:
          RELEASE_SIGNING_KEY_B64: ${{ secrets.RELEASE_SIGNING_KEY_B64 }}
        shell: bash
        run: |
          set -euo pipefail
          umask 077
          key="$RUNNER_TEMP/release-signing.key"
          trap 'rm -f "$key"' EXIT
          printf '%s' "$RELEASE_SIGNING_KEY_B64" | base64 --decode > "$key"
          export RELEASE_SIGNING_KEY_FILE="$key"
          "$RUNNER_TEMP/publisher/bin/python" ci/publish.py
```

Update pinned Action SHAs through reviewed pull requests. For production CI, also pin publishing dependency versions and hashes in the project's lock file. The `boto3` installation above demonstrates minimal setup, not a reproducible dependency lock. GitHub-hosted runners must provide `gh`; install it separately on other runners.

### `ci/publish.py`: signing, uploading, readback, GitHub Release, stable

This example uses regular `PutObject` and limits each file to 128 MiB. Larger artifacts require a separate multipart uploader with the create-only condition on **CompleteMultipartUpload**, not just at upload initiation.

```python
import datetime
import hashlib
import json
import os
from pathlib import Path
import re
import subprocess
import sys
import tempfile
import urllib.request

import boto3
from botocore.exceptions import ClientError


def digest(data):
    return hashlib.sha256(data).hexdigest()


def encode(value):
    return (json.dumps(value, ensure_ascii=False, sort_keys=True, indent=2) + "\n").encode()


project = os.environ["PROJECT"]
build = int(os.environ["BUILD"])
account = os.environ["R2_ACCOUNT_ID"]
bucket = os.environ["R2_BUCKET"]
base = os.environ["RELEASES_BASE_URL"].rstrip("/")
assert re.fullmatch(r"[a-z0-9]+(-[a-z0-9]+)*", project)
assert re.fullmatch(r"[0-9a-fA-F]{32}", account)
assert build > 0 and base.startswith("https://")
assert subprocess.check_output(["git", "rev-parse", "HEAD"], text=True).strip() == os.environ["GITHUB_SHA"]

tag = f"build-{build}"
prefix = f"{project}/builds/{tag}"
channel_key = f"{project}/channels/stable.json"
s3 = boto3.client("s3", endpoint_url=f"https://{account}.r2.cloudflarestorage.com", region_name="auto")

# Read the current stable pointer and ETag for compare-and-swap, not a blind overwrite.
try:
    old = s3.get_object(Bucket=bucket, Key=channel_key)
except ClientError as exc:
    if exc.response["ResponseMetadata"]["HTTPStatusCode"] != 404:
        raise
    channel_condition = {"IfNoneMatch": "*"}
else:
    previous = json.loads(old["Body"].read())
    assert previous["project"] == project and previous["build"] < build
    channel_condition = {"IfMatch": old["ETag"]}

ref = subprocess.run(["git", "ls-remote", "--exit-code", "origin", f"refs/tags/{tag}"], capture_output=True)
assert ref.returncode == 2, "Tag exists or remote lookup failed; do not reuse a Build"

matrix = json.loads(subprocess.check_output([sys.executable, "ci/targets.py"]))["include"]
assert len({(t['distro'], t['version'], t['arch']) for t in matrix}) == len(matrix)
artifacts = []
uploads = []
for target in matrix:
    distro, version, arch = (target[k] for k in ("distro", "version", "arch"))
    filename = f"{project}-{tag}-linux-{distro}-{version}-{arch}.tar.gz"
    local = Path("out") / distro / version / arch / filename
    assert local.is_file() and not local.is_symlink(), str(local)
    body = local.read_bytes()
    assert 0 < len(body) <= 128 * 1024 * 1024
    key = f"{prefix}/linux/{distro}/{version}/{arch}/{filename}"
    artifacts.append(dict(os="linux", distribution=distro, version=version, arch=arch,
                          path=key, size=len(body), sha256=digest(body)))
    uploads.append((local, key, body, "application/gzip"))
assert {p.resolve() for p in Path("out").rglob("*.tar.gz")} == {p.resolve() for p, *_ in uploads}

manifest = dict(schema_version=1, project=project, build=build, release=f"Build {build}", tag=tag,
                source_commit=os.environ["GITHUB_SHA"],
                published_at=datetime.datetime.now(datetime.timezone.utc).isoformat(), artifacts=artifacts)
release_dir = Path("release-metadata")
release_dir.mkdir(exist_ok=True)
manifest_file = release_dir / "manifest.json"
manifest_file.write_bytes(encode(manifest))
signature_file = release_dir / "manifest.json.minisig"
subprocess.run(["minisign", "-S", "-s", os.environ["RELEASE_SIGNING_KEY_FILE"],
                "-m", str(manifest_file), "-x", str(signature_file)], check=True)
subprocess.run(["minisign", "-V", "-P", os.environ["RELEASE_SIGNING_PUBLIC_KEY"],
                "-m", str(manifest_file), "-x", str(signature_file)], check=True)
checksums = release_dir / "SHA256SUMS"
checksums.write_text("".join(f"{a['sha256']}  {a['path'][len(prefix)+1:]}\n" for a in artifacts))
for local, content_type in [(manifest_file, "application/json"),
                            (signature_file, "text/plain"), (checksums, "text/plain")]:
    uploads.append((local, f"{prefix}/{local.name}", local.read_bytes(), content_type))

# Enforce create-only on the write itself, including concurrent writes.
for local, key, body, content_type in uploads:
    s3.put_object(Bucket=bucket, Key=key, Body=body, ContentType=content_type,
                  CacheControl="public, max-age=31536000, immutable", IfNoneMatch="*")
    received = s3.get_object(Bucket=bucket, Key=key)["Body"].read()
    assert digest(received) == digest(body), f"R2 readback mismatch: {key}"
    with urllib.request.urlopen(f"{base}/{key}", timeout=120) as response:
        assert digest(response.read()) == digest(body), f"Public readback mismatch: {key}"

# Attach archives and signed metadata to GitHub Release; tag the source commit.
subprocess.run(["gh", "release", "create", tag, "--target", os.environ["GITHUB_SHA"],
                "--title", f"Build {build}", "--notes", f"Source: {os.environ['GITHUB_SHA']}\nManifest: {base}/{prefix}/manifest.json",
                *[str(local) for local, *_ in uploads]], check=True)
release = json.loads(subprocess.check_output(["gh", "release", "view", tag,
                                               "--json", "name,tagName,isDraft,assets"]))
assert release["name"] == f"Build {build}" and release["tagName"] == tag and not release["isDraft"]
assert {a["name"] for a in release["assets"]} == {p.name for p, *_ in uploads}

channel = encode(dict(project=project, build=build, manifest=f"{prefix}/manifest.json",
                      signature=f"{prefix}/manifest.json.minisig"))
s3.put_object(Bucket=bucket, Key=channel_key, Body=channel, ContentType="application/json",
              CacheControl="no-store", **channel_condition)
assert s3.get_object(Bucket=bucket, Key=channel_key)["Body"].read() == channel
with urllib.request.urlopen(f"{base}/{channel_key}", timeout=30) as response:
    assert response.read() == channel, "Channel cache mismatch"
print(f"Published Build {build}: {len(artifacts)} targets")
```

If upload, readback, or GitHub Release creation fails, the workflow fails and must not promote `stable`. If a channel write has an ambiguous network outcome, read its actual value: a timeout does not prove the write did not commit. A failed CDN check after the write does not imply automatic rollback either. Do not delete objects to reuse the same Build; start a new run with a new number.

Every archive must have a unique filename to avoid GitHub Release asset conflicts. The example verifies public file availability and the Release asset inventory. For stricter acceptance, also download GitHub assets and compare their hashes.

## 9. Installation and updates on machines

- A trusted bootstrap provisions the public key, expected project slug, target, and initial minimum Build. Never execute downloaded `curl | bash` content without signature verification.
- Configuration: `/etc/<project>/`; data: `/var/lib/<project>/`; versions: `/opt/<project>/releases/build-N/`; active version: `/opt/<project>/current`.
- A `systemd timer` or another updater reads `stable.json`, downloads the manifest and signature, verifies the minisign signature and expected project/Build/target, then checks the archive's size and SHA-256.
- Serialize updates with a local lock. Extract into a new temporary directory with path and file-type checks, not over the running version.
- After verification, atomically switch `current`, perform a controlled restart, and run a health check. On failure, restore the saved working version and restart the service.
- Store the highest trusted Build number separately from the active version: a local rollback must not let the mirror offer older Builds as new updates. Do not update to the same number.
- Never overwrite configuration or data. Rolling back a binary does not roll back a database; incompatible migrations require a separate plan and backup.
- Roll out to a canary group first, then the remaining machines. A central bucket must not mean restarting the entire fleet at once.

## 10. New project acceptance checklist

- [ ] Project slug, source repository, workflow owner, and `Build N` / `build-N` naming are fixed.
- [ ] All required distro/version/architecture targets are present in `ci/targets.py`.
- [ ] `ci/build-target.sh` actually builds and tests every target rather than renaming one binary.
- [ ] Licenses, CPU/ABI/glibc compatibility, and the absence of secrets in artifacts are verified.
- [ ] R2 custom domain, TLS, read policy, Cache Rules, and bucket permissions are configured.
- [ ] Organization variables/secrets are available only to the intended publishers; the project slug is configured as a repository variable.
- [ ] Actions/toolchains/dependencies are pinned; the default branch and CI files are protected.
- [ ] The full matrix passes; the GitHub Release has the exact title `Build N` and points to the correct commit.
- [ ] Archives, manifest, signature, and checksums are available at the expected URLs; readback matches.
- [ ] `stable` is promoted last and is not served from a stale cache.
- [ ] Installation, canary upgrade, a failed health check, rollback, and configuration/data preservation are tested.

Changing this specification does not mean that CI, R2, or an updater is already configured. Enabling them and establishing actual target support require separate build and test results.
