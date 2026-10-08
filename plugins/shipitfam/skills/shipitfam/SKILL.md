---
name: shipitfam
description: Drive ShipItFam, an AI dev crew the user keeps on a leash, through its MCP tools. Use when the user wants to see their ShipItFam projects, create a project from a starter, start a mission (a goal the crew plans and builds), see what needs their answer on the Helm, approve a plan or a risky command, steer the crew with a note, open a mission's live preview, or ship finished work to their repo.
---

# Driving ShipItFam

ShipItFam is an AI dev team with a human on the trigger. The user states a goal, a crew of specialist agents plans and builds it in the project's own workspace, and the crew stops to ask before anything risky or final. You are the user's hands on the controls: read the board, relay what the crew needs in plain language, and carry the user's decisions back. You never make those decisions for them.

## The model in one screen

- **Project**: one codebase and its crew.
- **Mission**: one goal, run as ordered steps: `spec` (what and why) → `plan` (how) → `work` steps → `sync` with main → `ship`. A *quick* mission skips spec and plan and is a single `work` step. `sync` and `ship` exist only when the project's `push_mode` is not `off`.
- **Request**: a step either finishes or needs the user. It then blocks on a request: a question, a plan to approve, a risky command to allow, "Step N done. Continue?", "Ship it?", or a failure. The user's answer unblocks it. That is the whole loop.
- Words to use with the user: **Helm** (everything waiting on them), **Deck** (a project's missions: Needs you, In progress, Up next, Done), **Preview**, **Ship log** (history), **Treasure map** (project knowledge). A **crew** is a running agent session. A persona is a **Rank**, never a crew.

## Always start here

1. `project_list`. Never guess a `project_id`. Match a project the user named by name; if several fit or none was named, ask once.
2. Read `core_v2` on the project (`project_list` or `project_get`). `true`: use the mission tools below. `false`, or a mission tool answers `not_core_v2`: it is a legacy project, use the legacy flow at the end.
3. No project at all: see "Create a project". Only legacy projects: use the legacy flow for them, and offer to create a starter project for missions. Do not work around the `not_core_v2` refusal.

## Create a project

Missions run only on core-v2 projects, and a project is core v2 only when it is created from a starter. So creating one takes two calls:

1. `project_starter_list` (no arguments; the list is the same for every user). Each starter has `key`, `title`, `description`, `category` (`reporting`, `automation`, `content`, `coding`, `blank`), `suggested_integrations` and `first_goal_text`. Show the user the few that fit what they want, recommend one and say why. `blank` is the empty starter.
2. `project_create {name, starter_key, auto_provision?}` with a `key` from that list. Ask for the name if the user gave none. Create a project only when the user asked for one or said yes to your proposal.

After it:

- A starter project is local with no remote repo: its branch is `main` and `push_mode` is `off`. Never pass `repo_url`, or a `push_mode` other than `off`, together with a `starter_key`: both are refused. `repo_default_branch` is ignored.
- Read `provisioning_status` on the result. `provisioning`: the cloud workspace is being built, a few minutes; check `project_provisioning_status` sparingly and report the phase. `subscription_required`: the account has no active cloud-workspace plan (the Free plan, or a lapsed subscription), so the crew runs on the user's own machine (`npx shipitfam tentacle`) or the user picks a plan; both are set up in the ShipItFam app, not over MCP. `failed`: offer `project_provision` to retry. `none`: the project was created with `auto_provision: false` and has no workspace; pass `auto_provision: false` only when the user asks for that, and `project_provision` starts it later.
- `first_goal_text` is the starter's suggested first goal. Offer it as the first mission. Do not start it unasked.
- Errors, relayed in the server's own words. `409 Templates are not yet available on this plan's workspace type`: starters cannot be created on the user's current plan yet; say so and stop. A bare `404`: the key is not in the catalog, call `project_starter_list` again and pick a listed key. A bare `503`: the catalog is unavailable, try once more later. Do not fall back to a project without a starter to get around any of these.
- `project_create` without `starter_key` makes a **legacy** project: it can link an existing repo with `repo_url`, but it cannot run missions and every mission tool answers `not_core_v2` on it. Do that only when the user explicitly wants a legacy project, and say so before you call it.
- To ship a starter project's work to a git repo later, `project_update {project_id, repo_url}` links one. The first remote switches `push_mode` from `off` to `manual`, so each ship then waits for the user's "Ship it?". Do this only when the user asks. A private repo needs a git credential: ask the user to add it in the ShipItFam app rather than pasting secrets into chat. If they insist, call `project_repo_credential_set` and never repeat the secret back.
- The crew runs on the user's own Claude login, set up in the ShipItFam app, not over MCP. If a step fails with "Claude is not logged in on this project", give the user the card's `fix_login` link (the Claude sign-in screen), then retry. A connected app cannot change billing or manage the Claude login pool, so send the user to the app for those.

## Start a mission

`mission_create {project_id, title, text?, quick?, agent_type?}`

- `title` is the goal in the user's own words (up to 200 characters). `text` is extra detail (up to 20,000). Do not pad, rewrite or over-specify it.
- The default full path may come back with questions and a plan to approve. Use `quick: true` with an `agent_type` only for a small, well-defined change when the user wants no planning. `project_agent_type_list` shows the valid values.
- Tell the user what happens next ("the crew writes a spec first and may ask you questions") and how to check on it. If the project has no running crew yet (`provisioning_status` is not `ready`, or the Deck shows a `notice`), say so: the mission waits until a crew picks it up.

## Watch the board

- `request_list {project_id?}`: every open request on the user's core-v2 projects (the Helm). Legacy projects never appear here: their waits are in `helm_feed`. So an empty result means no core-v2 request is waiting. Before telling the user nothing needs them, also call `helm_feed` if any project has `core_v2` false. Empty never means the crew is idle.
- `mission_list {project_id}`: the Deck, grouped `needs_you`, `in_progress`, `up_next`, `done`, plus the project's preview bar and a crew `notice` (provisioning, no crew, crew asleep, crew offline). A notice explains why nothing is moving: relay it and its actions.
- `mission_get {mission_id}`: one mission in full: headline, steps and their states, the brief, the open request with its actions, `pr_url`.
- Missions run for minutes to hours and there is no push. Check when the user asks or after a sensible gap, say what you are looking for, and never poll in a tight loop.
- Report in plain language: what needs the user first, then what is running, then what finished. Do not dump raw JSON.

## Answer requests

Every open request carries its own `actions` (`id`, `label`, optional `input` and `confirm`). Read them from `mission_get` (`request.actions`) and call:

`action {mission_id, action_id, text?, answers?, step_id?}`

- Show the request in full first: the questions with their options, the plan's brief, steps and critique, the exact command and why it is needed, the step summary.
- `text` carries a note or revision. `answers` is `{question_id: answer}` for a question request.
- Common ids: `approve` (Approve plan, Allow, Continue, Ship, Retry), `reply` (Answer, Revise, Deny with note, Retry with note; send `text` where the card requires it), `skip` (Skip step, Don't ship, Cancel mission), and `revise` (on "Ship it?" only). Do not hard-code them: use an id the card lists. An unknown id returns the ids on offer.
- Decide only what the user decided. Never approve a plan, allow a risky command, or ship on your own, however obvious it looks, unless the user said so ("approve it", "yes, ship"). Never invent answers to the crew's questions: ask the user.
- An action with a `confirm` (cancelling) needs the user's explicit yes.
- A link action (open PR, open preview, live log, fix login, diff) does nothing and returns a link: hand it to the user. A link starting `app:` is a screen inside the ShipItFam app (Claude sign-in, a live log, a diff): name the screen and tell them to open it there. Any other link opens in a browser.
- "Already handled" means the request was answered elsewhere. Re-read with `mission_get` and report the current state.
- After every action, re-read `mission_get` and tell the user what changed: the step running again, the next request, the mission done.

## Steer the crew

- A note to the crew, or a follow-up on a finished mission: `action` with `comment` and `text` (add `step_id` to target one step). A follow-up on a done mission reopens it.
- `cancel` (confirm first) stops a mission, keeping its branch. On a card whose open request already offers "Cancel mission", that is the `skip` id instead: use whichever id the card lists. `dismiss` clears a closed mission off the Deck.
- Project dials, only when the user asks: `project_settings_set {project_id, approve_plan, approve_risky, pause_after_step, keep_working}`. Defaults: `approve_plan` on, `approve_risky` on, `pause_after_step` off, `keep_working` off (on means the crew moves to the next mission while one waits for an answer). Say what a change does before making it. Never turn `approve_risky` off unprompted.

## Review and ship

- **Preview**: the Deck's preview bar has a `status` (`ready`, `starting`, `failed`, `offline`, `stopped`, `none`), a `url` when ready, and the candidate missions. A project shows one mission's preview at a time. A mission's `open_preview` action returns the link, `preview_this` switches the preview to another mission. When `failed`, relay the one-line error and offer the "ask the crew to fix the preview" action if the card has it. `preview_get_main` returns a project's persistent main-branch preview URL.
- **Code**: `open_pr` returns the PR link once there is one (`pr_url`), `view_diff` a step's diff.
- **Ship**: with `push_mode` `manual` the last card is "Ship it?": Ship pushes the mission branch and opens the PR, Revise (with text) sends the crew back to fix and then asks again, Don't ship finishes without a PR. With `off` nothing is pushed and the work stays in the project's workspace; say so, and offer to link a repo (`project_update` with `repo_url`) only if the user wants that. Nothing lands in the user's repo without their yes.

## Everything else in the toolbox

Use these when asked, not on your own initiative.

- Treasure map (decisions, gotchas, direction): `ship_log_list`, `ship_log_add`, `ship_log_update`, `ship_log_delete`; `context_pack_get` for a briefing. Ship log history: `changelog_list`.
- Routines (recurring work): `routine_create`, `routine_update`, `routine_delete`, `routine_list`, `routine_run_list`.
- Team (lead only): `project_member_list`, `project_member_add`, `project_invite_create`, `project_invite_list`, `project_invite_revoke`.
- Ranks and tools for crews: `project_agent_type_list`, `agent_type_*`, `mcp_server_*` (headers and env are write-only and never shown).
- A stuck workspace: `project_box_diagnostic`, `project_fleet_update`, `session_inspect_all`, `session_cleanup`.
- Running the crew on the user's own machine (`npx shipitfam tentacle`) is set up from the ShipItFam app, not over MCP.

## Legacy projects (`core_v2` false)

No missions here: work is tasks and sessions.

- Create work with `task_create {project_id, title, description}`, then wake the crew with `session_start {project_id}`.
- `helm_feed {project_id?}` lists what waits on the user on legacy projects (questions, checkpoints, stuck crews, unshipped branches). It skips core-v2 projects. Answer a decision with `decision_answer {task_id, answer}`. Approve, reject or steer a plan or task with `approval_approve`, `approval_reject`, `approval_comment {task_id, project_id, comment}`.
- `task_list` and `task_get` for status, `preview_get_main` for the live URL, `session_list` and `session_stop` for crews.
- The same rule holds: the user decides.

## Rules everywhere

- Tools that remove or stop things (`*_delete`, `session_stop`, `session_cleanup`, `session_delete`, `session_force_resume`, `approval_reject`, `project_repo_credential_clear`, `project_invite_revoke`, cancelling a mission) run only on the user's explicit instruction naming the thing. State exactly what will go first.
- Two tools reach outside the user's ShipItFam account, and are flagged that way to the client: `mcp_server_verify` calls the URL the user registered (sending the headers they configured), and `action` on a "Ship it?" card pushes the branch to the user's git remote. Run them only on the user's say-so.
- Never put a secret in chat, and never echo one a tool returns.
- Long lists are truncated near 40,000 characters. Filter (a `project_id`, a status) and tell the user the result was cut.
- Relay the server's own sentence on errors. Do not retry blindly, and do not work around a refusal.
