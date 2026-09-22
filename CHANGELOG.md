# SDK 0.3.0

Regenerated from LoonFS `6ea950f9f5e0030066e18061efe787dcaf9be4db` (PR #996 plus the
Python compatibility fix in PR #1000). Targets LoonFS API v0.3.x.

Small-file helpers now prepare content inline when advertised, up to the smaller
of the server inline limit and 64 KiB. Streaming detection has bounded lookahead
and preserves bytes when falling back to normal uploads. Publication retries
must reuse the prepared value, commit ID, and all other publication inputs.

## Migration

Preparation now returns `PreparedFile`: inline content or the existing staged
`PreparedContent`. Passing the value directly to prepared publication continues
to work. Callers that annotate the result or inspect staged reference/token
fields must accept the union and narrow to the staged variant first.

This regeneration also includes the current API's access-control models and
subject/principal context support, including proxy header filtering. Existing
clients do not need to configure subject context; when supplied, principal scope
and principals must be configured together.
