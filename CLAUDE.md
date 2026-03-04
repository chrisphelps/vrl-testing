# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Run all unit tests
vector test vector.yaml tests.yaml

# Validate config syntax only
vector validate vector.yaml

# Interactive VRL REPL (for testing expressions)
vector vrl
```

## Architecture

AWS CloudWatch Logs → Kinesis Firehose → Vector → Elasticsearch

**Pipeline (vector.yaml):**
1. `kinesis_http` — `aws_kinesis_firehose` source on port 8080
2. `cloudwatch_decode` — remap: decodes the CloudWatch Logs subscription envelope using `parse_aws_cloudwatch_log_subscription_message(.message)`
3. `add_fields` — remap: injects enrichment fields, merges JSON message fields, normalizes timestamp, deletes envelope fields
4. `elasticsearch` — sink writing to `bulk.index` (date-stamped index name)

**tests.yaml** — Vector unit tests injected at individual transform stages.

## VRL / Config gotchas

- `parse_aws_cloudwatch_log_subscription_message` expects **GZIP-compressed bytes**. Plain JSON input aborts at runtime ("expected ident at line 1 column 2"). The success path cannot be unit-tested via `log_fields` in YAML — use the REPL or an integration test with a real Firehose payload.
- Output field names are **snake_case**: `log_group`, `log_stream`, `log_events`, `message_type`, `subscription_filters`.
- Use `merge!(., parsed)` (with `!`); `merge()` without `!` fails static type checking.
- Use `parse_timestamp(val, format: "%+")` — `to_timestamp` does not exist in VRL.
- `$WORD` anywhere in the config (including inside VRL `source:` blocks) is interpolated as an env var. Use `$$WORD` to escape (e.g., `$$LATEST` in log stream names in tests).
- Elasticsearch `index` is not a top-level field; it must be under `bulk.index`.
