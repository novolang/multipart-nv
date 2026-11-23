# Changelog

All notable changes to multipart-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.1.0] — 2026-11-23

### Added

- `mpwrite`, a writer of `multipart/form-data` bodies: whole with `body` and `sealed`, or in pieces with `head_of`, `tail` and `closing`, and `overhead_of` for the `Content-Length` of a streamed body.
- `mpboundary`, the checked boundary, the choice of one that no content holds, and the `Content-Type` header value.
- `mpread`, a reader of a complete body, for tests.
- `mperr`, the faults of both.
