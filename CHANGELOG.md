# Changelog

## 0.2.3
- **Fixed a packaging bug that caused `nexus` to fail with "permission denied" on some installs.**
  The npm launcher (`bin/nexus.js`) was published without its executable bit, so package managers
  that preserve tarball file modes (pnpm, and some yarn/extraction paths) could not execute it. The
  launcher now ships mode `0755`.

## 0.2.2
- Documentation refresh (README styling); no functional changes.

## 0.2.1
- Public product repository and refreshed documentation.
- Introduced the **Open Alexandria Project** pledge and the `nexus alexandria` opt-in/opt-out command.

## 0.2.0
- First public release: `@swfte/nexus` (npm) and `swfte-nexus` (PyPI).
- Ships as a single compiled, self-contained binary — no runtime dependencies.
- Platforms: macOS (arm64, x64 — Developer ID signed + notarized) and Linux (x64, arm64; glibc + musl).
- Savings (compression, tool-output filtering, reuse/recall, codified macros, outcome routing),
  in-flight policy enforcement, a grounded audit trail, and codebase mapping.
