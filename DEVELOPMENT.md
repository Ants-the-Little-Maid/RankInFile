# Rank in File: developer notes

How the addon is built and tested, the rules its code keeps, and how a release is made. The
player-facing overview, the design notes ("How it works") and the in-client checklist are in
[README.md](README.md); version history is in [CHANGELOG.md](CHANGELOG.md).

**Status:** version 0.4.3, released 2026-11-10 on GitHub and CurseForge (project 1788587).
The smoke test passes 161 of 161 locally and in GitHub Actions. 0.4.3 hasn't been run in the
Forever client yet; the README's "Points to check in the client" lists what to look at.

## Target

| | |
|---|---|
| Client | WoW Forever only, interface `16001` |
| Build checked against | 1.60.1.70790 (installed client) |
| API reference | Generated API docs from `Gethe/wow-ui-source` at `ca8368f2f2380cbe696854f7f2646387c278a61b` (1.60.1.70532) |

Those declarations say what exists and how secret values are treated; they don't prove how
the client behaves. Before passing an API result to a widget, check the function's `Secret*`
flags and the widget method's `SecretArguments` (see [Rules](#rules)).

## Files

| File | Role |
|---|---|
| `RankInFile.lua` | Everything secure and stateful: spellbook scan, macros, finding the bar slot that holds a macro, drawing, the secure snippets, button and menu construction, layout, slash commands, the move from Rank Flyout, the 0.4.0 hand-out, events |
| `Picker.lua` | The spell picker window. Nothing secure; it calls the core through `ns` |
| `Minimap.lua` | Minimap button and its settings menu (`MenuUtil.CreateContextMenu`) |
| `Bindings.xml` | The 12 `CLICK RankInFileButtonN:LeftButton` key bindings |
| `tests/rankinfile_smoke.lua` | Loads all three Lua files against stub WoW functions and checks behaviour |
| `.github/workflows/release.yml` | Smoke test on every push and pull request; packaging on `v*` tags |
| `.pkgmeta` | BigWigs packager settings: leaves `tests`, this file and `CLAUDE.md` out of the zip |

`ns` is shared between the files. The core exposes what the picker and minimap need
(`ns.PickUpSpell`, `ns.RemoveButton`, `ns.SetFloat`, `ns.SetDirection`, `ns.SetLocked`,
`ns.CursorSpell`, `ns.RankNumber`, `ns.CollectRanks`, `ns.FindFamily` and so on), and
`ns.OnChanged` / `ns.Changed()` batch "something changed" into one call per frame.

## Saved variables

| Table | Scope | Holds |
|---|---|---|
| `RankInFileDB` | Account | `direction`, `scale`, `locked`, `float`, `minimap = { angle, hide }`, `pickerRankedOnly` |
| `RankInFileCharDB` | Character | `buttons[n] = { family, spellID, top, point, wantSub }`, `moved = { [old Rank Flyout number] = new number }` |

- `family` is the spell name; ranks are found by scanning the spellbook for that name.
  `top = true` means "follow the highest known rank". `wantSub` is a rank's subtext
  ("Rank 3") still waiting for the spellbook, set by the Rank Flyout move.
