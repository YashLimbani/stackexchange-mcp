# Changelog

## [0.1.0] - Unreleased

First release.

- 6 read-only tools: `search_questions`, `get_question`, `search_by_error_message`, `search_tags`, `get_tag_overview`, `get_quota_status`.
- 3 resources (`stackexchange://sites`, `stackexchange://help/search-tips`, `stackexchange://quota`) and 3 prompts (`debug_error`, `compare_approaches`, `tag_primer`).
- Optional API key, API backoff handling, request spacing, retries with exponential delay, 5-minute response cache and quota tracking.
- Typed errors with messages written for AI models.
- Post bodies converted from HTML to Markdown with code blocks preserved.
