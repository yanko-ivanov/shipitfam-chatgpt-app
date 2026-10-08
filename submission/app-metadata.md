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

Run your ShipItFam AI dev crew from the conversation: queue missions, approve plans, answer the crew, follow up, review previews and ship.

### Description

ShipItFam is an AI dev team you keep on a leash. You describe a goal in plain language, a crew of specialist agents plans and builds it in your project's own workspace, and nothing risky or final happens without your approval.

Connect your ShipItFam account and ChatGPT can run the whole loop with you:

- list your projects and check on them,
- create a new project from a starter (an empty one, or a ready-made one for reporting, automation, content or coding),
- queue missions from a plain-language goal (they run oldest first, one at a time per crew),
- show what the crew is waiting on (questions, plans to approve, risky commands, step reviews, "Ship it?", failed steps) and read each one in full,
- carry your decision back, only when you have made it: approve or revise a plan, answer a question, allow or deny a command, ship,
- watch a mission: its steps, what the crew is doing right now, the code a step changed,
- follow up on finished work, steer a running step, or leave the crew a note,
- set how much the crew asks you: plan approval, risky commands, a pause after every step, and whether it keeps working while a mission waits on you,
- open a mission's live preview and its pull request,
- manage routines and see the runs they fired, the Treasure map (project knowledge) and your team.

You stay in charge. Plans, risky commands and shipping always wait for your yes, and you can revoke ChatGPT's access at any time from MCP access in ShipItFam.

### Suggested starter prompts

1. What does my ShipItFam crew need from me right now?
2. Queue a ShipItFam mission: add a pricing section to my landing page
3. Follow up on my latest finished ShipItFam mission: make the headline shorter

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

Computed from the built control-mcp (`buildTools` with the default config: no v1 workflow tools, no credential tool), not typed by hand. The authoritative list is `TOOL_SPECS` in ShipItFam's control-mcp (`kind`, `idempotent`, `openWorld` per tool); re-check this section against it before submitting.

<!-- annotations:start -->
The hosted server exposes **57 tools**. Every one declares `title`, `readOnlyHint`, `destructiveHint` and `openWorldHint`, and 38 also declare `idempotentHint` (all 21 read-only tools, plus the writes and destructive calls where repeating the call changes nothing further). The server serves no tool that returns a credential, and no v1 workflow tool.

| Class | Count | Annotations |
|---|---|---|
| Read-only | 21 | `readOnlyHint` true, `destructiveHint` false, `idempotentHint` true |
| Write | 26 | `readOnlyHint` false, `destructiveHint` false |
| Destructive | 10 | `readOnlyHint` false, `destructiveHint` true |

**Read-only (21):** `project_starter_list`, `project_list`, `project_get`, `project_box_diagnostic`, `component_list`, `project_member_list`, `project_invite_list`, `agent_type_list`, `project_agent_type_list`, `mcp_server_list`, `routine_list`, `routine_run_list`, `project_provisioning_status`, `preview_get_main`, `ship_log_list`, `inbox`, `request_get`, `mission_list`, `mission_get`, `job_log`, `step_diff`.

**Write (26):** `project_create`, `project_update`, `project_repo_credential_set`, `project_fleet_update`, `component_create`, `project_member_add`, `project_invite_create`, `agent_type_create`, `agent_type_update`, `agent_type_enable`, `agent_type_disable`, `mcp_server_create`, `mcp_server_update`, `mcp_server_verify`, `routine_create`, `routine_update`, `project_provision`, `project_wake`, `ship_log_add`, `ship_log_update`, `request_use_login`, `mission_create`, `mission_comment`, `mission_dismiss`, `preview_pick`, `project_settings_set`.

**Destructive (10):** `project_delete`, `project_repo_credential_clear`, `component_delete`, `project_invite_revoke`, `agent_type_delete`, `mcp_server_delete`, `routine_delete`, `ship_log_delete`, `request_answer`, `mission_cancel`.

**Open world (`openWorldHint` true) on exactly 2 tools, false on the other 55:**

- `mcp_server_verify` (write)
- `request_answer` (destructive)
<!-- annotations:end -->

`request_answer` is how the assistant answers a request (approve a plan, allow a command, reply, skip, ship), so it is destructive and ChatGPT may show a confirmation before each of those calls. That is expected. Its "Ship it?" answer is the one place a call pushes work to the user's git remote, which is why it is open world. Every other tool acts on the signed-in user's own ShipItFam account. Tools that only store configuration for an agent to use later (for example `mcp_server_create`) or that call ShipItFam's own infrastructure (for example `component_list`) are not open world.

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
- TODO(owner): demo recording URL, a video walkthrough reviewers can open. OpenAI reads it from `extensions.com.openai.review.demo_recording_url` in the package manifest. When that `extensions.com.openai` object exists, OpenAI ignores `.codex-plugin/plugin.json`, so decide how to package it: either a root `plugin.json` that repeats the `interface` block (including `supportURL`) under `extensions.com.openai`, or the dashboard form if it offers the field. Record it on a provisioned project, because the test cases never run a crew: show create from a starter, queue a mission, approve the plan, answer a question, watch the log, set the settings, follow up, preview, ship.
- TODO(owner): screenshots of the app in use (the dashboard asks for specific sizes at submission time).
- TODO(owner): the privacy policy and terms pages currently carry placeholder copy pending legal review. Finish them before submitting: reviewers read them.
- TODO(owner): run every case in `test-cases.md` once against production with the reviewer account and compare the tool calls. The cases were written from the code, not from a live run, and the live smoke of the hosted catalog (doc 52, section 7.3) has not run yet.
- TODO(owner): `request_get`, `request_answer`, `request_use_login`, `job_log`, `step_diff`, `preview_pick` and `project_wake` need a mission that has reached a request or a step, which needs a running crew on the owner's Claude login. No test case can create that, so the demo recording must show them. Optionally keep one provisioned project with an open plan request on the reviewer account so a reviewer can try `request_get` and `request_answer` by hand.
- TODO(owner): after each review, delete the leftover "ShipItFam review" projects the cases create (they are unprovisioned, so they cost nothing).
- TODO(owner): release notes for the first version, and country targeting if it should not be everywhere. Both go under `extensions.com.openai.publication` (`release_notes`, `countries`) and are optional.
