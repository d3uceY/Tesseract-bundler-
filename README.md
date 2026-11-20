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

Tesseract's version and the bundle revision are separate. Create a **new** tag
for every bundle release, including rebuilds of the same Tesseract version:

```bash
# Pin the upstream version, commit, and push the changes to main first.
printf '5.5.3\n' > TESSERACT_VERSION
# Create a new bundle revision from the reviewed commit.
git tag tesseract-5.5.3-bundle.1
git push origin tesseract-5.5.3-bundle.1
```

A tag push starts the build. Manual dispatch requires both the Tesseract version
and a new release tag in the same `tesseract-<version>-bundle.<revision>` format.
Existing releases are never reused. Consumers must update their release URL to
the new tag; archive filenames still use the upstream Tesseract version.

The release is created as a draft. The workflow downloads its assets, verifies
checksums, inspects both macOS binaries for non-system dependencies and smoke
tests the Apple Silicon binary before publishing. If verification fails, the
draft stays unpublished for investigation. A clean-machine OCR test is still
recommended before users adopt a new bundle. Never rerun an old tag or an old
workflow that can overwrite an existing release.

## Workflow notes

- `runner.*` is **not** available in a job-level `env:` block — use `github.workspace`
  instead. Getting this wrong makes the whole workflow invalid (it fails with 0 jobs).
- `VCPKG_ROOT` lives beside the checkout (`${{ github.workspace }}/../vcpkg`) so the
  Tesseract source tree stays pristine.
- Windows uses the `x64-windows-static` triplet (explicit `-static` suffix is required);
  the `*-linux` and `*-osx` default triplets already build static libraries.
- macOS runners: `macos-13` is retired. Use `macos-15-intel` for Intel — a job on a label
  with no runner queues until the 24h limit instead of failing fast.
