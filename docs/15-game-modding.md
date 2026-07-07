# Game Modding — Changing a Game You Didn't Write

Modding is programming against a system whose source you don't have, whose author didn't
plan for you, and whose rules you have to *discover*. That makes it one of the best
applied exercises in this whole curriculum: it fuses file-format literacy (module 14),
how-games-are-built knowledge (module 13), and reverse engineering into one skill. Many
professional engineers started exactly here.

This module is deliberately scoped to **legitimate modding**: single-player games, games
with official mod support, and your own projects. Read §1 before anything else — the line
between "modding" and "cheating/piracy/malware" is real, and staying on the right side of
it is part of the craft, not a footnote.

---

## 1. The Rules of the Road (read first)

Modding lives in a legal and ethical grey zone that you navigate by a few firm rules:

- **Respect the EULA.** Some games explicitly permit mods (often the ones with thriving
  scenes); some forbid them. The license is the contract — read it. "Everyone does it"
  is not a defense.
- **Never redistribute the game's assets or code.** A mod is *your* additions plus
  *instructions that reference* the original files the user already owns. Ship your new
  art, scripts, and configs — never a copy of the game's copyrighted content. This is
  why mods are "install into your copy," not "download the whole modded game."
- **Don't circumvent DRM or anti-piracy.** Cracking copy protection is a different
  activity with different laws (DMCA §1201 in the US) and is out of scope here.
- **Single-player and cooperative-by-consent only.** Modifying a game to gain advantage
  in competitive multiplayer is cheating; it breaks other people's experience and trips
  anti-cheat systems (which treat memory tampering as an attack — correctly). Keep mods
  to single-player, private servers, or games whose communities sanction it.
- **No malware, ever.** Mod installers are a classic malware vector precisely because
  users run them with trust. If you distribute a mod, distribute clean, inspectable,
  source-available work.

The mindset: you are a *guest* extending someone's world with their (explicit or implied)
permission, for players who own the game. Everything below assumes that footing.

## 2. The Spectrum: How Moddable Is This Game?

"How do I mod X?" has no single answer because moddability is a spectrum set by the
developer. Identify where your target sits before you plan anything:

| Tier | What the game gives you | Difficulty | Example-class |
|---|---|---|---|
| **1. First-class mod support** | Documented APIs, a scripting language, an SDK, a workshop | Lowest | Games shipping Lua/Python mod APIs, official level editors |
| **2. Data-driven & open** | Config/data files in text or documented formats, no code needed | Low | Games storing rules/levels/items in JSON, XML, INI, CSV |
| **3. Asset replacement** | Assets in known or crackable containers you can swap | Medium | Replacing textures/models/audio in archive files |
| **4. Script/plugin injection** | A community-built loader hooks the engine so your code runs | High | Community mod loaders / injectors for popular engines |
| **5. Binary patching / RE** | Nothing intended; you reverse-engineer and patch | Highest | Changing compiled behavior directly |

The skill is *recognizing the tier* and using the lowest-effort door that exists. People
attempt tier-5 binary hacking for something that was a tier-2 config edit all along.
**Always look for the official/data door first.**

## 3. Reconnaissance — Reading a Game You Can't See

Every mod starts with investigation. The detective method from the
[debugging module](09-debugging-and-observability.md) applies directly — you're forming
and testing hypotheses about a system's internals:

1. **Look in the install directory.** This is step one and people skip it. Folders named
   `data/`, `mods/`, `scripts/`, `assets/`, `config/` are invitations. File extensions
   tell you the tier: `.json/.xml/.ini/.cfg/.lua/.txt` = tier 2, you may be minutes from
   a mod. `.pak/.pk3/.assets/.dat/.bin` = a container (§4).
2. **Identify the engine.** It dictates *everything* about how modding works. Tells:
   a `UnityPlayer.dll` or `*_Data/` folder (Unity), `Engine/`+`.pak` files (Unreal),
   `.pck` (Godot), a `love`/`.love` file (LÖVE). Once you know the engine, you know the
   community tools, the asset formats, and the scripting story. Search
   "how to mod \<engine\> games" — the engine is more useful than the game title.
