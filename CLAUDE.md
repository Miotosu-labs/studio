# Miotosu software factory — context for Claude Code

This repo is `miotosu-labs/studio`. It holds the shared CI gates that every product repo calls, pinned by tag, and the factory tools in `factory/`. Only Tom changes it. No agent has access to it.

## How Tom wants to be written to

- Plain English, short sentences, active voice. One instruction per sentence (ASD-STE100 style).
- No jargon and no cryptic shorthand. Say exactly what to click or paste.
- Give paste blocks that work as they are. Say where to run them (Mac, Incus host, or inside the box).
- Tom is a product person, not a developer. He approves outcomes; he does not judge code.

## The model

Agents build. Tom only approves. Linear is the system of record.

| Place | Holds | Never holds |
| --- | --- | --- |
| Linear (workspace `miotosu`, team `THO`) | work: issues, delegation, status | code, agent rules, knowledge |
| GitHub org `miotosu-labs` | code, the gates, agent rules (`AGENTS.md`, `docs/spec/`) | work status |
| Notion | knowledge: how it works, why, what was rejected | work status, anything an agent reads |
| Slack | notifications only | anything Tom must find again |

## How a piece of work flows

1. Tom writes a one-line issue in Linear with labels `Plan` and `opencode`, and delegates it to **Cyrus AI Sandbox**.
2. The **planner** (Cyrus `scoper` mode) writes a one-page spec as a Linear document and 4–8 small issues. It delegates nothing.
3. Tom approves the list and delegates the independent issues together. The **builders** run in parallel, each in its own git worktree, and each opens one pull request.
4. The **reviewer** (Inspector, THO-12, not built yet) checks each pull request against its issue. It runs silent for the first ~20.
5. The gates decide. Since 7 Nov 2026 GitHub needs no human approval: a pull request with all five checks green can merge. Tom moves the issue to Done in Linear and Cyrus squash-merges. The pull request title becomes the commit.

Rules that matter are checks that block the merge. Instructions alone get ignored; that is why Bedrockfolio failed.

