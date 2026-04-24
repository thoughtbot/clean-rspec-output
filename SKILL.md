---
name: clean-rspec-output
description: Systematically eliminate unexpected output (warnings, stray puts/logs, deprecation notices) from a Ruby on Rails RSpec test suite, one issue at a time, with per-fix verification and commits. Use this skill whenever the user wants to clean up, silence, fix, or investigate noisy output from `bundle exec rspec`, mentions warnings or stray logs appearing during test runs, or asks to get their RSpec suite running "clean". Also use when a user says things like "my test output is noisy", "there's a bunch of warnings when I run specs", or "help me clean up the RSpec output" - even if they don't explicitly name this skill.
---

# Clean Up Unexpected RSpec Output

A procedural workflow for identifying and eliminating unexpected output from a Rails RSpec suite, one issue at a time, verifying each fix in isolation and committing as you go.

## When to use this

The user has an RSpec suite that produces unexpected output during test runs. This is typically a mix of:

- Ruby warnings (e.g. `warning: already initialized constant`, `warning: method redefined`)
- Gem deprecation notices
- Stray `puts`, `p`, `pp`, or `Rails.logger` calls left in application or test code
- ActiveRecord or ActionView deprecation warnings
- Output from third-party libraries that should be silenced in the test environment

The goal is a suite that runs with no unexpected output so that when something new appears, it's immediately visible.

## Before you start

Ask the user which git branch this work should happen on, and explain why: the skill iterates through each unexpected output one at a time and creates a separate commit per fix, so that individual fixes can be reviewed or cherry-picked independently. Do not assume a branch name. If the branch does not exist, create it from the current branch. If it does exist, check it out. You can suggest a branch name eg. fix-unexpected-test-ouput

If the user has told you to ignore specific output patterns (e.g. a deprecation they plan to fix separately), note them and skip them when you encounter them. Do not invent patterns to ignore on your own.

## Tool usage

Prefer native file and search tools over Bash for reading and searching log files:

- Use **Read** with `offset` and `limit` to view slices of a file, not `sed`, `head`, or `tail`.
- Use **Grep** to search within files, not Bash `grep`.

Read and Grep do this natively without permission prompts, so the workflow stays uninterrupted and is faster as a result. Bash is still the right tool for running the suite, git operations, and anything that genuinely needs a shell. Reading and searching captured output is not one of those cases.

## Fix scope: what counts as a fix

Fixes may change **how** a test runs. Fixes must not change **what** it covers.

A fix preserves every one of these:

- The assertions (`expect(...)` calls, their matchers, and the values passed to them)
- The branches the test exercises (loops, conditionals, locale/environment/role switches, multiple data variations)
- The setup data that drives those branches

If a change removes an assertion, collapses multiple variations into one, or narrows the scenarios the test walks through, it is not a fix. It is a change to the test, and it belongs in a separate, intentional commit, not in output cleanup.

When the unexpected output comes from code that the test genuinely needs to exercise, the fix belongs in the application code, in a helper, or in the test configuration. Not in the test itself.

### Changes to tests that ARE legitimate

- Updating a deprecated RSpec API (e.g. replacing `expect_any_instance_of` with an equivalent modern idiom), as long as the assertion semantics are preserved.
- Updating setup that uses a deprecated framework API (e.g. `update_attributes` to `update`), assertions untouched.
- Replacing an inline `puts`/`pp` that was never load-bearing with nothing at all.

### Anti-pattern: simplifying a test until the noise stops

A subtler variant of this anti-pattern is **keeping the branch structure but substituting the values passed to assertions**. If a branch produces noise because the test data is incomplete (e.g. a factory missing a required attribute), and the fix makes the assertion check a different value instead — one that happens not to produce noise — the branch still runs but no longer verifies what it was written to verify. This is the same loss of coverage as deleting the branch, just harder to see in a diff.

When test data is the root cause, the fix belongs in the test data.

If a test exercises multiple locales, roles, or data variations, and one of them produces unexpected output, do not delete the branches. The branches are the coverage.

**Wrong:**

```ruby
# Before:
I18n.with_locale(:en) do
  expect(page).to have_content(product_name)
end

I18n.with_locale(:pt) do
  expect(page).to have_content(product_name)
end

# After (wrong, "simpler"):
expect(page).to have_content(product_name)
```

