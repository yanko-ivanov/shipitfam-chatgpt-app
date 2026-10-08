# ChatGPT app directory: app metadata

Copy these fields into the OpenAI dashboard when submitting. Items marked TODO are the owner's.

## Listing

| Field | Value |
|---|---|
| App name | ShipItFam |
| Subtitle (30 characters or fewer) | Steer your AI dev crew |
| Category | Developer tools (Codex plugin category: Coding) |
| Developer name | Yanko Ivanov |
| Logo | `plugins/shipitfam/assets/logo.png` (1024 x 1024 PNG) |
| Brand color | `#FF6B6B` (the ShipItFam call-to-action red) |
| Languages | English |

### Short description

Drive your ShipItFam AI dev crew from the conversation: create projects, start missions, answer the crew, review previews and ship.

### Description

ShipItFam is an AI dev team you keep on a leash. You describe a goal in plain language, a crew of specialist agents plans and builds it in your project's own workspace, and nothing risky or final happens without your approval.

Connect your ShipItFam account and ChatGPT can:

- list your projects and check on them,
- create a new project from a starter (an empty one, or a ready-made one for reporting, automation, content or coding),
- start a mission from a plain-language goal,
- show what the crew is waiting on (questions, plans to approve, risky commands, "Ship it?"),
- carry your decision back, only when you have made it,
- steer a running crew with a note, or follow up on finished work,
- open a mission's live preview and its pull request,
- manage routines, the Treasure map (project knowledge) and your team.

You stay in charge. Plans, risky commands and shipping always wait for your yes, and you can revoke ChatGPT's access at any time from MCP access in ShipItFam.

### Suggested starter prompts

1. What does my ShipItFam crew need from me right now?
2. Start a ShipItFam mission: add a pricing section to my landing page
3. Show me the preview of my latest ShipItFam mission and help me ship it

## URLs

| Field | URL |
|---|---|
| Website | https://shipitfam.com |
| Privacy policy | https://shipitfam.com/privacy.html |
| Terms of service | https://shipitfam.com/terms.html |
| Support | https://shipitfam.com/contact.html |
| MCP server | https://shipitfam.com/mcp |

The Codex package carries the same URLs in `plugins/shipitfam/.codex-plugin/plugin.json` under `interface` (`websiteURL`, `supportURL`, `privacyPolicyURL`, `termsOfServiceURL`).

## Connection and authentication

| Field | Value |
|---|---|
| Transport | Streamable HTTP |
| Authentication | OAuth 2.1 authorization code with PKCE (S256), public client |
| Client registration | Dynamic client registration (RFC 7591), `https://shipitfam.com/api/oauth/register` |
| Protected resource metadata | `https://shipitfam.com/.well-known/oauth-protected-resource/mcp` |
| Authorization server metadata | `https://shipitfam.com/.well-known/oauth-authorization-server` |
| Authorization endpoint | `https://shipitfam.com/api/oauth/authorize` |
| Token endpoint | `https://shipitfam.com/api/oauth/token` |
| Scope | `mcp` |
| ChatGPT redirect URI | `https://chatgpt.com/connector_platform_oauth_redirect` |

Full OAuth facts are in `reviewer-notes.md`.

## Tools and annotations

The hosted server exposes **78 tools**. Every one declares `title`, `readOnlyHint`, `destructiveHint` and `openWorldHint`, and 47 also declare `idempotentHint` (all 27 read-only tools, plus the writes and destructive calls where repeating the call changes nothing further). The server serves no tool that returns a credential.

| Class | Count | Annotations |
|---|---|---|
| Read-only | 27 | `readOnlyHint` true, `destructiveHint` false, `idempotentHint` true |
| Write | 37 | `readOnlyHint` false, `destructiveHint` false |
| Destructive | 14 | `readOnlyHint` false, `destructiveHint` true |

**Read-only (27):** `project_starter_list`, `project_list`, `project_get`, `project_box_diagnostic`, `component_list`, `project_member_list`, `project_invite_list`, `agent_type_list`, `project_agent_type_list`, `mcp_server_list`, `routine_list`, `routine_run_list`, `task_list`, `task_get`, `task_deliverable_get`, `helm_feed`, `project_provisioning_status`, `session_list`, `session_inspect_all`, `preview_get`, `preview_get_main`, `ship_log_list`, `changelog_list`, `context_pack_get`, `mission_list`, `mission_get`, `request_list`.

