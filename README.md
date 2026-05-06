# bouncer-slack

[Bouncer](https://github.com/jkylling/bouncer) API specs for the
[Slack Web API](https://docs.slack.dev/reference/methods).

## Installing

From a bouncer data directory:

```sh
bouncer apis add github.com/jkylling/bouncer-slack
```

## What's in the box

| File                    | Service                  | Notes                                                          |
|-------------------------|--------------------------|----------------------------------------------------------------|
| `apis/slack.yaml`       | Slack Web API            | 6 metas (conversation, user, bot, file, team, identity), 67 actions across conversations, chat, users, team, reactions, pins, bookmarks, files, views, bots, emoji, auth. |
| `apis/slack-files.yaml` | Slack file-transfer edge | No local metas (cross-references `slack.identity` and `slack.file`); 2 actions: `upload_file_chunk` (POST `/upload/v1/{token}`) and `download_file` (GET `/files-pri/{team_id}-{file_id}/{filename}`). |

## Authentication

Depending on the user, Slack has multiple tokens available:

### Bot tokens (`xoxb-` / `xoxp-`)

The straightforward case — a single bearer:

```sh
bouncer issue-token \
  --subject my-agent \
  --access-token "$SLACK_BOT_TOKEN" \
  --ttl 240h
```

The proxy forwards it as `Authorization: Bearer xoxb-…`. No
additional headers needed.

### Browser-session tokens (`xoxc-` + `d=xoxd-…`)

A browser session is a *pair*: the workspace-scoped `xoxc-` token
lives in `localStorage.localConfig_v2.teams[<team_id>].token` (or in form bodies of any
`*.slack.com/api/...` POST), and the `d=xoxd-…` cookie marks the
session as alive (you find these in developer tools > Application > Storage). Slack rejects requests that have only one. The edge also returns `invalid_auth` if `Origin` / `Referer` don't
match the web client.

Issue a single token carrying all four:

```sh
bouncer issue-token \
  --subject my-agent \
  --access-token "$SLACK_XOXC_TOKEN" \
  --header "Cookie=d=$SLACK_XOXD_VALUE" \
  --header "Origin=https://app.slack.com" \
  --header "Referer=https://app.slack.com/" \
  --ttl 240h
```

### Sanity check

If you have a working browser session, this curl call (no proxy)
should return `{"ok":true,...}`:

```sh
curl -s https://slack.com/api/auth.test \
  -H "Authorization: Bearer xoxc-…" \
  -H "Cookie: d=xoxd-…" \
  -H "Origin: https://app.slack.com" \
  -d ""
```

## Calling the proxy

### Curl
```sh
curl -H "Authorization: Bearer $BOUNCER_JWT" \
     -H "Content-Type: application/json" \
     -d '{"channel":"C0123456789","text":"hello"}' \
     http://localhost:8080/api/chat.postMessage
```

### agent-slack CLI

The [agent-slack CLI](https://github.com/stablyai/agent-slack) hard-codes
`https://slack.com` as the API host, so the path-prefix shape above
doesn't apply — point bouncer at it via `HTTPS_PROXY` and let MITM
mode terminate TLS:

```sh
# 1. Trust bouncer's MITM CA (one-time bootstrap).
curl -fsS http://localhost:8080/_api/ca.crt -o bouncer-mitm-ca.crt
export SSL_CERT_FILE=$PWD/bouncer-mitm-ca.crt

# 2. Issue a bouncer JWT once (any of the patterns above) and hand it
#    to agent-slack as the Slack token.
export SLACK_BOT_TOKEN=$(bouncer issue-token \
    --subject my-agent --access-token "$REAL_SLACK_TOKEN" --ttl 240h)

# 3. Point HTTPS_PROXY at bouncer.
export HTTPS_PROXY=http://localhost:8080
agent-slack ...
```

## Slack — policy patterns

Placeholders in `<UPPER_SNAKE>` are operator-supplied. The signals
these patterns lean on come from three of the bundle's metas:

- **`conversation`** — resolved via `conversations.info`. Carries
  `is_private`, `is_im`, `is_mpim`, `is_general`, `creator`,
  and (for IMs) the resolved `creator_user`.
- **`identity`** — resolved via `auth.test`. Carries `bot_id` (set
  on bot tokens) and `user_id` (the calling principal).
- **`team`** — resolved via `team.info`. Carries `id` (the
  workspace).
- **`file`** — resolved via `files.info`. Carries `is_external`.



### Post messages only to one channel

Resolve the channel through the `conversation` meta. One
`conversations.info` side fetch per request — costlier than a body
match, but Slack accepts channel *names* (`#general`) as well as
ids on `chat.postMessage`, and the meta normalises every form to
the canonical `id` before the predicate runs. The fetched
conversation is also shared with any other policy that touches the
meta on the same request.

```yaml
action: action.name == "post_message"
condition: |
  conversation.id == "<CHANNEL_ID>"
result: permit
```

### Read history in public channels only

Refuses private channels, DMs, and MPIMs. Triggers one
`conversations.info` side fetch per request. Meta fields exposed via
`?` in the YAML come through as the natural type or `null` (missing
and JSON-null are treated identically), so a missing `is_private`
fails closed naturally — `null == false` is false:

```yaml
action: |
  action.name in ["get_conversation_history", "get_conversation_replies"]
condition: |
  conversation.is_private == false
  && conversation.is_im == false
  && conversation.is_mpim == false
result: permit
```

### DM only one specific user

For an agent that should only converse with one operator. The `creator`
on an IM channel is the other party from the bot's perspective:

```yaml
action: action.name == "post_message"
condition: |
  conversation.is_im == true
  && conversation.creator == "<USER_ID>"
result: permit
```

### Lock to the expected workspace

Belt-and-braces deny when the token's workspace doesn't match
`<TEAM_ID>`. Useful for a bot installed in multiple workspaces where
only one is the live target. One `team.info` fetch per request,
shared across every policy that inspects the team meta:

```yaml
action: action.name != ""
condition: |
  team.id != "<TEAM_ID>"
result: deny
```

### Only read files uploaded by some users

Permit `get_file` (which fronts `files.info`) when the uploader's
user id is in an allowlist. The file meta is keyed by the same id
the action consumes, so the body match and the meta match agree on
the same file the upstream resolves.

```yaml
action: action.name == "get_file"
condition: |
  file.user in ["<USER_ID_1>", "<USER_ID_2>"]
result: permit
```

To gate the byte download too, pair this with a `slack_files`
policy on `download_file` (see below) — `files.info` returns the
metadata, but the bytes come from `files.slack.com` and are gated
separately.

### Only read files from the last 24 hours

Slack's `file.created` is Unix seconds; bouncer's `timestamp_seconds`
helper casts it to a `Timestamp` so CEL duration arithmetic works
against the request-scoped `now`:

```yaml
action: action.name == "get_file"
condition: |
  timestamp_seconds(file.created) > now - duration("24h")
result: permit
```

`now` is fixed once per request and shared with every other policy
on the same evaluation, so two policies inspecting `now` can't
straddle a clock boundary.

### Don't DM users outside `<ALLOWED_DOMAIN>`

Shared / external workspaces can route DMs to guests; this guards
against accidentally leaking content to them. Resolves the recipient
through the IM-channel's `creator_user`. Methods like `endsWith`
don't apply to null, so the chain guards email's presence first:

```yaml
action: action.name == "post_message"
condition: |
  conversation.is_im == true
  && (conversation.creator_user.email == null
      || !conversation.creator_user.email.endsWith("@<ALLOWED_DOMAIN>"))
result: deny
```

### Never make external files world-readable

External files (uploaded to / shared from another workspace) going
public is almost always a leak:

```yaml
action: action.name == "share_public_file_url"
condition: |
  file.is_external == true
result: deny
```

### List only files I uploaded

Permit `files.list` when the `user` query field matches the token's
own user id. Reading `identity.user_id` rather than hard-coding a
subject means the policy is portable. The body field stays optional
(`request.body.?user.orValue("")`); the meta field surfaces as a
plain string-or-null:

```yaml
action: action.name == "list_files"
condition: |
  request.body.?user.orValue("") == identity.user_id
result: permit
```

## `slack_files` — policy patterns

The `slack_files` API covers `files.slack.com`. It declares no metas
of its own; its actions cross-reference `slack.identity` and
`slack.file` from `slack.yaml`. Because the policy's API container
is `slack_files`, those metas have to be referenced by their
**fully qualified** names — `slack.identity.user_id`, not
`identity.user_id` (which would resolve to the non-existent
`slack_files.identity`). Same applies to constructor literals in
the YAML's `binds`.

### Download files I uploaded

Permit `download_file` when the file's uploader is the calling user.
Triggers one `files.info` and one `auth.test` per request, both
shared with any other policy on the same request:

```yaml
api: slack_files
action: action.name == "download_file"
condition: |
  slack.file.user == slack.identity.user_id
result: permit
```

### Pass through pre-signed upload URLs

The upload URL was already authorised by `files.getUploadURLExternal`
(which lives on `slack.com/api` and is gated by policies on the
`slack` API). The chunk upload to `files.slack.com` is just bytes on
the wire, so an unconditional permit is the typical shape:

```yaml
api: slack_files
action: action.name == "upload_file_chunk"
condition: "true"
result: permit
```


### Placeholders

| Placeholder        | Meaning                                            |
|--------------------|----------------------------------------------------|
| `<CHANNEL_ID>`     | Slack channel id (e.g. `C0123456789`).             |
| `<USER_ID>`        | Slack user id (e.g. `U0123456789`).                |
| `<TEAM_ID>`        | Slack workspace id (e.g. `T0123456789`).           |
| `<ALLOWED_DOMAIN>` | Workspace email domain (e.g. `example.com`).       |

## License

Apache License 2.0 — see [LICENSE](LICENSE).