The "simpler" version no longer verifies Portuguese locale rendering at all. The English/Portuguese split was the point of the test. If `I18n.with_locale(:pt)` produced noise, silence whatever emitted it, do not stop testing Portuguese.

**Right:** trace the noise to its source (application code, a view helper, a gem), fix it there, and leave the assertions intact.

## The workflow

### 1. Capture the full output

Before running the suite, create a task list using `TaskCreate` to track the full workflow. Create these tasks in order:

1. Run plain RSpec suite
2. Run documentation RSpec suite
3. Fix all unexpected output *(subtasks to be added after scanning)*
4. Final suite verification

Do not mark any task complete until you have actually finished it. The task list is the source of truth for what remains — do not consider the workflow done until all four tasks are marked complete.

Run the full suite first without any special formatting. The default progress format is compact enough that unexpected output stands out clearly against the dots and test summary:

```bash
bundle exec rspec 2>&1 | tee tmp/rspec-output.log
```

**Wait for this command to finish completely before running the next one. Do not run these two commands in parallel.** Both commands use the same shared test database, and concurrent runs will corrupt state and produce unreliable output.

This is the file you scan for unexpected output. Documentation format is too noisy to scan for strays - the test descriptions themselves become visual clutter.

Then run the suite a second time with documentation format:

```bash
bundle exec rspec --format documentation 2>&1 | tee tmp/rspec-output-doc.log
```

Wait for this run to finish before doing anything else. Do not start fixing issues while it is running. You need the documentation log to locate the source test for each issue, and without it you cannot complete the verification loop in step 3. Running fixes in parallel with the documentation capture means working without the data the fix depends on.

The documentation format prints the describe/context/it hierarchy inline, which lets you locate the exact test immediately before or after the stray output. The plain log (`tmp/rspec-output.log`) is the definitive source for triage in step 2 — it is the only log you should scan for unexpected output. Do not scan the documentation log for additional unexpected output; use it only to locate the specific test that produced output you have already identified from the plain log.

If the user has indicated the suite takes a long time, run both in the background and poll for completion rather than blocking the terminal, but do not start step 2 until both logs are on disk. Tell the user you are running in the background and will report back.

### 2. Scan for unexpected output

Read the plain log (`tmp/rspec-output.log`) and identify each distinct unexpected output. Group by likely root cause where possible, but plan to fix them one at a time.

Skip any patterns the user has flagged as "ignore for now".

Once you have the complete list of issues, update task 3 to add one subtask per issue, using the stray output text as the description (e.g. "Fix: `warning: already initialized constant Foo`"). The subtask list is now the definitive picture of remaining work — every issue must have a subtask, and no issue should be fixed without one.

### 3. For each issue, follow the verification loop

This loop is the core of the skill. Do not skip steps and do not reorder them. In particular, do not start planning or applying a fix until you have completed step 2 and seen the unexpected output appear in a single scoped command. Thinking ahead about the fix while still locating the source test is the most common way this workflow goes wrong.

1. **Locate the source test.** Use the documentation-format log (`tmp/rspec-output-doc.log`). In this format, each test description is its own line, so stray output appears as lines that don't match the indented description pattern, typically sandwiched between two test descriptions.

   Use these tools in this order:

   - **Grep** (not Bash `grep`) to find the exact line number of the stray output. Use anchored patterns to avoid false matches against test descriptions:
     - Exact single-line match: pattern `^true$` for a line that is literally `true`
     - Starts-with match: pattern `^Branch2 ID:` for a line beginning with a known prefix
     - For deprecation or warning text, match on the distinctive prefix: `^\[DEPRECATION\]` or `^warning:`
   - **Read** (not Bash `sed`/`head`/`tail`) with `offset` and `limit` to view the surrounding lines. A window of roughly 10 lines before and 5 after is usually enough to see the nearest test description.

   The test description immediately preceding the stray output is almost always the one that produced it. If the output appears before any test description (e.g. during boot), it came from `spec_helper.rb`, `rails_helper.rb`, or an eagerly loaded initializer.

   If the output comes from a setup hook or application code, the nearest test is still useful as a reproduction case.

