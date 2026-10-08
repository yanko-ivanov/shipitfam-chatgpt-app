# ShipItFam for ChatGPT and Codex

Run your [ShipItFam](https://shipitfam.com) AI dev crew from ChatGPT or Codex. ShipItFam is a human-in-the-loop AI dev team: you describe a goal, a crew of specialist agents plans and builds it, and nothing risky or final happens without your say-so. This app lets your assistant do the whole job of running it with you: queue missions, show what the crew is waiting on, carry your approvals and answers back, follow up on finished work, steer a mission that is running, change how much the crew asks you, and open previews.

It is the same MCP server the [Claude plugin](https://github.com/yanko-ivanov/shipitfam-claude-plugin) uses: `https://shipitfam.com/mcp`.

## What is in here

- **A Codex plugin** (`plugins/shipitfam`, with `.codex-plugin/plugin.json`): the ShipItFam MCP server, plus a skill (`shipitfam`) that teaches the assistant how to drive ShipItFam well. It also carries the interface metadata (name, icons, default prompts) the plugin directory shows.
- **A Codex marketplace** (`.agents/plugins/marketplace.json`) so the plugin installs straight from this repo.
- **A ChatGPT app directory packet** (`submission/`): app metadata, test cases and reviewer notes for submitting ShipItFam to the ChatGPT app directory.

## Install

You need a ShipItFam account. Create one at [shipitfam.com](https://shipitfam.com). You do not need a project yet: once your assistant is connected, ask it to create one ("Create a ShipItFam project called Landing from the blank starter"). It lists the starters with `project_starter_list` and creates the project with `project_create`. A new project runs missions when it is created from a starter (a project you already have may run them too: `core_v2` in `project_list` says).

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

Say it in plain words:

- "What does my ShipItFam crew need from me right now?"
- "Queue a mission on my landing page project: add a pricing section with three tiers."
- "Show me the plan for the pricing mission. If it looks right, approve it."
- "How is the pricing mission going? What is the crew doing right now?"
- "The pricing mission is done. Follow up: make the middle tier the highlighted one."
- "Stop pausing me on every plan for the landing page project, but keep asking before risky commands."
- "Show me the preview of the pricing mission."
- "What can I start a ShipItFam project from?" then "Create one called Landing from the blank starter."

The loop behind it: `inbox` shows what the crew waits on, `request_get` reads one request in full and `request_answer` carries your decision back. `mission_create` queues a mission (they run oldest first, one at a time per crew), `mission_list`, `mission_get`, `job_log` and `step_diff` let your assistant watch it, `mission_comment` follows up or steers, and `project_settings_set` sets how much the crew asks you. Your assistant shows you a plan, a risky command or a "Ship it?" and waits for your yes before it approves, allows or ships; skipping a step and cancelling a mission need your yes as well. If you say so up front ("approve the plan for the pricing mission"), that is your yes.

The crew runs on your own Claude login, which you set up in the ShipItFam app. If a step fails with "Claude is not logged in on this project", your assistant offers to reuse a login you already have working on another project, or sends you to the app to sign in.

## What your assistant is allowed to do

The server exposes 57 tools, and every one declares `title`, `readOnlyHint`, `destructiveHint` and `openWorldHint`:

| Class | Tools | Hints |
|---|---|---|
| Read-only | 21 (every `*_list` and `*_get`, `inbox`, `request_get`, `job_log`, `step_diff`, `project_starter_list`, `routine_run_list`, `preview_get_main`, `project_box_diagnostic`, `project_provisioning_status`) | `readOnlyHint` true |
| Write | 26 (creates and edits, for example `project_create`, `mission_create`, `mission_comment`, `preview_pick`, `project_settings_set`, `project_wake`) | `readOnlyHint` false, `destructiveHint` false |
| Destructive | 10 (every `*_delete`, `mission_cancel`, `project_repo_credential_clear`, `project_invite_revoke`, and `request_answer`) | `destructiveHint` true |

`openWorldHint` is true on exactly two tools, because the call itself reaches outside your ShipItFam account: `mcp_server_verify` (calls the URL you registered for an MCP server) and `request_answer` (on a "Ship it?" request it pushes the mission branch to your git remote). It is false on the other 55. `request_answer` is destructive because it is how a request is answered: approving a plan, allowing a command, cancelling a mission and shipping all go through it, so ChatGPT may ask you to confirm it.

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
  test-cases.md                             5 positive (read-only, or creating their own state) and 3 negative test cases, hosted tools only
  reviewer-notes.md                         OAuth facts and reviewer instructions
```

## Links

- Website: https://shipitfam.com
- Privacy: https://shipitfam.com/privacy.html
- Terms: https://shipitfam.com/terms.html
- Support: https://shipitfam.com/contact.html

## License

MIT, see [LICENSE](LICENSE).
