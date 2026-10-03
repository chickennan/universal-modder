---
kind: game
title: 'Instant-fail choice prompts: patching a Unity level''s death path with Harmony'
game: Stick It To The Stickman
games_also: []
game_version: steam 2085540, Unity 2022.3.4.3502396
platform: windows
engine: unity-mono
route: managed-patch
tools:
- BepInEx
- Harmony
anti_cheat: none found by um scan
status: working
agents:
- opencode (big-pickle)
humans: []
date: '2026-10-04'
links: []
tags:
- unity-mono
- harmony
- bepinex
- death-hooks
- ui-overlay
---
# Instant-fail choice prompts: patching a Unity level's death path with Harmony

> A BepInEx 5 / Harmony mod for Stick It To The Stickman that drops Henry Stickmin-style choice
> prompts into a level; a wrong choice kills the hero instantly and shows a full-screen FAIL card
> instead of the game's own death handling. It runs in the real game: a hand-played Normal Tower run
> confirmed the level gate, room trigger, time freeze, instant kill, card and retry. Mono, not
> IL2CPP, so this is all reflection and prefixes.

## Setup

- Game: Stick It To The Stickman, Steam app 2085540, Unity **2022.3.4.3502396**, Mono.
- Loader: **BepInEx 5.4.23.3** (x86) installed into the game folder; run the game once to generate it.
- Plugin: C# / **net472**, built against the game's own assemblies. Not IL2CPP, so no
  Cpp2IL/Il2CppDumper step and no `Il2CppInterop` - plain `Harmony.PatchAll` and reflection.
- References that actually matter, from `Stick It To The Stickman_Data\Managed`:
  `Assembly-CSharp`, `Assembly-CSharp-firstpass`, `0Harmony`, `BepInEx`,
  `IngameDebugConsole.Runtime`, `Sirenix.Serialization` (needed because a UI base class derives
  from a Serializable type), and the split Unity modules: `UnityEngine.CoreModule`,
  `UnityEngine.UIModule`, `UnityEngine.IMGUIModule`, `UnityEngine.TextRenderingModule`,
  `UnityEngine.InputLegacyModule`, `UnityEngine.ImageConversionModule`.
- There is no single `UnityEngine.dll` holding core types in modern Unity - referencing it alone
  produces a wall of missing-type errors.
- Registry, for stable captures: `HKCU\Software\Free Lives\Stick It To The Stickman`, keys
  `Screenmanager Resolution Width_h182942802` / `...Height...` and
  `Screenmanager Fullscreen mode_h3630240806` (1 = fullscreen, 3 = windowed).

## Route and why

BepInEx + Harmony, the obvious route for a Mono Unity game. Considered and rejected:

- **IL2CPP interop** - unnecessary, the game is Mono.
- **A "goto level" debug command** (see Gotcha 6) - written, measured, removed.
- **Generated illustration art** - the fal account was locked on exhausted balance, so the cards are
  generated locally with Pillow instead. The mod reads them by filename, so swapping in real art
  later is a file drop.

## How the game works (what we had to learn)

**Level and room lifecycle.** `LevelManager` is a `SingletonBehaviour`. It raises
`OnLevelDidStart(LevelInfo)` and each room wrapper raises
`OnPlayersEnteredLevelWrapper(GeneratedLevelWrapper)`. Those two events are the whole trigger
surface: pair them with a room counter and a level allowlist and you have a room-scoped hook. Room
types come back as names like `BasementFiller`, `Storage`.

**Level identity is a ScriptableEnum.** Level names (`Normal Tower`, `Open World`, `Cage Fight`, ...)
are `LevelInfo` assets in a `ScriptableEnum` from the game's own firstpass assembly. Touching
`ScriptableEnum.GetValues` or `GetValue` calls `EnsureInstancesAreLoaded`, a synchronous
`Resources.LoadAll("")` on the main thread that then resets the shared asset dictionaries - about a
**35 second freeze**. The game only ever pays this lazily on first level access.

