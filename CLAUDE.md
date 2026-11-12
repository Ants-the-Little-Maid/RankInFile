# Notes for agents working on Rank in File

Read [DEVELOPMENT.md](DEVELOPMENT.md) before changing code: it has the status, the rules the
code keeps (frame names, character macros only, combat lockdown, secret values), how to run the
tests and how releases work.

- **Forever only.** Look up APIs for the Forever client (interface `16001`), and check every
  `Secret*` flag before a result reaches a widget.
- **Test every change.** Run the smoke test (`lua tests/rankinfile_smoke.lua .`, or the fengari
  command in DEVELOPMENT.md). Each fix gets a check that fails without the fix.
- **Ask the author before** committing, pushing, tagging, releasing, or anything else that
  reaches GitHub or CurseForge. A pushed tag publishes to both.
- **Never handle the CurseForge token.** The author sets the `CF_API_KEY` repository secret.
- **Keep the docs neutral.** README, CHANGELOG and DEVELOPMENT.md never mention users,
  players or testers, or who reported or asked for what.
- **Moving old data never deletes a character's own buttons.** It only fills empty numbers.
- When shipping a version, update CHANGELOG.md, the README "Checks" table and the status line
  in DEVELOPMENT.md together.
