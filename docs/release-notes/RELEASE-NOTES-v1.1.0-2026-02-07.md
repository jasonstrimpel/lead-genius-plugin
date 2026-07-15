# Release Notes

## v1.1.0 (2026-02-07)

### Changes

- **Output directory simplified**: All pipeline output now writes to `./{slug}/` at the project root instead of `./output/{slug}/`. Shorter paths and easier to find your results.
- **Marketplace support**: Added `marketplace.json` so the plugin can be installed via `/plugins marketplace add jasonstrimpel/lead-genius-plugin`.
- **Cross-platform README**: Installation and setup instructions now include both macOS/Linux and Windows (PowerShell) commands.
- **Additional troubleshooting**: Added guidance for plugin discovery issues and WebSearch errors.

### Fixes

- Fixed marketplace source field schema validation error (use URL object format instead of plain string)

### Upgrade Notes

If upgrading from v1.0.0, your existing output in `./output/{slug}/` will not be moved. New pipeline runs will write to `./{slug}/` at the project root.