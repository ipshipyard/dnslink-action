# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## Unreleased

## [1.1.0] - 2026-01-05

### Added
- Cloudflare: `cf_record_id` is now optional, the action auto-discovers or creates TXT records
- Cloudflare: smart `_dnslink.` prefix handling avoids double-prefix if already present
- Documentation: Cloudflare setup guide with links to official docs

### Changed
- Cloudflare: TXT record content now properly quoted to avoid dashboard warnings
- Cloudflare: improved error handling with GitHub Actions annotations

## [1.0.0] - 2025-08-25

### Added
- Added support for Gandi DNS provider

### Fixed
- Support `workflow_run` events for correct GitHub commit status (e.g., secure fork PR workflows)

## [0.1.0] - 2025-02-20

- Initial release of of the dnslink-action
