# Handoff: rtk PR #1873 — pnpm script rewrite with vitest streaming

Context for the agent picking this up on a new machine. Read this first.

## Environment rules (important)

- **`gh` CLI is logged in as the work account `ben-smith-atg`. Do NOT use `gh`** in this repo — it would act as the work account. Use git over SSH only.
- `origin` = `git@github.com-personal:ben-w-smith/rtk.git` (personal fork, SSH alias `github.com-personal` in `~/.ssh/config`).
- `upstream` = `git@github.com:rtk-ai/rtk.git`.
- For GitHub metadata without `gh`, use anonymous REST reads (public repo, 60 req/hr):
  `curl -s "https://api.github.com/repos/rtk-ai/rtk/pulls/1873"`.
- Any PR comments must be posted manually via the GitHub web UI as `ben-w-smith`.

## Where things stand

- PR: https://github.com/rtk-ai/rtk/pull/1873 (`feat(pnpm): hook-based script rewrite with vitest streaming`), open, base `develop`, CLA signed, no reviews/comments yet.
- Branch `feat/pnpm-scripts` is pushed and current. History: PR work → `96e6991` merge of upstream/develop (2026-07-17) → `8b28864` fix commit from the adversarial-review pass.
- **CI is gated**: every push puts the CI workflow into `action_required` — a maintainer must click "Approve and run" (first-time-contributor gate). This has NOT been requested yet.
- Repo social norms (from reading merged fork PRs): maintainers are `aeppling` (merges most) and `KuSh` (lead reviewer), owner `pszymkowiak`. Nobody cold-pings; post a plain comment first, mention a maintainer only if 1-2 weeks pass with no response.

## Next actions (in order)