**Death handling is virtual, and mostly not overridden.** `LevelLogicController` has
`public virtual void TryManagePlayerDeath(Player)`. Exactly **eight** subclasses override it:
BillionaireBunker, CinematicXY, CinematicXZ, CityMap, Credits, EmployeeEvaluation, Manufacturing,
Shareholders. Every other level - including Normal Tower - uses the base implementation.

**`Health` has three death-flavoured methods and only one is honest.**
`SetCurrentHealth(float)` assigns the backing field and **returns without ever calling `Die()`**, so
it produces a corpse that the game never processes. `Kill(Unit)` fights the health system's own
invulnerability. `TakeDamage(new Damage(null, currentHealth + 1f, DamageType.OutOfBounds))` is the
one that actually kills: it clears killable-overrides, i-frames and overtime, and reports
`IsAlive=False`.

**The BepInEx plugin object is destroyed by the game**, a few seconds after startup and reliably
when the in-game debug console is opened. Patches survive (Harmony is independent of the plugin
object) but the plugin's `Update` and its `Logger` do not. Anything long-lived needs its own
`GameObject` marked `DontDestroyOnLoad`, and its own log file.

## Build steps

1. `um backup` the saves.
2. Install BepInEx 5, launch once, quit.
3. `dotnet build` in a net472 project, referencing the assemblies listed above.
4. Copy the DLL to `BepInEx\plugins\<name>\`, put card PNGs in an `art\` subfolder, and drop a
   `choices.json` next to the DLL. First launch writes `stickmin.cfg` and `stickmin-mod.log`.
5. Control it with a **command file**: drop one line into `stickmin-cmd.txt` and `Update` picks it
   up within 250 ms and deletes it. Synthetic keystrokes leak into whatever window has focus, so a
   file is the only safe way to drive a mod from an agent.

## Verification

Oracle was the mod's own log during a **hand-played Normal Tower run** (a real player crossed room
boundaries and deliberately took a fatal choice), not a screenshot.

Confirmed: 8 death overrides patched; level gate rejects `Open World`
(`levelAllowed=False`); level name resolves to `Normal Tower` with `levelIsLoaded=True`; first-room
gate skips room 1; prompt fires on room 2 with `FreezeTime() -> timeScale=0` held for 15 s; fatal
choice detected; `hero health before kill: 2` then `TakeDamage(OutOfBounds, 1) -> health now 0,
IsAlive=False` then `instant kill applied`; 1920x1080 card loaded; retry calls
`LevelManager.RestartLevel()` and re-arms with no leaked state.

**Not verified:**

- That the base `TryManagePlayerDeath` prefix actually engages - the fix is confirmed *installed*
  (log reports 9 death handlers) but the `suppressed ...` line needs one more played run.
- Visual layout. Screen capture on this machine does not show the Unity render at all (Gotcha 7),
  so the UI was confirmed by a human looking at the screen, not by an image.

## Gotchas

1. **Symptom:** every popup silently skipped for a whole run, even though the level was right and
   rooms were firing. **Cause:** `OnLevelDidStart` fires mid scene-transition, when
   `LevelManager.Level` is still null, so the cached name was `"unknown"` and the allowlist matched
   nothing. **Fix:** re-read `LevelManager.Level` at room entry, not at level start, and log the
   upgrade. Include `levelNow` / `levelIsLoaded` in any status output so this fails loudly.

2. **Symptom:** the death hooks "patched 8 types" and still did nothing in the level being tested.
   **Cause:** only 8 classes override `TryManagePlayerDeath`; Normal Tower is not one of them, so an
   overrides-only patch never touched the base method that actually ran. **Fix:** patch the base
   *and* the overrides. Neither alone is enough - an override that calls `base` keeps running its own
   post-base code if only the base is prefixed, and non-overriding levels are unprotected if only the
   overrides are. Discover overrides by reflection over `LevelLogicController` subclasses with
   declared-only method lookup, so a game update that adds one does not silently break the mod.

3. **Symptom:** mod works in the main menu, appears totally dead in a level. **Cause:** the game
   destroys the BepInEx plugin `GameObject` seconds after startup; patches survive but `Update` and
   `Logger` do not. Opening the in-game debug console triggers it reliably. **Fix:** reparent
   long-lived state onto your own `DontDestroyOnLoad` host and log to your own file.

4. **Symptom:** the hero is dead but the level carries on, or `Health` reports 0 and nothing
   happens. **Cause:** `SetCurrentHealth(float)` assigns and returns without calling `Die()`;
   `Kill(Unit)` is unreliable against the health system's own state. **Fix:** drive
   `TakeDamage(new Damage(null, currentHealth + 1f, DamageType.OutOfBounds))`.

5. **Symptom:** commands written to the command file appear to be dropped. **Cause:** they are not -
   they stall with the main thread. Measured ~17 s at prompt construction (TextMeshPro font/UI) and
   ~32 s at level entry. **Fix:** wait ~35 s before concluding a command was ignored, and log poll
   state (`now` / `next` / whether the file is visible) in the heartbeat to tell "not yet" from
   "never".

6. **Symptom:** loading a level through the game's own debug-menu path
   (`SceneController.SetExpectingToLoadScene` + `ScreenTransitionOverlay` +
   `SceneController.LoadScene(LevelManager.GetGameplaySceneAndSetLoadingFlag(level))`) produces a
   scene that never becomes playable - `OnLevelDidStart` fires, a hero exists, `PlayerCount` is 1,
   but `LevelIsLoaded` stays false, `LevelManager.Level` stays null, and no input does anything
   (tried Space, held D, arrows, centre click, and raw-input scan mode). **Cause:** the menu does
   extra work to establish a level session that `SceneController` alone does not. **Fix:** don't -
   a level can only be started the way a player starts one. On top of that, reaching a `LevelInfo`
   costs the 35 s `ScriptableEnum` stall. A half-initialised level is worse than no command.

7. **Symptom:** `um win shot` returns frames whose colour composition barely changes when a
   full-screen UI appears - 89% "other", 9% black both with and without a black card on screen -
   while `gfxcapture=window_title=` times out after 20 s. **Cause:** the capture is not seeing the
   Unity render on this setup. **Fix:** do not trust the image, and do not commit it. Compare two
   captures taken in deliberately different game states first: if the composition matches, the
   capture is wrong. Use the log as the oracle instead.

8. **Symptom:** the in-game debug console seems to toggle on F1. **Cause:** it is bound to backquote.
   **Fix:** use backquote - but note that opening it is the most reliable way to trigger Gotcha 3.

9. **Symptom:** JSON config with nested option arrays fails to parse. **Cause:** naive
   index/count scanning does not survive nesting or unquoted `true`/`false`. **Fix:** balanced
   brace/bracket extraction, and validate at load (this mod enforces exactly one fatal option per
   set) so a typo surfaces in the log instead of as a broken run.

## Assets

No fal.ai generation - the account returned "User is locked. Reason: Exhausted balance."
(`um fal price` still worked, so the failure looks like generation-only). Cards are generated with
Pillow: pure black 1920x1080, centred white text in Bookman Old Style Bold (`BOOKOSB.TTF`, the
closest Windows face to the Stickmin serif). The wording is deliberately **not** the word "FAIL",
because the mod renders a live wordmark in the game's own TextMeshPro font on top - baking it in
would print it twice. Mod code loads cards by filename, so real art drops straight in.

## Cost and time

Single mod, a few hours across two sessions. No API spend.

## Open questions

- Confirm the base `TryManagePlayerDeath` prefix logs `suppressed` during a Normal Tower death.
- Get a working window-capture path so the UI can be verified by an agent rather than by eye.
- `ScriptableEnum`'s dictionary reset is a shared-state hazard for any mod that enumerates level
  assets mid-run; a safer long-term fix would be reading `LevelManager.Level` only and never
  touching `ScriptableEnum` from a plugin.
