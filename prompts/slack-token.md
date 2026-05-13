Stage Slack credentials for CLIs on this machine. Bouncer's
`get_slack_token` MCP tool returns a bouncer-issued bearer that any
Slack-aware CLI or `curl` can use; the proxy unwraps it to your real
Slack token on every upstream call.

# 0. Skip if already staged
Run:

    test -f "${BOUNCER_HOME:-$HOME/.config/bouncer}/slack-token"

If exit code is 0 the credentials are already staged. Skip to step 3
(usage hint, per-project). Otherwise continue.

# 1. Fetch the bouncer-issued token
Call the MCP tool `get_slack_token`. The response has:

    {
      "service": "slack",
      "access_token": "<encrypted-bearer>",
      "credential_path": "~/.config/bouncer/slack-token",
      "credential_mode": "0600",
      "file_template": "{{ .AccessToken }}",
      "env": {
        "SLACK_TOKEN": "{{ .AccessToken }}",
        "SLACK_API_TOKEN": "{{ .AccessToken }}"
      }
    }

# 2. Write the credentials file
Render `file_template` with the response's `access_token` — Slack's
template is just the bare token, no JSON envelope. Write to
`credential_path` with mode `0600`.

# 3. Append the per-service subsection
Append the following block to the bouncer fragment in the project's
instruction file (CLAUDE.md / .cursorrules / AGENTS.md / etc.). If a
`### slack` heading already exists below `## bouncer`, skip.

    ### slack
    Source the slack token from [[ .CredentialPath ]] when invoking
    Slack-aware CLIs. Example:

        SLACK_TOKEN=$(cat [[ .CredentialPath ]]) \
          bouncer-wrap curl -H "Authorization: Bearer $SLACK_TOKEN" \
          https://slack.com/api/conversations.list

# Done
Tell the user:
    "Slack credentials staged. Try: bouncer-wrap curl https://slack.com/api/auth.test"