1. **Finish the remaining review findings below** (at minimum #1 docs — repo policy requires it).
2. Run the pre-commit gate: `cargo fmt --all --check && cargo clippy --all-targets && cargo test` (see "Known test quirk" below).
3. Commit + push (updates the PR, re-triggers `action_required`).
4. Post this comment on the PR via web UI:

> Hi — friendly bump on this PR when a maintainer has a moment.
>
> CI is currently in the `action_required` state and needs someone to click **"Approve and run"** (standard gate for first-time fork contributors). Could you approve the workflow so the checks run?
>
> The branch is up to date with `develop` and conflict-free. Happy to make any changes once CI or review feedback comes in. Thanks!

## Done in the review-fix pass (commit 8b28864)

Verified adversarial review found 5 blockers; all fixed and independently verified:
1. `--filter` was silently dropped for `pnpm run` — now threaded through (`main.rs:1811`, `pnpm_cmd.rs` run_script).
2. `VitestStreamFilter` emitted lines without `\n` (output glued into one line) — fixed.
3. `TEST_FILE_RESULT` required `❯` but vitest prints passed files with `✓` — now symbol-aware `✓❯↓` matching; `saturating_sub` on parsed-count arithmetic.
4. Dash-builtins (`patch-commit`, `patch-remove`, `self-update`, …) were wrongly rewritten to `rtk pnpm run` — now routed to passthrough in `rules.rs`.
5. `VitestStreamFilter` had zero tests and savings tests hit a dead buffered path — 11 streaming tests added; dead arm bails to fallback. Plus 8 rewrite-rule tests, 1 clap parse test.

## Remaining review findings (fix these)

Ordered roughly by severity. File:line refs are as of 8b28864 — re-locate before editing.

1. **Docs gap (repo policy blocker)** — CONTRIBUTING.md's Documentation table requires: `src/cmds/js/README.md` (pnpm run routing + new js→rust dep on `runner::extract_test_summary`), root `README.md` (~line 206, Package Managers list — add `rtk pnpm run <script>`), `docs/usage/FEATURES.md` (~797-807, claims unrecognized pnpm subcommands pass through — now wrong for `run`).
2. **Bare `rtk pnpm run` regression** — clap requires `<SCRIPT>` (`main.rs` ~988), so `pnpm run` (lists scripts, a common discovery idiom) now exits 2 instead of passing through. Fix: `Option<String>` → `run_passthrough` on `None`.
3. **Summary regex misses skipped/todo** — `RE_SUMMARY_FILES` (`pnpm_cmd.rs` ~821) doesn't match `Test Files  2 passed | 1 skipped (3)` → green suite reports `PASS (0) | 0 suites`. Add optional skipped/todo segments.
4. **Playwright route drops failure diagnostics** (`pnpm_cmd.rs` ~758-766) — without `--reporter=json`, failures degrade to counts-only and stdout (where playwright prints errors) is replaced. On failure, fall back to raw/stripped.
5. **Substring routing can fabricate success** (`detect_tool` ~632-651) — `contains("tsc")` matches `"build": "tsc && vite build"`; `filter_tsc_output` prints "compilation completed" on unrecognized input → failed build can report success. Route on resolved first command token (strip `cross-env`/`npx`/`pnpm exec`); require positive format recognition on failure path.
6. **`never_worse` guard missing in `run_script`** buffered paths (~1139-1205) — siblings cap filtered+hint at raw; small inputs can emit more than raw.
7. **Echo suppression deletes user output** — `PNPM_CMD_ECHO`/`SCRIPT_ECHO` (~809, 866-874) drop any `console.log("> foo")` line. Only suppress actual pnpm echo/header lines.
8. **`in_failure_detail` never resets** (~930-960) — a `×` line permanently flips detail mode; later lines consume the 30-line budget spuriously. Reset on file-result/summary lines.
9. **Non-hermetic test** — `test_package_scripts_load_returns_none_without_package_json` (~1481) depends on no ancestor of CWD having a package.json. Refactor `load()` to take a path; test with tempdirs.

Minors: script-defined `--reporter=` breaks the stream filter (skip streaming when present); `package.json` parsed eagerly on every `pnpm run` (lazy-load after static routes); buffered success path discards stderr; caps don't use `CAP_*` from `src/core/truncate.rs`; dead `skip_env` param; stale module doc comment (claims `--reporter=json` injection, refs "PR #232"); unanchored alternation (`rules.rs:69`) means scripts like `run:test`/`i-foo` never rewrite; multi `--filter` flags don't rewrite (safe gap, tested); quoted filter values don't rewrite (safe gap, tested).

## Known limitations (accepted, tested as safe gaps)

- Multiple `--filter` flags → no rewrite (passthrough).
- `--filter "quoted value"` → no rewrite (passthrough).
- From a workspace root, `PackageScripts::load()` only reads CWD's package.json, so with `--filter` the tool detection may miss the filtered package's scripts → buffered passthrough (works, just unfiltered).

## How to verify changes

```bash
cargo fmt --all --check && cargo clippy --all-targets   # clippy: zero warnings
cargo test --bin rtk                                    # unit tests
cargo build
./target/debug/rtk hook check -- pnpm test:micro        # => rtk pnpm run test:micro
./target/debug/rtk hook check -- pnpm self-update       # => rtk pnpm self-update (NOT run)
./target/debug/rtk hook check -- pnpm run-script t:u    # => rtk pnpm run-script t:u
./target/debug/rtk hook check -- pnpm --filter p t:u    # => rtk pnpm --filter p run t:u
./target/debug/rtk -v pnpm --filter p run test:unit     # verbose shows filter reaches pnpm
```

**Known test quirk**: 2 tests in `hooks::rewrite_cmd::tests::unattestable_passthrough` fail under a real HOME because `~/.claude/settings.json` grants `Bash(git *)`. Unrelated to this branch. To run clean:

```bash
FAKEHOME=$(mktemp -d); HOME=$FAKEHOME RUSTUP_HOME=/Users/b.smith/.rustup \
  CARGO_HOME=/Users/b.smith/.cargo PATH=/Users/b.smith/.cargo/bin:$PATH \
  cargo test --bin rtk; rm -rf $FAKEHOME
```

(Adjust RUSTUP_HOME/CARGO_HOME on the new machine — `rustup show home` tells you.)