2. **Reproduce in isolation. This is a hard gate.** Run just that one test:
   ```bash
   bundle exec rspec spec/path/to/spec.rb:LINE
   ```
   You must see the unexpected output in this single-test run before you are allowed to propose, plan, or apply any fix. No exceptions. The point of the fix is to make this output stop appearing in this exact command, so without the baseline you cannot verify anything.

   If the output does not appear, the output may depend on test ordering or another test's state. Widen the scope (run the full file, then the directory) until you can reproduce it reliably with a single command. That command, whatever its scope, becomes the reproduction case and the verification case in step 4.

   If you cannot reproduce the output with any scoped command short of the full suite, stop and tell the user. Do not attempt a fix against the full suite as the only reproduction, the feedback loop is too slow to verify cleanly.

   Only once you have a reliable, scoped reproduction may you proceed to step 3.

3. **Categorize and fix.**
   - **Warnings** (Ruby warnings, deprecation notices from gems or Rails): resolve the underlying cause. For `already initialized constant`, find the duplicate definition. For deprecations, update to the non-deprecated API. Suppressing warnings is a last resort and should be called out explicitly to the user.
   - **Stray puts / p / pp / Rails.logger calls**: apply the most idiomatic fix without asking. In almost all cases this means deleting the stray call - it was left over from debugging. Note that `pp` or `p` of a boolean, integer, or short string often prints as a single-word line (`true`, `false`, a number), which is why the anchored regex patterns in step 1 matter - unanchored searches for `true` will match dozens of test descriptions. Exceptions: if the call looks intentional (inside a rake task, a generator, a CLI), leave it and flag it to the user. If a test genuinely needs to assert on logger output, refactor to use `expect(Rails.logger).to receive(...)` instead of letting output leak.
   - **Output from third-party code**: silence at the test configuration level (e.g. in `spec/spec_helper.rb` or `spec/rails_helper.rb`), scoped as narrowly as possible. Prefer fixing the caller over globally silencing.

4. **Verify the fix.** Re-run the same isolated test:
   ```bash
   bundle exec rspec spec/path/to/spec.rb:LINE
   ```
   Confirm the output is gone. If it is not, the fix is wrong or incomplete - iterate before moving on.

5. **Simplify the fix.** Run the `/simplify` skill, scoped to the fix itself (the helper you added, the config change you made, the deprecated call you updated). This is a required step, not optional. Do not let `/simplify` touch the surrounding test code, assertions, or branches that were not part of the fix. If `/simplify` proposes collapsing or removing assertions, reject those suggestions, they violate the fix scope rule above.

6. **Check the diff.** Before committing, review the diff and ask: does this change only *how* the test runs, or has it changed *what* it covers? If any assertion has been removed, any branch collapsed, or any data variation dropped, back out and rethink the fix, the noise source is elsewhere.

7. **Commit.** Use the commit message format below unless the user has already told you to use a different format. Confirm the user is on the branch they specified at the start, then commit. One fix per commit, do not batch. After committing, mark that issue's subtask in task 3 complete.

8. **Move on.** Return to step 3 with the next issue.

### 4. Final verification

When you believe all issues are resolved, re-run the full suite without the documentation format and confirm the output is clean (aside from any user-specified ignore patterns). If new unexpected output appears that was masked by earlier issues, loop back through step 3 for each.

Stop when the full suite runs without unexpected output. Mark the final verification task (task 4) complete.

Once complete, direct the user to review each commit to ensure that the changes are correct.

## Commit messages

Use this format unless the user has already specified a preferred format:

- Title line
- `Before,` followed by current state context
- Explanation of the problem (the `Before` and problem may be one paragraph if they are closely tied, or two paragraphs if separating them reads more clearly)
- `Now,` followed by what changed and the impact

Avoid overexplaining individual changes. Prefer to explain the overall problem being solved rather than re-explaining what can be seen in the diff.

**Example:**

```
Suppress Puma cluster-mode warnings in test environment

When Capybara starts a Puma server for JS feature specs, Puma loads
config/puma.rb and runs in single mode (no workers). The
before_worker_boot and after_worker_boot hooks defined there are
cluster-mode-only, so Puma emits warnings about them never executing.

Now, those hooks are skipped when RACK_ENV=test. They remain active
in all other environments where Puma runs in cluster mode as intended.
```

## What this skill does not do

- It does not address test failures, only unexpected output from passing tests. If tests are failing, resolve those first using standard debugging.
- It does not rewrite the suite for speed or structure.
- It does not add new tests, only fixes noise in existing ones.
