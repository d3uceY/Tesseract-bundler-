# Tesseract-bundler

Builds self-contained [Tesseract OCR](https://github.com/tesseract-ocr/tesseract) archives — one
per OS/arch — and publishes them as a GitHub release with `SHA256SUMS`.

Self-containment is the point. Each archive must run on a clean machine with no shared
libraries installed beyond the OS base, so a bundle that piggybacks on the build machine's
`liblept`, `libicu`, `libtiff`, … is useless. Leptonica (and its image codecs) is therefore
built from source via vcpkg and linked statically; each Unix job *asserts* the result has no
non-system dependencies, so a regression fails the build instead of shipping a broken bundle.

ICU is deliberately **not** installed — the Windows bundle never had it and OCR does not need it.

## Artifacts

Five archives per release, named `tesseract-<version>-<platform>.<ext>`:

| Platform      | Runner            | Archive  |
| ------------- | ----------------- | -------- |
| windows-amd64 | windows-2025      | `.zip`   |
| linux-amd64   | ubuntu-24.04      | `.tar.gz`|
| linux-arm64   | ubuntu-24.04-arm  | `.tar.gz`|
| macos-amd64   | macos-15-intel    | `.tar.gz`|
| macos-arm64   | macos-14          | `.tar.gz`|

Each archive unpacks to `tesseract/` with a portable runtime layout — `bin/` plus
`tessdata/` (`eng` + `osd`). No `lib/`, no `include/`, no `.pdb`.

## Running a release

The Tesseract version is pinned in `TESSERACT_VERSION` (one line, e.g. `5.5.3`). The workflow
reads that file — **not** the git tag name — to decide what to build, so bump the file first
if you want a different version.

### Releasing a new version

```bash
# 1. Pin the Tesseract version to build.
echo "5.5.3" > TESSERACT_VERSION
git add TESSERACT_VERSION
git commit -m "Bump Tesseract to 5.5.3"
git push origin main

# 2. Tag main and push THE TAG. The push is what triggers the build.
git tag tesseract-5.5.3
git push origin tesseract-5.5.3
```

Pushing the tag runs `.github/workflows/release.yaml`, which builds all five platforms and,
on success, creates the GitHub release `tesseract-5.5.3` with the archives and `SHA256SUMS`.
Expect roughly **30–90 minutes** (leptonica builds from source; the vcpkg binary cache makes
repeat runs much faster).

### Re-running / re-releasing the same version

Tags are immutable on the remote once pushed, so to retrigger a build for a version that
already has a tag, move the tag to the commit you want and force-push it:

```bash
git tag -f tesseract-5.5.3 main
git push --force origin tesseract-5.5.3
```

Or skip tagging entirely and run the workflow manually from the Actions tab
(**Actions → Build Tesseract Release → Run workflow**), typing the version to build.

## Important rules

- **Tag `main`, not an old commit.** A tag push runs the workflow *as it existed at the
  tagged commit*. If the tag points at an outdated commit, it runs an outdated pipeline —
  which historically meant non-self-contained Unix bundles. Always tag the current `main`.
- **One tag push = one build.** Pushing several tags at once starts several concurrent runs
  that all publish the same release and race on the same asset filenames. Push one tag.
- **The release name comes from `TESSERACT_VERSION`**, not the tag: a tag named `v0.2.0` still
  produces a release named `tesseract-5.5.3`. Name tags for the Tesseract version to avoid
  confusion.

## Workflow notes

- `runner.*` is **not** available in a job-level `env:` block — use `github.workspace`
  instead. Getting this wrong makes the whole workflow invalid (it fails with 0 jobs).
- `VCPKG_ROOT` lives beside the checkout (`${{ github.workspace }}/../vcpkg`) so the
  Tesseract source tree stays pristine.
- Windows uses the `x64-windows-static` triplet (explicit `-static` suffix is required);
  the `*-linux` and `*-osx` default triplets already build static libraries.
- macOS runners: `macos-13` is retired. Use `macos-15-intel` for Intel — a job on a label
  with no runner queues until the 24h limit instead of failing fast.