**Write (37):** `project_create`, `project_update`, `project_repo_credential_set`, `project_fleet_update`, `component_create`, `project_member_add`, `project_invite_create`, `agent_type_create`, `agent_type_update`, `agent_type_enable`, `agent_type_disable`, `mcp_server_create`, `mcp_server_update`, `mcp_server_verify`, `routine_create`, `routine_update`, `task_create`, `task_update`, `task_claim`, `task_ask`, `task_assign`, `task_complete`, `plan_artifact_attach`, `plan_finalize`, `plan_claim`, `update_post`, `update_ack`, `decision_request`, `decision_answer`, `approval_approve`, `approval_comment`, `project_provision`, `session_start`, `ship_log_add`, `ship_log_update`, `mission_create`, `project_settings_set`.

**Destructive (14):** `project_delete`, `project_repo_credential_clear`, `component_delete`, `project_invite_revoke`, `agent_type_delete`, `mcp_server_delete`, `routine_delete`, `approval_reject`, `session_stop`, `session_delete`, `session_force_resume`, `session_cleanup`, `ship_log_delete`, `action`. Each removes, stops, restarts, revokes or commits something. `action` is how the assistant answers a mission card (approve, answer, ship, cancel), so ChatGPT may show a confirmation before each of those calls. That is expected.

**Open world (`openWorldHint` true) on exactly 2 tools, false on the other 76:**

- `mcp_server_verify` (write): sends an MCP `initialize` request to the URL the user registered for an MCP server, with the headers they configured.
- `action` (destructive): on a "Ship it?" card its Ship action pushes the mission branch to the user's git remote.

Every other tool acts on the signed-in user's own ShipItFam account. Tools that only store configuration for an agent to use later (for example `mcp_server_create`) or that call ShipItFam's own infrastructure (for example `component_list`) are not open world.

The authoritative list is `TOOL_SPECS` in ShipItFam's control-mcp (`kind`, `idempotent`, `openWorld` per tool). Re-check this section against it before submitting.

## Data handling

- The server acts as the signed-in user and sees only that user's projects.
- It returns no server addresses, SSH details or credentials. Git credentials and MCP-server secrets are write-only and are never returned.
- Access tokens last 24 hours. The user can revoke a connected app at any time under MCP access in ShipItFam.
- A connected app cannot change billing or manage the user's Claude logins: those routes refuse the connected app's token and need the user's own ShipItFam session.

## Owner to-do before submitting

The submission requirements are on https://developers.openai.com/apps-sdk/deploy/submission. Re-read them before submitting: they change.

- TODO(owner): organization verification in the OpenAI dashboard.
- TODO(owner): domain verification. The dashboard gives a challenge token. Serve exactly that token, as plain text and nothing else, at `https://shipitfam.com/.well-known/openai-apps-challenge`. The nginx snippet in `deploy-mcp.sh` only routes `/.well-known/oauth-*` and `/.well-known/openid-configuration` to the MCP service, so this path needs its own file in the landing site or its own nginx location.
- TODO(owner): reviewer test account **without multi-factor authentication** (email and password only), entered in the dashboard's Review details form and kept available for later reviews. Details in `reviewer-notes.md`. Put it on the Free plan: starters are refused on a plan whose workspace is a pod (`409 starter_unavailable_on_substrate`), and the test cases create projects with `auto_provision` false so nothing is provisioned. Optionally create one starter project on it so cases P1 and P2 have something to list.
- TODO(owner): demo recording URL, a video walkthrough reviewers can open. OpenAI reads it from `extensions.com.openai.review.demo_recording_url` in the package manifest. When that `extensions.com.openai` object exists, OpenAI ignores `.codex-plugin/plugin.json`, so decide how to package it: either a root `plugin.json` that repeats the `interface` block (including `supportURL`) under `extensions.com.openai`, or the dashboard form if it offers the field. Record it on a provisioned project, because the test cases never run a crew: show create from a starter, start a mission, answer the crew, preview, ship.
- TODO(owner): screenshots of the app in use (the dashboard asks for specific sizes at submission time).
- TODO(owner): the privacy policy and terms pages currently carry placeholder copy pending legal review. Finish them before submitting: reviewers read them.
- TODO(owner): run every case in `test-cases.md` once against production with the reviewer account and compare the tool calls. The cases were written from the code, not from a live run.
- TODO(owner): after each review, delete the leftover "ShipItFam review" projects the cases create (they are unprovisioned, so they cost nothing).
- TODO(owner): release notes for the first version, and country targeting if it should not be everywhere. Both go under `extensions.com.openai.publication` (`release_notes`, `countries`) and are optional.
