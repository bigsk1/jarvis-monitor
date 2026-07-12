# Release Runbook — jarvis-monitor

Point the agent at this file to ship a change end-to-end: **verify → commit → push to
GitHub main → multi-arch Docker build with attestations → push → verify on Docker Hub**.

- **GitHub repo:** `bigsk1/jarvis-monitor` (branch `main`, https remote)
- **Docker Hub repo:** `bigsk1/jarvis-monitor`
- **Working dir:** this repo is a standalone git repo (no nested/parent repo).
- Docker runs as non-root `monitor` user (added to `root` group for socket access). That is intended — do not "fix" it.

---

## 0. Pre-flight: secrets & hygiene (ALWAYS do first)

Never commit or bake secrets into the image. Verify before anything else:

```bash
git status --short
git check-ignore .env && echo ".env ignored OK"      # must print a match
git ls-files | grep -iE 'env' || true                # should ONLY show .env.example
grep -iE 'key|token|secret|pass' .env.example        # must be placeholders only, no real values
```

Rules:
- `.env` MUST stay untracked/ignored. `.env.example` is the only tracked env file and holds
  placeholders only (e.g. `JARVIS_API_KEY=your-secure-api-key-here`).
- If `git status` shows a real `.env`, `*.key`, token, or credential staged → STOP, unstage, fix `.gitignore`.
- Ignore files that must exist:
  - `.gitignore` ignores `.env`, `.env.*` (keeps `!.env.example`), `__pycache__/`, venvs, IDE, logs, `/tmp/`.
  - `.dockerignore` excludes `.git/`, `.env`, `.env.*`, `*.md`, `docker-compose.yml`, `Dockerfile`, caches — so no secrets/junk enter the image.

---

## 1. Verify the code change

- Review the diff: `git diff` (and `git diff --stat`).
- Optional local smoke test (needs Docker socket + env): `python3 monitor.py`.
- Confirm the change matches what the user intended; call out anything risky.

---

## 2. Version bump (keep published tags immutable)

Published Docker tags are treated as immutable — **do not overwrite an existing version tag.**

1. Check what already exists on Docker Hub (use the dockerhub MCP `listRepositoryTags`, or ask).
2. Pick the next semver (e.g. `1.1.1` exists → new build is `1.1.2`).
3. Update BOTH places so they match:
   - `Dockerfile`: `LABEL org.opencontainers.image.version="X.Y.Z"`
   - `AGENTS.md`: the `-t bigsk1/jarvis-monitor:X.Y.Z` line and the "Update version tag (X.Y.Z)" note.

```bash
NEW=1.1.2   # <-- set this
sed -i "s|org.opencontainers.image.version=\"[0-9.]*\"|org.opencontainers.image.version=\"$NEW\"|" Dockerfile
sed -i "s|bigsk1/jarvis-monitor:[0-9.]* |bigsk1/jarvis-monitor:$NEW |; s|Update version tag ([0-9.]*)|Update version tag ($NEW)|" AGENTS.md
grep image.version Dockerfile
```

---

## 3. Commit & push to main

```bash
git add -A
git status                      # final check: no secrets staged
git commit -m "<concise message matching repo style>"
git push origin main
```

Notes:
- The git push uses the repo's https credentials (already configured). If it fails with auth →
  STOP and tell the user to re-auth.
- Match existing commit style (short imperative subject + bulleted body for larger changes).

---

## 4. Build multi-arch image WITH attestations, then push

The default `docker` driver **cannot** do multi-arch or attestations. You must use a
`docker-container` buildx builder and register QEMU for arm64. One-time setup per machine
(safe to re-run; recreate the builder AFTER installing binfmt so it sees arm64):

```bash
# 1) register emulators (arm64) so cross-build works
docker run --privileged --rm tonistiigi/binfmt --install arm64

# 2) create/refresh a container-driver builder
docker buildx rm jarvis-builder 2>/dev/null || true
docker buildx create --name jarvis-builder --driver docker-container --bootstrap

# 3) confirm both platforms are listed (must include linux/arm64)
docker buildx inspect jarvis-builder | grep -i platform
```

Confirm you are logged in to Docker Hub as `bigsk1` (`~/.docker/config.json` has an
`index.docker.io` auth). If not logged in / auth expired → STOP and ask the user to re-auth.

Build + push (update `NEW` to the version from step 2):

```bash
NEW=1.1.2
docker buildx build \
  --builder jarvis-builder \
  --platform linux/amd64,linux/arm64 \
  --sbom=true \
  --provenance=mode=max \
  -t bigsk1/jarvis-monitor:latest \
  -t bigsk1/jarvis-monitor:$NEW \
  --push \
  .
```

Why these flags (they drive the Docker Scout / supply-chain grading):
- `--platform linux/amd64,linux/arm64` → multi-arch manifest list.
- `--sbom=true` → Software Bill of Materials attestation.
- `--provenance=mode=max` → full build provenance attestation.
- `--push` → pushes the manifest list + attestation manifests directly to the registry.

The build log should show `exporting attestation manifest ...` for each platform and
`pushing manifest for ...:latest` and `...:$NEW`. Those "unknown/unknown" entries on Docker
Hub are the attestation manifests — that's expected and correct.

---

## 5. Verify on Docker Hub

Use the dockerhub MCP `getRepositoryTag` (namespace `bigsk1`, repo `jarvis-monitor`, tag `$NEW`)
and confirm:
- `linux/amd64` and `linux/arm64` images present.
- Two `unknown/unknown` entries present (= SBOM + provenance attestations).
- `latest` and `$NEW` share the same top-level digest (they were pushed together).

CLI alternative:
```bash
docker buildx imagetools inspect bigsk1/jarvis-monitor:$NEW
```

---

## 6. Cleanup

```bash
docker buildx rm jarvis-builder    # optional: remove the temporary builder
```

---

## Quick checklist

- [ ] No secrets staged; `.env` ignored, `.env.example` placeholders only
- [ ] Version bumped in `Dockerfile` + `AGENTS.md` (not overwriting an existing tag)
- [ ] Committed + pushed to `main`
- [ ] buildx `docker-container` builder + arm64 QEMU ready
- [ ] Built with `--platform linux/amd64,linux/arm64 --sbom=true --provenance=mode=max --push`
- [ ] Tags `latest` + `X.Y.Z` verified on Docker Hub with both arches + attestations
