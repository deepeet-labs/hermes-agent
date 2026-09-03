# Scoped Heimdall Discord incident intake

Heimdall alerts are opt-in and fail closed. They do not use the global
`DISCORD_ALLOW_BOTS` behavior. Configure the exact Discord webhook identity in
the Discord platform configuration:

```yaml
discord:
  heimdall_incident_intake:
    guild_id: "your-guild-snowflake"
    channel_id: "your-error-alert-channel-snowflake"
    webhook_id: "the-incoming-webhook-snowflake"
    author_id: "the-webhook-author-snowflake"
    toolsets: ["safe"]
```

All five fields are required. The alert must be a default/reply Discord event
from the configured webhook author in the exact guild and channel, with no
message text or attachments and at least one embed. Any identity mismatch,
malformed configuration, ordinary bot, text-bearing message, or attachment is
denied in that scoped channel; it cannot fall back to `DISCORD_ALLOW_BOTS`.

Only embed title, description, and field text is accepted as untrusted evidence.
Control characters are removed and the resulting input is capped at 4,000
characters. A single `Incident ID` field (normalized and limited to 256
characters) is preferred for the SHA-256 fingerprint, so changing prose,
timestamps, or occurrence counts reuses the incident thread. Duplicate,
ambiguous, or malformed `Incident ID` fields are denied. Legacy alerts without
an ID fingerprint all normalized evidence except only the explicit `발생 횟수:`
description line and `발생 횟수` / `연속 실패` fields; other text remains part of
the identity. The fingerprint's exact guild/channel/webhook scope, thread ID,
lifecycle, and updated time are persisted as bounded, profile-local state under
`~/.hermes/gateway/discord_heimdall_incidents.json` (or that profile's Hermes
home). Writes are atomic. On restart, Hermes fetches the saved Discord thread
and verifies its live guild and parent channel before reuse; a missing or
wrong-parent thread is removed from state and the alert safely creates a new
thread. Corrupt state is fail-closed and is never overwritten automatically.
Claims are serialized by fingerprint and exact scope across adapters in one
event loop and across processes. Cross-process file-lock acquisition and
release run off the gateway event loop; the lock remains held through state
reread, thread resolution or creation, and state persistence. Lock errors or
unsupported platforms fail closed.
Each created or reused incident thread is marked as participated; newly created
threads also pre-seed message deduplication with `thread.id` to suppress
Discord's duplicate thread-starter event.
After a repeated alert is durably copied into the verified incident thread, its
standalone parent-channel webhook message is removed. This applies to recovery
alerts as well, so lifecycle updates remain in the original incident thread.
If the copy fails, the parent-channel message is retained rather than losing the
alert.
When the bot intentionally lacks Discord `Manage Messages`, set the secret
`DISCORD_HEIMDALL_WEBHOOK_URL` to the same admitted webhook. The adapter verifies
the URL's webhook ID and uses its credential only to remove that webhook's copied
parent post.

The bot principal receives only the explicit `toolsets` list intersected with
the platform's enabled toolsets. It never inherits an owner toolset or private
owner context, even when the general principal-toolset feature is disabled.

Leave this block absent to preserve existing human and generic Discord bot
behavior. Do not place IDs or behavioral settings in `.env`; use `config.yaml`.
