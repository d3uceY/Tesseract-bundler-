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

The Tesseract version is pinned in `TESSERACT_VERSION`. A tag such as
`tesseract-5.5.3` publishes assets under the same release and archive names
used by Vectile. The executable remains `bin/tesseract`.

To repair the existing 5.5.3 release, merge the reviewed workflow changes and
run **Actions → Build Tesseract Release → Run workflow** from the updated main
branch with version `5.5.3`. Do not rerun the old tag workflow: it runs the
workflow stored in that old commit. The manual run rebuilds all platforms and
replaces the assets and checksums at the existing release URL.

The workflow validates the build before upload, then downloads the release
assets, verifies checksums, checks both macOS binaries for non-system
libraries, and smoke-tests the Apple Silicon binary. Asset replacement is not
atomic: downloads during the upload may fail checksum verification and should
be retried. Check the workflow's final verification before announcing a
repair. A clean-machine OCR test is still recommended. Existing Vectile
installations may need to remove and reinstall the OCR plugin.

For a new Tesseract version, update `TESSERACT_VERSION`, commit to main, and
push the corresponding `tesseract-<version>` tag. Run only one release at a
time and never rerun an old workflow that can overwrite verified assets.

## Workflow notes

- `runner.*` is **not** available in a job-level `env:` block — use `github.workspace`
  instead. Getting this wrong makes the whole workflow invalid (it fails with 0 jobs).
- `VCPKG_ROOT` lives beside the checkout (`${{ github.workspace }}/../vcpkg`) so the
  Tesseract source tree stays pristine.
- Windows uses the `x64-windows-static` triplet (explicit `-static` suffix is required);
  the `*-linux` and `*-osx` default triplets already build static libraries.
- macOS runners: `macos-13` is retired. Use `macos-15-intel` for Intel — a job on a label
  with no runner queues until the 24h limit instead of failing fast.