- Left over from older versions and cleared on load or first use: `db.buttons` (0.4.0, becomes
  `db.shared040`), `db.shared040` (handed to the first character with a loaded spellbook,
  then deleted), `db.shared040Taken` (0.4.1), `db.movedButtons[guid]` (0.4.0, moved into that
  character's `moved`), `charDB.declined040` and `charDB.looked040` (0.4.2).

## Rules

These are the things a change is most likely to break.

- **Frame names and the macro body are a contract.** Bar macros and key bindings click
  `RankInFileButtonN` by name, and the addon finds its own macros by the text
  `RankInFileButtonN RightButton;` in the body (`MacroTag`). Renaming `BUTTON_PREFIX` or
  changing the body format strands every macro already on a bar unless a move is written for
  it, as was done for Rank Flyout in 0.4.0.
- **Character macros only.** The addon makes, edits and deletes character macros only
  (`ForEachMacro(fn, true)`). Account macros are shared by every character while buttons are
  per character, so editing one would change another character's bar.
- **Never overwrite a character's own buttons** when moving data from an older version. A move
  only fills empty button numbers.
- **Moves have no "done" flag.** Macros and bindings load in batches after `PLAYER_LOGIN`, and
  saved variables are partly account-wide while macros are per character, so each move is
  written to find nothing the second time and runs again on `UPDATE_MACROS` and
  `UPDATE_BINDINGS` (batched by `QueueCatchUp`).
- **Combat lockdown.** Anything touching protected frames (show, move, anchor, attributes outside
  the secure snippets), macros or bindings runs out of combat. In combat it sets a flag in
  `pending` (`rebuild`, `macros`, `anchors`, `migrate`) and `PLAYER_REGEN_ENABLED` does the
  work. A Shift-drag still going when combat starts is ended on `PLAYER_REGEN_DISABLED`.
  Only the secure snippets (`MAIN_PRECLICK`, `ARROW_ONCLICK`, `RANK_PRECLICK`) act in combat:
  they swap the `spell` attribute and show or hide the menu.
- **Secret values.** In instances some spell APIs return secret values. Addon code may not
  compare or test them, nor pass them to methods marked `AllowedWhenUntainted`
  (`Cooldown:SetCooldown` is one: 0.3.1 fixed an error flood from this). Check with
  `issecretvalue`, use duration objects for cooldowns, and keep drawing code inside `pcall`
  with `ReportOnce` so a surprise is reported once, not on every event.
- **`useOnKeyDown` stays false.** `/click` sends one click with `down = false`.
- **Spellbook bursts.** `SPELLS_CHANGED` and `LEARNED_SPELL_IN_SKILL_LINE` queue one rebuild
  0.1 s later (`QueueCatchUp(true)`); `PLAYER_LOGIN` rebuilds straight away. A rebuild scans
  the spellbook, the macros and up to 456 bar frame names per button, so don't call
  `RebuildAll` per event.
- **Docs stay neutral.** README, CHANGELOG and this file describe behaviour and check results
  only. They don't say who reported, tested or asked for anything.

## Running the tests

GitHub Actions runs the smoke test on Lua 5.1.5. With a local Lua 5.1, from the repository root:

```bash
lua tests/rankinfile_smoke.lua .
```

Without Lua, it runs under fengari (Lua 5.3 in JavaScript). Install fengari once in a folder
outside the repository (`npm install fengari@0.1.4`), then run this from that folder, with the
repository path as the last argument:

```bash
node -e "const {lua,lauxlib,lualib,to_luastring:s}=require('fengari');const L=lauxlib.luaL_newstate();lualib.luaL_openlibs(L);process.chdir(process.argv[1]);if(lauxlib.luaL_dofile(L,s('tests/rankinfile_smoke.lua'))){console.log(lua.lua_tojsstring(L,-1));process.exit(1)}lua.lua_getglobal(L,s('__FAILED'));process.exit(lua.lua_tointeger(L,-1)?1:0)" "path/to/RankInFile"
```

It prints `smoke: N passed, M failed` and exits non-zero on a failure. fengari accepts some
Lua 5.3 code that Lua 5.1 rejects, so CI is the final word; `luaparse` with
`luaVersion: "5.1"` catches syntax that only 5.3 allows.

What the smoke test does and doesn't cover:
- It stubs widgets, the spellbook, macros, bindings, the cursor and timers. `C_Timer.After`
  runs its function straight away unless a check replaces it.
- It runs the secure snippets as plain Lua against frame handles. That checks their logic, not
  that the client's restricted environment accepts them.
- The cursor stub only holds spells, so the picker putting down a held macro has no check.
- Every fix gets a check, and the check is confirmed to fail with the fix taken out.

## Releasing

1. Add a section to `CHANGELOG.md` (the packager uses it as the release notes), and update the
   smoke test count and version in the README's "Checks" table and the status above.
2. Commit to `main` and push.
3. Tag and push the tag: `git tag vX.Y.Z` then `git push origin vX.Y.Z`.

The tag runs `.github/workflows/release.yml`: the smoke test, then the
[BigWigs packager](https://github.com/BigWigsMods/packager), which:
- replaces `@project-version@` in the TOC with the tag,
- builds `RankInFile-vX.Y.Z-forever.zip` (interface `16001` is detected as Forever),
- creates the GitHub release and uploads the zip to CurseForge project `1788587`, using
  `## X-Curse-Project-ID` from the TOC and the repository secret `CF_API_KEY`.

`CF_API_KEY` must be a CurseForge author API token (from authors.curseforge.com). If the
CurseForge upload fails, the packager stops before the GitHub release too, so nothing is
published; fix the cause and rerun the failed job (`gh run rerun <id> --failed`). Don't re-tag
or re-upload an unchanged version: CurseForge counts duplicate uploads against its Fair Play
rules. Upload the packaged zip, never a zip of the repository folder: CurseForge rejects
archives containing `.git`.

## Installing a build locally

Copy the committed addon files (`Bindings.xml`, `CHANGELOG.md`, `LICENSE`, `Minimap.lua`,
`Picker.lua`, `README.md`, `RankInFile.lua`, `RankInFile.toc`) into the Forever client's
`Interface\AddOns\RankInFile`, replacing `@project-version@` in the TOC with the version.
`git archive HEAD <files>` gives exactly the committed versions. Saved settings are in
`WTF\Account\<account>\SavedVariables\RankInFile.lua` and, per character,
`WTF\Account\<account>\<realm>\<character>\SavedVariables\RankInFile.lua`; WoW writes them on
logout or `/reload`.

## Known limits

- The menu opens beside a bar slot only for Blizzard's bars, Bartender4 and Dominos (a fixed
  list of frame names, `BAR_BUTTONS`). Other bar addons get the menu beside the floating button.
- In combat the menu can't move, so it opens where it was last placed. A rank swapped in combat
  casts straight away, but the bar slot's icon, label and tooltip catch up when combat ends.
- Coming from 0.4.0, only the first character to log in with a loaded spellbook takes the old
  account-wide buttons. Other characters that hadn't logged in since 0.4.0 start empty.
- Each button needs one free character macro slot.
