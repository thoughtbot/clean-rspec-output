# Clean RSpec Output skill

A [Claude Code][claude-code] skill that systematically eliminates unexpected
output from a Ruby on Rails RSpec test suite.

[claude-code]: https://docs.anthropic.com/en/docs/claude-code

You know the feeling. You run `bundle exec rspec` and instead of a clean wall
of dots, you get this:

```
❯ bundle exec rspec
[DEPRECATION] This gem has been renamed to 'omniauth-entra-id' and will no longer be supported.
.......Warning: The code in the `before_worker_boot` block will not execute in
the current Puma configuration...
............Checking for expected text of nil is confusing and/or pointless...
...Checking for expected text of nil is confusing and/or pointless...
..WARNING: Using the `raise_error` matcher without providing a specific error...
.....Order ID: 2789
Order2 ID: 2790
.........true
true
```

Your eyes water. Your OCD bleeds. Somewhere, a Toyota engineer reaches for the
Andon cord.

Drop everything. Yes, everything. That critical production bug? It can wait. The
six-month feature you are one PR away from shipping? Not important. There is
only one priority now, and it stares back at you from your terminal in a cascade
of deprecation warnings and orphaned `puts` calls from 2019.

The. Output. Must. Be. Cleaned.

After running this skill. Bliss:

```
❯ bundle exec rspec
..............................................................................................................................
..............................................................................................................................
..............................................................................................................................

Finished in 4 minutes 2 seconds (files took 8.06 seconds to load)
892 examples, 0 failures
```

## Overview

This skill attempts to handles the full range of unexpected RSpec output:

- Ruby warnings (`already initialized constant`, `method redefined`)
- Gem and Rails deprecation notices
- Stray `puts`, `p`, `pp`, or `Rails.logger` calls left behind from debugging
- ActiveRecord or ActionView deprecation warnings
- Output from third-party libraries that should be silenced in test

It works through the output one issue at a time: locate the source test,
reproduce in isolation, fix the root cause, verify it's gone, then commit. Each
fix is its own commit so individual changes can be reviewed or cherry-picked
independently.

## Installation

Copy the skill to your Claude Code skills folder:

```bash
cp -r clean-rspec-output ~/.claude/skills/
```

Or clone directly:

```bash
git clone https://github.com/thoughtbot/clean-rspec-output ~/.claude/skills/clean-rspec-output
```

## Usage

From inside a Claude Code session, in the root of your Rails project:

```
/clean-rspec-output
```

The skill will ask which branch to work on before doing anything (it creates
one commit per fix, so the branch choice matters).

## How it works

1. **Capture**: runs the full suite twice: once in default format to identify
   noise, once in documentation format to locate the source test for each issue.
2. **Triage**: reads the output and groups issues by likely root cause.
3. **Fix loop**: for each issue: locates the test, reproduces the output in
   isolation with a single `rspec` command, fixes the root cause, verifies the
   output is gone, then commits.
4. **Final check**: re-runs the full suite and confirms nothing new surfaced.

The skill will not start planning or applying a fix until it can reproduce the
output with a scoped command. That reproduction case is also the verification
case, so there is no ambiguity about whether the fix worked.

## What it will and won't change

The skill fixes *how* a test runs. It should not change *what* it covers.

**Legitimate fixes:**
- Removing stray `puts`/`p`/`pp` calls left over from debugging
- Updating deprecated RSpec or Rails APIs to their modern equivalents
- Silencing third-party output at the configuration level
- Fixing incomplete factory data that causes a warning during setup

**Out of scope:**
- Removing assertions or collapsing branches to make noise stop
- Changing the values passed to `expect(...)` to ones that happen not to warn
- Rewriting or restructuring tests beyond what the noise fix requires
- Addressing test failures

## Contributing

Contributions are welcome. If you'd like to improve the workflow or add handling
for new categories of output:

1. Fork the repository
2. Create a feature branch (`git checkout -b my-new-feature`)
3. Commit your changes
4. Push to the branch (`git push origin my-new-feature`)
5. Open a pull request

## License

This skill is open source and available under the [MIT License](LICENSE).

## About thoughtbot

![thoughtbot](https://thoughtbot.com/thoughtbot-logo-for-readmes.svg)

This skill is maintained by [thoughtbot][thoughtbot]. We are passionate about
open source software and helping teams ship maintainable, well-tested software.

See [thoughtbot's other projects][community].

The names and logos for thoughtbot are trademarks of thoughtbot, inc.

[thoughtbot]: https://thoughtbot.com
[community]: https://thoughtbot.com/community?utm_source=github
