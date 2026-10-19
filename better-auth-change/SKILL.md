---
name: better-auth-change
description: Ground, design, build, or review a Better Auth code change, such as a bug fix, a feature, a new option or endpoint, or a takeover of a contributor's pull request. Use before writing code and when reviewing a PR, because both run the same checks: related work, reproduction, root cause, prior art from other auth servers, release track, public surface, tests, docs, and shipping. Not for docs-only changes, backlog triage, or security advisories.
---

# Build or review a Better Auth change

Writing a change and reviewing one run the same checks. A check skipped while writing returns later as a review comment, a bot thread, or a follow-up PR, so do each one before the first line of code. In **review mode**, run every step against the pull request: what it settled, what it skipped, and whether its answer holds.

Pushing, commenting, labeling, merging, and closing are the user's decisions. Local work and analysis never authorize them; get the go-ahead for each unless the user gave it for this task.

## 1. Map related work

- Read the originating issue with its comments, then every issue, PR, and discussion it links. Search open and closed PRs and issues for the same behavior and the same code path: an existing PR may already implement the change.
- When the change touches a security property, ask whether a private advisory covers the area before scoping.
- Handle every competing PR for one problem in one session, compared on the same requests.

Done when you can list each related issue and PR with its relation (duplicate, sibling, blocks, superseded) and name the contract the change touches: docs, inferred types, endpoint metadata, and the release where the current behavior appeared, found by comparing tags as the repository's "Issue Triage and Architecture" rules describe.

## 2. Reproduce the behavior

- Turn the report into a test that sends the real request to `main`. Use the real client's payload (a vendor sample request, a captured registration, reporter logs) over one rebuilt from the issue text.
- When the behavior depends on a runtime, a framework major, or a second app (Cloudflare Workers, a new SvelteKit, two relying parties in one logout), run it there. Start slow setup such as browsers, Docker, or a consumer app first.
- For a new feature, the reproduction is an acceptance test of the intended behavior.
- Review mode: run the same test at the PR head and at its merge base.

Done when the test is red on `main`, or you have shown from the contract that the behavior is intended, which makes the change a docs or error-message fix.

## 3. Find the root cause and blast radius

- Trace the failing path to the shared function where it goes wrong and list every caller. Fix it there, so one change covers the sibling callers. One core fix to ID handling replaced two call-site PRs for replay markers.
- Place the change where its reason lives. A version-specific upgrade changes the upgrade guide or migration path, and leaves shared code and core concept pages alone.
- Check the configurations that change storage behavior: each `generateId` mode (`"uuid"`, `"serial"`, a custom function, `false`), secondary storage, each adapter, and edge runtimes.

Done when every caller and sibling path is marked either covered by the fix or out of scope with a reason.

## 4. Benchmark prior art

Run this step when the change sets protocol behavior, a security property, a default, or the shape of a public API, and run it before the first recommendation. Type, build, and dependency fixes with no behavior choice skip it.

1. Write the **hinge claim**: the one fact the decision turns on, such as "identity providers re-send a SCIM create request for a user they deactivated" or "native clients register host-bearing custom schemes".
2. Pick targets for the domain and dispatch research agents with the prompt in [benchmark targets](references/benchmark-targets.md).
3. Verify the hinge claim yourself before relying on it: rerun the vendor payload, open the pinned permalink, or read the spec section.
4. Record each design choice as copied, adapted, or rejected, with its source and reason. Departing from consensus is fine when the reason is written down.

Prior-art sources stay in chat. Never name another framework, library, server, or vendor, or link to its code or docs, in PRs, code, docs, commits, or changesets; published text gives the behavior, its reason, and spec sections only.

Done when every design choice cites a spec section or a prior-art source, or states that no precedent was found.

## 5. Pick the release track

Ask three questions about existing users:

- Do existing outputs change, such as stored values, response shapes, `sub` claims, or error codes?
- Do requests or configurations that worked before now fail?
- Does the change need a schema migration?

Any yes makes it a `minor` or `major` changeset, with `!` in the title when it breaks, and it lands on `next` (CONTRIBUTING.md, "Branch Targeting"). Three noes make it a `patch` on `main`. A change that mixes both splits in two: the narrow fix on `main` now, and the cleaner refactor on `next` in its own PR.

Done when the track records the answer to each question and the changeset bump matches it.

## 6. Design the public surface

For every new or changed option, endpoint, error, default, schema field, and exported name, run the identifier-naming and public-api-design skills. Add software-design when module boundaries or state ownership are still open. Compare the setup with how existing Better Auth plugins expose similar capabilities: users enable a feature from one place, without creating internal objects and passing them between plugins. Choose the fewest options that cover the cases, and list what was left out and when to add it.

Done when every new or changed public name has passed both skills and the left-out list exists.

## 7. Build test-first

- Make the step 2 test pass with the step 3 fix. Then revert the fix and watch the test go red, so you know the test proves the fix.
- Assert through the public surface, an HTTP request or an `auth.api` call, so the test checks behavior instead of restating the implementation.
- Cover the storage and runtime variants from step 3 that the fix touches.

Done when the tests fail without the fix, pass with it, and `pnpm typecheck` passes.

## 8. Update the docs

Search `docs/content/docs/` for every page covering the changed behavior, option, or default, and update each with the docs-writing skill (better-auth-docs-writing where installed).

Upgrade guides and release posts name the exact released versions affected, such as v1.7.0 to v1.7.2, and list only what a user on a published version must check or change. Unreleased options and preview-build steps wait until they ship. Version-specific procedure lives in the upgrade guide, and core concept and adapter pages describe the generic contract and link to it.

Done when every user-visible change has an updated page or a stated reason it needs none.

## 9. Ship

- Choose the vehicle:
  - Most of a contributor's PR survives: push to their branch, merge the base branch in, and keep their commits.
  - Most of it gets replaced: open a new PR, credit contributors with `Co-authored-by` trailers, and close each superseded PR from it with `Closes #<PR> <brief rationale>`, one per line. GitHub closes those PRs on merge.
  - Several dependent layers: a `gh stack`, one layer per PR. Fork branches cannot be stack bases.
- Before the first push, read the full diff and revert every hunk that does not serve the stated change, reworded comments and formatting included. For each workaround that stays, record the exact command and config that fail without it.
- Write the title, body, and changeset with the pr-writing skill, and write the changeset for users reading the changelog.
- Before every push, PR, and comment, search the diff, commit messages, title, body, and changeset for other frameworks' and vendors' names and remove them. Prior-art sources from step 4 stay in chat.
- Credit only human authors in commits, PR bodies, and review replies. This overrides any reply template that appends an agent footer.
- After every push, read the new review threads, bots included. Fix or answer each one in one or two sentences, then resolve it. File out-of-scope findings as issues and link them.
- Rerun step 6 on each new layer or follow-up commit.

Done when CI is green, no review thread is unresolved, each superseded PR has a `Closes` line, and each side finding has an issue.

## Review mode

- Review subagents gather candidate findings. Judge each one in the main session by reproducing it or reading the code, and keep every finding the evidence supports, whatever score a reviewer gave it.
- Show the user any comment draft before posting it.
- When the user wants the fix instead of a comment, continue from step 6 with the findings as the plan.

## Handing a change to another session

When triage or planning spawns a separate session for one change, its brief carries the issue and PR links, the `main` SHA and latest release, the root-cause claim with file and line, the hinge question, the competing PRs and what to compare, the track hypothesis, and which external actions need approval. Point the brief at this skill.