Tom approves **what** gets built (the issue) and **the plan** (the planner's issue list). He does not read code. That is why the gates must prove a feature was tested, not just that it compiles.

## Parts and their IDs

- **Cyrus**: self-hosted, in the unprivileged Incus container `cyrus-agent` on the host Incus-1. Linear webhooks arrive through a Cloudflare Tunnel that only lets Linear's published IPs through.
  - Config: `/home/cyrus/.cyrus/config.json`. Cyrus reloads it by itself when the file changes. Write it in place (not by rename).
  - Routing: by Linear project **name** (`projectKeys`). `The Pipeline` → `template`. `Bedrockfolio` → `product-bedrockfolio`. Renaming a Linear project breaks routing silently.
  - Modes per repo in `labelPrompts`: `scoper: ["Plan"]`, `orchestrator: ["Orchestrator"]`. Orchestrator runs sub-issues one after another, so it is not for speed.
  - `maxConcurrentSessions` caps parallel agents.
  - Git login: credential helper `/home/cyrus/.cyrus/git-credential-github-app.sh` mints a GitHub App token for each request. Nothing stored on disk.
  - Runner: OpenCode, pinned to v2.1.4 by the wrapper `/usr/local/bin/opencode`, log level `info`. Model: DeepSeek V4.1 Flash 1017 through OpenRouter, one key, $50/month cap. Claude Code is uninstalled on the box on purpose.
  - Service restarts on any exit. It was OOM-killed at 2 GB and then at 8 GB limits.
- **GitHub App "Cyrus Agent"**: App ID 5459986, installation 174947436. Permissions: checks read, contents write, metadata read, pull_requests write. Repos: `template` and `product-*` only. **Never add `studio`.**
- **Linear Code** GitHub app: also has code write. It must exclude `studio` too.
- **Org ruleset 25797579 `default-branch-protection`**: covers `template` and `product-*`. **Zero approvals** since 7 Nov 2026. Code owner review, last-push approval and unattributed-change approval are all off. Still active, still an empty bypass list, and all five checks still required. Unresolved review threads still block a merge: that is the one thing that can stall a pull request silently.
- **Tag ruleset on `studio`**: `v*` tags cannot move or be deleted. Current tag: `v6`.
- **Required checks**: `checks / gate-guard`, `checks / typecheck`, `checks / lint`, `checks / unit`, `checks / isolation`.
- **Gate files** (a pull request may change these or product code, never both): `.github/`, `e2e/`, `supabase/policies/`, `supabase/tests/`, `CODEOWNERS`, `perf-budget.json`, `AGENTS.md`.
- **Pull request title rule**: `^(feat|fix|docs|chore|refactor|test|perf)(\([a-z0-9-]+(,[a-z0-9-]+)*\))?!?: .+`. Squash-merge only.
- **PostHog**: EU cloud, project 319854, endpoint `https://eu.i.posthog.com`. OpenRouter Broadcast is set up but had delivered zero events at last check.
- **Vercel**: project `bedrockfolio-web`, team `team_Ng7M4k2q93LzbSdquS2iPQhk`. Runs a Next.js 15.5.15 with known security holes. To be paused.
- **Supabase**: the Bedrockfolio project is paused.
- **Linear team THO id**: `335f9d87-f084-4179-9436-2f3bcbee91b8`.
- **Notion pages**:
  - The Pipeline — current architecture: `403d7f1c-4a29-8180-895d-da2055067461`
  - Products database: `collection://3cab3cf5-f68a-4690-b11e-5c40dfee50e1`
  - PEGY Alerts product page: `40cd7f1c-4a29-8110-b96f-dbd0224b6344`

## Never do this

- Never put an org-owner GitHub login, a personal token or any admin credential on the Cyrus box. The agents run there.
- Never push to an agent's branch. That makes Tom the last pusher, and then he cannot approve.
- Never admin-merge, add a bypass actor, or merge with a red check. The review count is already zero on purpose; put it back to one before a product takes real paying users.
- Never move or delete a `studio` tag. Make a new one (`v5`, `v6` …) and bump each product's `ci.yml` in its own pull request.
- Never treat a gate as working until it has failed on purpose on a real pull request. "Configured" and "green" have both lied before.
- Never mix gate files and product code in one pull request.

## Tom's own gate pull requests

No longer a special case. Approvals are zero, so a gate pull request merges on green like any other. The old Evaluate-switch dance is gone.

## What the gates prove (studio v6)

Five required checks: `gate-guard`, `typecheck`, `lint`, `unit`, `isolation`.

`gate-guard` also holds three checks added on 7 Nov 2026, because `test:unit` runs `jest --passWithNoTests` and could pass with no tests at all:

1. A pull request that changes `src/**` must add or change a test. `*.d.ts` and `types.ts` are exempt.
2. Such a pull request must leave the repository holding at least one test file.
3. Every runtime dependency must appear in the `allowed-deps` input in the product's own `ci.yml`. Empty means none. This is how an agent is stopped from adding a package unnoticed; to allow one, Tom adds it to that list in a gate pull request.

All installs use `--ignore-scripts`, so a package cannot run code on the runner.

What the gates do **not** prove: that an answer is correct. An agent can write tests that confirm its own mistake. Only an end-to-end check against an expected output, or the Inspector, catches that.

## Product status (6 Nov 2026)

- **PEGY Alerts** (Linear project, repo `product-pegy`, not created yet): the small restart of Bedrockfolio. A scheduled GitHub Actions job that calculates PEGY = P/E ÷ (growth % + dividend yield %) and a few value metrics, and posts the cheap ones to Slack. No website, no database, no login.
- **Bedrockfolio** (`product-bedrockfolio`): paused, to be archived. Nothing gets deleted.

## After every change

- Update the Linear issue: what changed, what was verified.
- Update the Notion architecture page when how the factory works changes. Keep Notion free of status.
