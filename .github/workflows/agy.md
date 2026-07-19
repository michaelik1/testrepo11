---
name: Antigravity PR Agent
description: Antigravity agent invoked with the /agy command

on:
  slash_command:
    name: agy
    strategy: centralized
    events:
      - issues
      - issue_comment
      - pull_request
      - pull_request_comment

engine:
  id: antigravity
  command: /tmp/gh-aw/agent/agy-oauth-wrapper

permissions:
  contents: read
  issues: read
  pull-requests: read

network:
  allowed:
    - defaults
    - github

tools:
  github:
    toolsets:
      - repos
      - pull_requests

safe-outputs:
  add-comment:
    max: 1

pre-agent-steps:
  - name: Install Agentic Workflow Firewall
    run: |
      set -euo pipefail

      bash "$RUNNER_TEMP/gh-aw/actions/install_awf_binary.sh" v0.27.11

      command -v awf
      awf --version

  - name: Prepare Antigravity OAuth
    env:
      AGY_AUTH_B64: ${{ secrets.AGY_AUTH_B64 }}
    run: |
      set -euo pipefail

      AUTH_HOME="/tmp/gh-aw/agent/agy-home"
      INSTALL_HOME="/tmp/gh-aw/agent/agy-install"

      rm -rf "$AUTH_HOME" "$INSTALL_HOME"
      mkdir -p "$AUTH_HOME" "$INSTALL_HOME"

      printf '%s' "$AGY_AUTH_B64" \
        | base64 --decode \
        | tar -xzf - -C "$AUTH_HOME"

      test -s "$AUTH_HOME/.gemini/antigravity-cli/antigravity-oauth-token"
      test -s "$AUTH_HOME/.gemini/antigravity-cli/installation_id"

      chmod 700 "$AUTH_HOME/.gemini"
      chmod 700 "$AUTH_HOME/.gemini/antigravity-cli"
      chmod 600 "$AUTH_HOME/.gemini/antigravity-cli/antigravity-oauth-token"

      curl -fsSL https://antigravity.google/cli/install.sh \
        | HOME="$INSTALL_HOME" bash

      cat > /tmp/gh-aw/agent/agy-oauth-wrapper <<'EOF'
      #!/usr/bin/env bash
      set -euo pipefail

      export HOME="/tmp/gh-aw/agent/agy-home"
      export PATH="/tmp/gh-aw/agent/agy-install/.local/bin:$PATH"

      unset ANTIGRAVITY_API_KEY
      unset GEMINI_API_KEY
      unset ANTIGRAVITY_API_BASE_URL
      unset GEMINI_API_BASE_URL

      exec /tmp/gh-aw/agent/agy-install/.local/bin/agy "$@"
      EOF

      chmod 700 /tmp/gh-aw/agent/agy-oauth-wrapper
      /tmp/gh-aw/agent/agy-oauth-wrapper --version

timeout-minutes: 20
---

# Antigravity PR Agent

Follow the request written after the `/agy` command.

When the command is used on a pull request:

1. Inspect the pull request description, changed files, diff, and relevant surrounding code.
2. Read project instructions such as `AGENTS.md` when present.
3. Focus on confirmed bugs, security vulnerabilities, regressions, data loss, authorization errors, race conditions, and broken error handling.
4. Do not report purely cosmetic style issues.
5. Include exact file paths and line numbers for confirmed findings.
6. Publish the final result as a comment on the triggering pull request or issue.

Never print, read, disclose, or include credentials, OAuth files, environment variables, or workflow secrets in the response.