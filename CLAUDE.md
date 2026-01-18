# CLAUDE.md - Slack CLI

Project-specific guidance for Claude Code.

## Overview

CLI tool for Slack using user tokens (`xoxp-`). Sends messages as the authenticated user, not a bot.

See [README.md](README.md) for setup, installation, and usage.

## Architecture

### File Organization

| Path | Purpose |
|------|---------|
| `slack_cli/cli.py` | Main CLI entry point, Click groups |
| `slack_cli/client.py` | WebClient singleton, error handling, token masking |
| `slack_cli/config.py` | Token storage (`~/.config/slack-cli/token`) |
| `slack_cli/commands/*.py` | One file per command group (message, channel, user, etc.) |
| `slack_cli/utils/resolution.py` | Smart user resolution (ID/email/username) |
| `slack_cli/validators.py` | Input validation callbacks for Click |
| `slack_cli/formatters.py` | Rich table output, JSON output |

### Code Patterns

**Commands**: Click group + subcommands pattern
```python
@click.group()
def message():
    """Message commands."""
    pass

@message.command()
@click.argument("channel", callback=validate_channel)
def send(channel, text):
    ...
```

**User Resolution**: `resolve_user(client, identifier)` handles multiple formats:
- User ID (`U12345678`) → `users.info` (direct, fast)
- Email (`user@example.com`) → `users.lookupByEmail` (direct, fast)
- Username (`@name` or `name`) → `users.list` with pagination (fallback)

**Pagination**: `paginate_until(method, key, limit=, find_func=)` for cursor-based APIs

**Error Handling**: `handle_api_error(e)` masks tokens in output

**Output**: `--json` flag → `output_json()`, else Rich tables via `format_*()` functions

### Token Priority

1. `SLACK_USER_TOKEN` environment variable (for CI, scripts, wrappers)
2. `~/.config/slack-cli/token` file (interactive use via `slack auth configure`)

### Validation Limits

| Limit | Value | Source |
|-------|-------|--------|
| `MAX_MESSAGE_LENGTH` | 40,000 chars | Slack API limit |
| `MAX_SEARCH_QUERY_LENGTH` | 1,000 chars | Slack API limit |
| `MAX_FILE_SIZE` | 300 MB | Slack API limit |
| `MAX_LIMIT` | 1,000 | Pagination limit |
| `DEFAULT_LIMIT` | 100 | Default for list commands |

## Slack API Notes

- **User IDs**: `U` + 8-12 alphanumeric chars (e.g., `U12345678`)
- **Channel resolution**: API accepts names directly (`#general`, `general`, `C123...`)
- **Rate limits**: SDK `RateLimitErrorRetryHandler(max_retry_count=3)` handles automatically
- **Email lookup**: `users.lookupByEmail` is faster than paginating `users.list`
- **Thread replies**: Need parent message `ts`, not the reply's `ts`

## Testing

See [tests/README.md](tests/README.md) for testing conventions, mocking patterns, and mock response examples.

Key points:
- Mock `slack_cli.client.WebClient` and `slack_cli.client.load_token`
- Use valid-length user IDs in tests (9+ chars, e.g., `U12345678`)
- Pagination mocks need `response_metadata.next_cursor` and `.get()` method on response

## Adding New Commands

1. Create command file in `slack_cli/commands/newcmd.py`
2. Follow existing pattern (Click group + subcommands)
3. Register in `slack_cli/cli.py`: `cli.add_command(newcmd.newcmd)`
4. Add tests in `tests/test_commands/test_newcmd.py`
5. Add validation callbacks in `validators.py` if needed

Reference: [Slack Python SDK Documentation](https://slack.dev/python-slack-sdk/)

## Common Gotchas

- Short user IDs (`U123`) don't match the ID pattern and fall back to slow username search
- `conversations_history` needs channel ID, but API resolves names automatically
- Always patch where imported (`slack_cli.client.load_token`), not where defined (`slack_cli.config.load_token`)
- Global client singleton must be reset between tests (handled by `conftest.py` fixture)
