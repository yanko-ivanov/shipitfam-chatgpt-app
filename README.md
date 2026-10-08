# ShipItFam for ChatGPT and Codex

Drive your [ShipItFam](https://shipitfam.com) AI dev crew from ChatGPT or Codex. ShipItFam is a human-in-the-loop AI dev team: you describe a goal, a crew of specialist agents plans and builds it, and nothing risky or final happens without your say-so. This app lets your assistant list your projects, create new ones from a starter, start missions, show you what the crew is waiting on, carry your answers back, open previews and ship.

It is the same MCP server the [Claude plugin](https://github.com/yanko-ivanov/shipitfam-claude-plugin) uses: `https://shipitfam.com/mcp`.

## What is in here

- **A Codex plugin** (`plugins/shipitfam`, with `.codex-plugin/plugin.json`): the ShipItFam MCP server, plus a skill (`shipitfam`) that teaches the assistant how to drive ShipItFam well. It also carries the interface metadata (name, icons, default prompts) the plugin directory shows.
- **A Codex marketplace** (`.agents/plugins/marketplace.json`) so the plugin installs straight from this repo.
- **A ChatGPT app directory packet** (`submission/`): app metadata, test cases and reviewer notes for submitting ShipItFam to the ChatGPT app directory.

## Install

You need a ShipItFam account. Create one at [shipitfam.com](https://shipitfam.com). You do not need a project yet: once your assistant is connected, ask it to create one ("Create a ShipItFam project called Landing from the blank starter"). It lists the starters with `project_starter_list` and creates the project with `project_create`. Missions run on projects created from a starter.

### ChatGPT (custom connector, developer mode)

1. In ChatGPT, open Settings, then Apps & Connectors, then Advanced settings, and turn on **Developer mode**.
2. Create a connector. Name it `ShipItFam`, set the MCP server URL to `https://shipitfam.com/mcp`, and choose **OAuth** for authentication. Leave client ID and secret empty: ShipItFam registers ChatGPT automatically.
3. Click Connect. A ShipItFam page opens: log in, click Allow, and you are back in ChatGPT.
4. In a new chat, add the ShipItFam connector and ask: "What does my ShipItFam crew need from me right now?"

The labels in ChatGPT's settings move around between releases. If one is missing, look for the option to add a custom MCP connector.

### Codex

```
codex plugin marketplace add yanko-ivanov/shipitfam-chatgpt-app
```

Then open the plugin directory in Codex (`/plugins`) and install **ShipItFam**. The first time it connects, Codex opens ShipItFam's sign-in page in your browser: log in and click Allow.

## How sign-in works

ShipItFam is an OAuth server. There are no tokens to copy and nothing to paste into a config file.

1. The client registers itself with ShipItFam (dynamic client registration).
2. A browser opens on ShipItFam's consent page. You log in and click Allow (PKCE with S256, no client secret).
3. The client receives an access token that lasts 24 hours and renews itself with a refresh token.

Every connected app shows up in ShipItFam under **MCP access**. Revoke one there and its refresh stops working.

Do not add an `Authorization` header to the MCP config. A static header switches OAuth off.

## Using it

- "What can I start a ShipItFam project from?"
- "Create a ShipItFam project called Landing from the blank starter."
- "What projects do I have on ShipItFam?"
- "Start a mission on my landing page project: add a pricing section with three tiers."
- "Approve the plan for the pricing mission."
- "Show me the preview and ship it if it looks right."

The crew runs on your own Claude login, which you set up in the ShipItFam app. If a step fails with "Claude is not logged in on this project", sign in to Claude in the app and retry.

## What your assistant is allowed to do

The server exposes 78 tools, and every one declares `title`, `readOnlyHint`, `destructiveHint` and `openWorldHint`:

| Class | Tools | Hints |
|---|---|---|
| Read-only | 27 (every `*_list` and `*_get`, `helm_feed`, `preview_get`, `preview_get_main`, `project_box_diagnostic`, `project_provisioning_status`, `session_inspect_all`, `context_pack_get`, `changelog_list`) | `readOnlyHint` true |
| Write | 37 (creates and edits, for example `project_create`, `mission_create`, `project_update`, `project_settings_set`) | `readOnlyHint` false, `destructiveHint` false |
| Destructive | 14 (every `*_delete`, `session_stop`, `session_cleanup`, `session_force_resume`, `approval_reject`, `project_repo_credential_clear`, `project_invite_revoke`, and `action`) | `destructiveHint` true |

`openWorldHint` is true on exactly two tools, because the call itself reaches outside your ShipItFam account: `mcp_server_verify` (calls the URL you registered for an MCP server) and `action` (on a "Ship it?" card it pushes the mission branch to your git remote). It is false on the other 76. `action` is destructive because it is how a mission card is answered, cancelled or shipped, so ChatGPT may ask you to confirm it.

A connected app cannot change your billing or manage your Claude logins: those stay behind your own ShipItFam session, so your assistant sends you to the app for them.

## Layout

```
.agents/plugins/marketplace.json            Codex marketplace (policy: AVAILABLE, auth: ON_INSTALL)
plugins/shipitfam/
  .codex-plugin/plugin.json                 plugin manifest and interface metadata
  .mcp.json                                 the remote MCP server (streamable-http), no static headers
  skills/shipitfam/SKILL.md                 how to drive ShipItFam
  assets/logo.png                           1024x1024 logo
  assets/composer-icon.png                  512x512 composer icon
submission/
  app-metadata.md                           listing text, URLs, auth summary
  test-cases.md                             5 positive (read-only, or creating their own state) and 3 negative test cases
  reviewer-notes.md                         OAuth facts and reviewer instructions
```

## Links

- Website: https://shipitfam.com
- Privacy: https://shipitfam.com/privacy.html
- Terms: https://shipitfam.com/terms.html
- Support: https://shipitfam.com/contact.html

## License

MIT, see [LICENSE](LICENSE).
