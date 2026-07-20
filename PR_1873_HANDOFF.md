# Handoff: rtk PR #1873 — pnpm script rewrite with vitest streaming

Context for the agent picking this up on a new machine. Read this first.

## Environment rules (important)

- **`gh` CLI is logged in as the work account `ben-smith-atg`. Do NOT use `gh`** in this repo — it would act as the work account. Use git over SSH only.
- `origin` = `git@github.com-personal:ben-w-smith/rtk.git` (personal fork, SSH alias `github.com-personal` in `~/.ssh/config`).
- `upstream` = `git@github.com:rtk-ai/rtk.git`.
- For GitHub metadata without `gh`, use anonymous REST reads (public repo, 60 req/hr):
  `curl -s "https://api.github.com/repos/rtk-ai/rtk/pulls/1873"`.
- Any PR comments must be posted manually via the GitHub web UI as `ben-w-smith`.
- cmux note: shell `CMUX_WORKSPACE_ID`/`CMUX_SURFACE_ID` env vars can be stale — pass explicit refs to `cmux` commands (e.g. `--workspace workspace:N --surface surface:N`), don't rely on env defaults.

## Where things stand

- PR: https://github.com/rtk-ai/rtk/pull/1873, open, base `develop`, CLA signed, no reviews/comments yet.
- Branch `feat/pnpm-scripts` (all pushed): original PR work → `96e6991` upstream merge → `8b28864` review-fix pass 1 → `87b7569` review-fix pass 2 → `3e77207` docs.
- **CI is gated**: every push puts CI into `action_required` — a maintainer must click "Approve and run" (first-time-contributor gate). Approval has NOT been requested yet.
- Repo social norms (from merged fork PRs): maintainers `aeppling` (merges most) and `KuSh` (lead reviewer), owner `pszymkowiak`. Nobody cold-pings; plain comment first, mention a maintainer only after 1-2 weeks of silence.

## Next action (the only one left)

Post this comment on the PR via the GitHub web UI as `ben-w-smith`:

> Hi — friendly bump on this PR when a maintainer has a moment.
>
> CI is currently in the `action_required` state and needs someone to click **"Approve and run"** (standard gate for first-time fork contributors). Could you approve the workflow so the checks run?
>
> The branch is up to date with `develop` and conflict-free. Happy to make any changes once CI or review feedback comes in. Thanks!

Then respond to maintainer/CI feedback as it arrives.

## What was done (two adversarial-review fix passes)

Pass 1 (`8b28864`) — blockers: `--filter` threaded through `PnpmCommand::Run`; VitestStreamFilter newline contract; symbol-aware `✓❯↓` matching + `saturating_sub`; dash-builtins (`patch-commit` etc.) routed to passthrough; 11 streaming tests + 8 rule tests.

Pass 2 (`87b7569`) — majors + minors: bare `rtk pnpm run` passthrough (clap regression); first-token tool routing (no more substring false positives) + tsc/lint filters can't fabricate success on unrecognized input; echo suppression stops after RUN banner (user console.log survives); `in_failure_detail` resets; summary regexes accept skipped/todo; playwright keeps raw errors on failure; `never_worse` guard on buffered paths; lazy package.json load; hermetic tempdir tests; skip streaming when `--reporter` forced; `CAP_*` constants; anchored subcommand alternation (`run:test`/`i-foo` rewrite, `install-test`/`install-completion` protected); multiple `--filter` flags rewrite.

Docs (`3e77207`): `src/cmds/js/README.md` routing section, root `README.md` entry, `docs/usage/FEATURES.md` passthrough fix + table row.

## Remaining known gaps (accepted — fix only if a maintainer asks)

- Quoted filter values (`--filter "my pkg"`) don't rewrite (safe passthrough).
- From a workspace root, tool detection reads only CWD's package.json → `--filter` runs may take the buffered passthrough path (works, unfiltered).
- Playwright path has no E2E test (browser download too heavy); covered by transcript unit tests.
- `$ cmd` echoes on stderr still show in failure output (pre-existing `strip_pnpm_stderr` scope).
- `docs/usage/FEATURES.md` `~90%` savings figure for the run row is a judgment call (test floor is 60%; `60-90%` would be more defensible).
- `docs/usage/FEATURES.md` pre-existing `rtk pnpm build` row looks stale (no `Build` variant exists; predates this PR — leave for maintainers to confirm).
- Optional: `docs/guide/resources/what-rtk-covers.md:58` and `docs/guide/analytics/gain.md:101` coverage tables list only `pnpm list`/`outdated` — accurate but incomplete.

## How to verify changes

```bash
cargo fmt --all --check && cargo clippy --all-targets   # clippy: zero warnings
cargo test --bin rtk                                    # unit tests
cargo build
./target/debug/rtk hook check -- pnpm test:micro        # => rtk pnpm run test:micro
./target/debug/rtk hook check -- pnpm self-update       # => rtk pnpm self-update (NOT run)
./target/debug/rtk hook check -- pnpm run-script t:u    # => rtk pnpm run-script t:u
./target/debug/rtk hook check -- pnpm --filter p t:u    # => rtk pnpm --filter p run t:u
./target/debug/rtk hook check -- pnpm run:test          # => rtk pnpm run run:test
./target/debug/rtk hook check -- pnpm --filter a --filter b t:u  # => rewrites (multi-filter)
./target/debug/rtk -v pnpm --filter p run test:unit     # verbose shows filter reaches pnpm
```

**Known test quirk**: 2 tests in `hooks::rewrite_cmd::tests::unattestable_passthrough` fail under a real HOME because `~/.claude/settings.json` grants `Bash(git *)`. Unrelated to this branch. To run clean:

```bash
FAKEHOME=$(mktemp -d); HOME=$FAKEHOME RUSTUP_HOME=/Users/b.smith/.rustup \
  CARGO_HOME=/Users/b.smith/.cargo PATH=/Users/b.smith/.cargo/bin:$PATH \
  cargo test --bin rtk; rm -rf $FAKEHOME
```

(Adjust RUSTUP_HOME/CARGO_HOME on the new machine — `rustup show home` tells you.)