3. **Find the community.** Modding is overwhelmingly a *communal* reverse-engineering
   effort. Nexus Mods, game-specific wikis, the modding subreddit/Discord — someone has
   likely mapped the file formats and built tools already. Standing on that work isn't
   cheating; it's how the entire field operates. (It's also where you learn the specific
   game's EULA stance.)
4. **Inspect files with general tools.** A hex editor (`hexdump -C`, ImHex, HxD) reveals
   magic numbers and text strings in "binary" files; `file` guesses types; `strings`
   pulls readable text out of binaries (often revealing paths, formats, even Lua source).
   This is module 14's "read the format" lesson with the training wheels off.

## 4. Working With Game Files

### Text/data files (tier 2) — start here whenever possible
The dream case: the game reads its rules from files you can edit in a text editor. Weapon
damage in a JSON, spawn tables in CSV, UI layout in XML. Your workflow is: back up the
original, change a value, launch, observe. This is *exactly* what Rebound's `levels.js`
ASCII layouts and `config.js` constants are — a data-driven surface a modder edits without
touching code. The lab (§8) makes you build that surface on purpose, so you understand it
from the developer's side.

Watch for: schema validation (some games reject malformed files — your edit must stay
well-formed), and checksums (some verify file integrity and revert changes — a signal the
developer *didn't* intend edits there).

### Archives (tier 3) — the container problem
Most games pack thousands of assets into a few big archive files (`.pak`, `.zip`-family,
custom formats) — fewer files loads faster and bundles neatly. To mod assets you must:
unpack (community tool, or the format is documented), replace an asset with yours *in the
same format and dimensions/constraints*, and repack — or drop a loose file the engine
prefers over the archived one (many engines support this "loose files override" path,
which is the cleanest mod mechanism there is). The recurring gotcha: your replacement must
match the original's format precisely (codec, color space, sample rate, vertex layout), or
the engine rejects or corrupts it. This is a file-format fidelity exercise.

### Save files — a common, gentle entry point
Save files are often just serialized state (sometimes JSON/XML, sometimes compressed or
lightly obfuscated). Editing them to understand a game's data model is low-stakes
single-player practice. If it's compressed, `file`/magic bytes tell you the scheme; if
it's plain, you're reading the game's domain model directly — a nice tie-back to how
*your* projects serialize state.

## 5. Scripting APIs & Mod Loaders (tiers 1 & 4)

The best-case modding is **writing code against an API the game exposes** — commonly
**Lua** (small, embeddable — the industry's default game-scripting language; worth
learning for this reason alone), sometimes C#, Python, JavaScript, or a bespoke language.
You write scripts against documented hooks: "run this when an entity spawns," "register a
new item," "add a menu command." If your target offers this, learn *its* API — that's the
whole job, and it's ordinary programming with an unusual standard library.

When a game has *no* official API but a big community, modders build a **mod loader** (or
"injector"): a program that loads the game, then loads community plugins into it so their
code runs inside the game's process. Conceptually it hooks the engine — intercepting or
adding to function calls — so your plugin's code executes at the right moments. The deep
version of this ("hooking," "detours," DLL injection) is advanced, engine-specific, and
sits right next to anti-cheat territory — which is why it's **single-player only** and why
you should always prefer an official API or a data surface when one exists.

Two concepts you'll meet here that generalize:
- **Hooking / interception** — making your code run in place of (or around) existing code.
  It's the same idea as the request middleware in TaskFlow, or a debugger's breakpoints:
  inserting yourself into a call path you don't own.
- **The plugin/loader pattern** — a host program discovers and loads independent modules
  at runtime against a defined contract. That contract *is* an API boundary, the same
  concept as this workspace's repository interfaces and REST contracts — just enforced by
  a loader instead of a compiler.

## 6. Reverse Engineering (tier 5) — the deep end, in outline

When nothing is exposed and no community tool exists, modders reverse-engineer: observe
behavior, inspect the running process, and infer structure. The honest summary of what's
involved (each is a field of its own):

- **Static analysis** — disassemblers/decompilers (Ghidra is the free, superb standard;
  it's an NSA-released tool taught in university RE courses) turn a compiled binary back
  into readable-ish assembly or pseudo-C. You read it to find *where* a behavior lives.
- **Dynamic analysis** — debuggers and memory scanners (Cheat Engine is the classic
  learning tool, aptly named — keep it to single-player) watch values change as you play,
  to *locate* the data structure behind, say, the score or health.
- **The loop** — hypothesize where a value or behavior lives, verify by observing or
  changing it, document, repeat. It is the scientific-method debugging loop again, on the
  hardest possible target.

This is a legitimate, valuable skill (it's the core of security research and
vulnerability analysis too), and also the slowest, most game-specific path. Learn it
because it's fascinating and broadly useful — not as a first resort for a mod that a
config file could have delivered.

## 7. The Knowledge Base — What You Actually Need

Mapped to what this workspace already taught you, plus what's new:

| You need | You have it from | Modding adds |
|---|---|---|
| File formats: text (JSON/XML/INI) and binary | [Anatomy module](14-anatomy-of-a-codebase.md); every project's serialization | Reading *undocumented* binary formats with a hex editor |
| How games are structured (loop, entities, assets, data-driven design) | [Game module](13-game-development.md) + Rebound | Inferring another team's version of the same structures |
| The detective/scientific method | [Debugging module](09-debugging-and-observability.md) | Applying it with no source and no symbols |
| A scripting language, usually **Lua** | Your polyglot base makes this a weekend | Lua specifically; embedding/sandboxing models |
| APIs and contracts | [Networking](08-networking-and-apis.md), repository pattern everywhere | Contracts enforced by a loader/engine, not a compiler |
| Engine literacy | Godot port story (module 13) | Recognizing engines from their footprint; per-engine toolchains |
| Version control & release discipline | [Git module](02-git-workflow.md) | Packaging installable mods; semantic versioning for compatibility |
| **New:** RE tooling | — | Hex editors, Ghidra, memory scanners — the deep-end kit |

The meta-skill, and the thing to take away: **modding is API discovery.** Whether the API
is a documented Lua interface, an undocumented JSON schema, an archive format, or a
function you found in a disassembler, you are reverse-engineering a contract and then
programming against it. Every other skill is in service of that.

## 8. Lab Track — Learn Modding From Both Sides

The best way to understand modding is to make something moddable, *then* mod it — you'll
never design a worse mod API after feeling it from the developer's chair.

| # | Lab | Where | What it teaches |
|---|-----|-------|-----------------|
| 1 | **Mod Rebound with zero new code**: edit `src/game/levels.js` ASCII layouts and `config.js` constants; make a "chaos mode" (fast ball, tiny paddle, custom level) | Rebound | You're a tier-2 modder against a data-driven game — the easiest, most common real case |
| 2 | Read `parseLevel` in `levels.js` as a *mod loader author*: it's a schema + trust boundary for external content. What does it accept? What should it reject? | Rebound | The developer's side of "users will feed my parser garbage" |
| 3 | **Build a real mod surface** — story S3 below: load levels from an external file/URL the game doesn't ship, with validation and error UX | Rebound | Designing an API for content you don't control; the loose-files-override pattern in miniature |
| 4 | Recon a game you own with official mod support (or an open-source game): find its data/script files, identify the engine, read the format, make one documented change | A real game | The reconnaissance method (§3) on a real target, within the rules (§1) |
| 5 | Learn Lua (an afternoon) and write a trivial script against any game or app that embeds it | Lua + a target | The tier-1 dream case; the language the field runs on |

**Research prompts:** why Lua became games' embedded-scripting default (small, fast,
sandboxable); the "loose files override archives" pattern across engines; how Ghidra
reconstructs pseudo-C from a binary; sandboxing untrusted mod code (the security problem
you inherit the moment you run other people's scripts — a real concern for *your* mod
loader in Lab 3); semantic versioning for mod-vs-game-version compatibility; how official
mod workshops handle distribution without shipping game assets.
