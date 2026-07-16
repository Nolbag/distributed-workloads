# Ray Runtime Container Images

Sources for the Ray runtime container images (CUDA, ROCm, and Training Hub variants) provided by Distributed Workloads in OpenShift AI.

---

## Konflux Preflight Certification Pattern

Every new Ray GPU runtime image (CUDA, ROCm, or Training Hub) **must** follow this pattern to pass Konflux's `ecosystem-cert-preflight-checks` task. Skipping it causes two specific certification failures: `HasLicense` and `HasModifiedFiles`.

> Reference implementation: `images/runtime/ray/cuda/2.55.1-py312-cu129-th081/Dockerfile` (fixed in [PR #948](https://github.com/opendatahub-io/distributed-workloads/pull/948)).

### When to apply this

Only in **new** version-bump directories (new Ray version, new CUDA/ROCm/Python version, new Training Hub variant, etc.). **Never** retrofit it into an already-published, already-certified directory. Editing a published image's Dockerfile triggers an unnecessary Konflux rebuild of a release that's already out.

### 1. `HasLicense` — install the NVIDIA license into `/licenses/`

CUDA and Training Hub images ship `NGC-DL-CONTAINER-LICENSE`. Preflight requires it under `/licenses/`, not `/`:

```dockerfile
RUN mkdir -p /licenses
COPY NGC-DL-CONTAINER-LICENSE /licenses/
```

(Not applicable to ROCm images. They don't ship this license file.)

### 2. `HasModifiedFiles` — restore `redhat-rpm-config` after `yum` installs

Installing `gcc`/toolchain packages (directly, or transitively via packages like `cuda-minimal-build`) can trigger an RPM scriptlet that rewrites `usr/lib/rpm/redhat/redhat-annobin-cc1`, a file owned by `redhat-rpm-config`. If the rewritten file no longer matches what the package originally shipped, Konflux flags it as a disallowed modification.

Fix: end the **final** `yum install`/cleanup chain in the Dockerfile with `rpm --restore redhat-rpm-config`, after the cache cleanup:

```dockerfile
RUN yum install -y \
    <packages> \
    && yum clean all \
    && rm -rf /var/cache/yum/* \
    && rpm --restore redhat-rpm-config
```

A single restore at the end of the file's last `yum`-touching layer is enough: it covers any drift left behind by earlier layers (such as the one installing `gcc`), as long as no further `yum install` runs afterward. Running it after `yum clean all`/cache removal specifically also ensures nothing else in that same layer re-modifies the file before it's committed.

Whether this actually gets triggered depends on the exact package combination, and some image variants may not hit it today. Add the line unconditionally anyway: it's a no-op when nothing was modified, so there's no downside to including it defensively.

### Verifying locally before opening a PR

```bash
cd images/runtime/ray/<cuda|rocm>/<version-dir>
podman build --platform linux/amd64 -t ray-test:local -f Dockerfile .

# HasLicense check — expect the license listed under /licenses/
podman run --rm --entrypoint sh ray-test:local -c "ls -la /licenses/"

# HasModifiedFiles check — expect no output
podman run --rm --entrypoint sh ray-test:local -c "rpm -V redhat-rpm-config"
```

This is a fast local pre-check, not a substitute for the real thing. The authoritative signal is the Konflux PR pipeline (`ecosystem-cert-preflight-checks` task), which runs automatically against PRs targeting the `red-hat-data-services/distributed-workloads` downstream repo.
