# Modding Steam Games — the Platform, in Practice

The [game-modding module](15-game-modding.md) taught modding in general: the moddability
spectrum, reconnaissance, file formats, mod loaders, and — critically — the legal/ethical
rules (§1 and §8 there; **read them first, they still apply**). This module is the practical
companion for the most common real case: **the game is installed through Steam.** Steam adds
a first-class modding ecosystem (the Workshop), a specific on-disk layout you must learn to
navigate, and a handful of platform gotchas that trip up everyone exactly once.

It pairs with **[Depot](../project-12-depot/)**, a tool that parses Steam's own manifest
files to locate your installed games and subscribed Workshop content — the reconnaissance
step of doc 15, turned into something you can run against your real Steam install.

> A note for this machine: you're on macOS, where Steam lives at
> `~/Library/Application Support/Steam`. Many of the most-modded titles are Windows-only or
> run through a translation layer — §7 is honest about what that means for you.

---

## 1. Why Steam Matters for Modding

Steam isn't just where you bought the game — for modding it changes the whole picture:

- **The Workshop is the largest sanctioned mod distribution channel that exists.** For a
  Workshop-enabled game, modding can be as easy as clicking "Subscribe" — Steam downloads the
  mod and the game loads it, no files touched by hand. That's tier-1 modding (doc 15 §2) made
  trivial, and it's the developer *explicitly inviting* mods, which resolves most of the
  ethical ambiguity before you start.
- **Many games ship their mod tools through Steam.** Look in your library for a separate
  "\<Game\> Tools", "SDK", or "Editor" entry (under the Tools category) — that's the
  developer handing you the official level editor / mod kit.
- **But Steam also constrains modding**, in ways worth knowing up front: it can't host code
  (so script-injecting mod loaders live *off* Steam, on Nexus Mods — §4); its file-integrity
  check silently undoes hand-placed mods (§6); and its anti-cheat can ban you for modding the
  wrong (multiplayer) game (§6, §8).

The mental model: Steam is a layer *between* you and the game's files. Sometimes it's a
superhighway (Workshop), sometimes a locked cabinet you have to know how to open (finding and
editing files), and sometimes a tripwire (integrity checks, VAC).

## 2. Finding the Game's Files — Reconnaissance, Steam Edition

Doc 15's first reconnaissance step is "look in the install directory." On Steam, *finding*
that directory is the skill, because Steam hides it behind its own bookkeeping.

**Every game has an App ID** — a stable integer Steam uses internally. Find it in the store
URL (`store.steampowered.com/app/**570**/` → App ID 570), on the community site SteamDB, or
from the manifest filename below. The App ID is the key that unlocks everything else: it's how
you find the install folder, the Workshop content, and the manifests.

**The on-disk layout** (paths shown for macOS; Windows is
`C:\Program Files (x86)\Steam\...`, Linux `~/.steam/steam/...`):

```
~/Library/Application Support/Steam/
└── steamapps/
    ├── libraryfolders.vdf              ← lists EVERY library folder (incl. other drives)
    ├── appmanifest_<appid>.acf         ← one per installed game: state, name, installdir
    ├── common/
    │   └── <installdir>/               ← THE GAME'S FILES — where you mod
    └── workshop/
        ├── appworkshop_<appid>.acf     ← which Workshop items are subscribed
        └── content/
            └── <appid>/
                └── <publishedfileid>/  ← each subscribed Workshop mod's files
```

Two things make this non-obvious: games can install to **library folders on other drives**
(so `steamapps/common` on your main drive isn't the whole story — `libraryfolders.vdf` is the
index of them all), and the folder name under `common/` is the game's `installdir`, which
often differs from its store name. Depot exists to resolve exactly this.

**The shortcut** every modder uses: in Steam, right-click the game → **Manage → Browse Local
Files**. Steam opens the correct `common/<installdir>/` folder directly. Do that first; use
the manifest-reading approach (or Depot) when you need to script it, find *all* installs, or
locate Workshop content.

**The manifest files are your friend.** `appmanifest_<appid>.acf` is a small text file in
Valve's **VDF / KeyValues** format (nested `"key" "value"` pairs in braces) recording the
game's name, `installdir`, install state, and build id. Reading it tells you a game is fully
installed (vs. queued/updating) and exactly where. It's a lovely little file-format exercise
— which is why Depot parses it (and ties back to the [anatomy module](14-anatomy-of-a-codebase.md)'s
"read the format" lesson).

## 3. The Steam Workshop — the Sanctioned Superhighway

For a Workshop-enabled game this is where you start, and often finish:

- **Subscribing** to a Workshop item tells Steam to download it to
  `steamapps/workshop/content/<appid>/<publishedfileid>/` and keep it updated. Most such games
  then load subscribed items automatically (or via an in-game mod menu). No file editing, no
  archives, no risk of corrupting your install.
- **`appworkshop_<appid>.acf`** (VDF again) records what you're subscribed to and its download
  state — useful when a mod "isn't showing up" and you need to confirm it actually downloaded.
- **Load order matters** when mods conflict (two mods changing the same thing). Games with
  real mod scenes provide an in-game mod manager to order/enable them; the principle is the
  same "loose content overrides / last-writer-wins" idea from doc 15.

Because Workshop content is just files on disk under a path you can now find, subscribing to a
mod and *then reading what it installed* is one of the best ways to learn how a game's mods are
structured — you get a working example handed to you. That's Depot's lab (§8).

## 4. The Moddability Spectrum, Mapped to Steam

Doc 15's five tiers, re-expressed in Steam terms so you know which door to use:

| Tier (doc 15) | On Steam it looks like | Where the mods live |
|---|---|---|
| 1 · Official API / SDK | A "Tools/SDK/Editor" entry in your library; a Workshop with a documented uploader | Workshop, or the game's mod folder |
| 2 · Data-driven | Editable config/data files under `common/<game>/` | The game's own data folders |
| 3 · Asset replacement | Loose files overriding archived assets, or repacking archives | `common/<game>/`, often a `mods/` subfolder |
| 4 · Script/plugin loader | A community mod loader — **usually distributed on Nexus Mods, NOT the Workshop** | Loader's own folder inside the game dir |
| 5 · Reverse engineering | No official support; community tools | Wherever the community documents |

**The Workshop-vs-Nexus split is the key Steam-specific insight.** The Workshop is great for
*content* mods (maps, items, skins, scenarios) but Valve won't host mods that inject code, and
some large modding communities prefer to self-govern. So the pattern for heavily-modded games
is: **content mods on the Workshop, deep/code mods (and their loaders like a "script
extender") on Nexus Mods or the community's own site.** When a game's wiki says "install the
script extender," that's a tier-4 loader you get off-Steam and drop into the game folder — and
it's exactly the kind of mod Steam's integrity check will fight you over (§6).

## 5. Publishing a Mod to the Workshop (Brief)

When you've made something and want to share it the sanctioned way:

- **The game must be Workshop-enabled by its developer** — you can't add Workshop support to a
  game that lacks it. Most enabled games provide an **in-game uploader** or an SDK tool; use
  that first.
- **Under the hood** it's the Steamworks SDK's UGC (user-generated content) API (`ISteamUGC`),
  or **SteamCMD** with a `workshop_build_item` VDF script naming your App ID, the content
  folder, a preview image, title, and visibility. Steam packages and hosts it; subscribers get
  it automatically.
- **Terms:** publishing means agreeing to the Steam Workshop legal terms *and* the game's — and
  the doc 15 §1 rule stands: publish *your* work, never the game's copyrighted assets
  redistributed.

You likely won't need the SDK for a first mod (the in-game tools cover it), but knowing UGC +
SteamCMD exist tells you what those "Upload to Workshop" buttons actually do.

## 6. The Steam-Specific Gotchas (the stuff that bites everyone once)

These aren't in doc 15 because they're pure Steam, and each has burned countless first-time
modders:

- **"Verify integrity of game files" REVERTS your mods.** Properties → Installed Files →
  Verify integrity re-downloads anything that differs from Steam's copy — which is *by
  definition* your loose-file mods. Great for fixing a broken install; catastrophic if you
  forgot it wipes hand-placed files. Workshop mods survive (they live outside `common/`);
  loose-file mods do not.
- **Auto-updates break mods.** A game patch can change the very files or formats a mod
  depended on, silently breaking it. Mitigations: Properties → Updates → "Only update this
  game when I launch it" (plus staying offline / not launching), and **beta branches**
  (Properties → Betas) — many modded communities pin a specific older version there because a
  script extender only supports certain builds. There is no true "never update," so modders
  learn to control *when* they update.
- **Anti-cheat and bans — the serious one.** Modifying a game protected by **VAC** (Valve
  Anti-Cheat) or third-party anti-cheat (BattlEye, EAC) while on secured multiplayer servers
  can get you **permanently banned**, across the whole game family, non-appealably. VAC does
  **not** trigger on single-player or on unsecured/sanctioned modded servers — but the rule is
  simple and absolute: **mod single-player and developer-sanctioned contexts only; never touch
  competitive multiplayer.** This is doc 19 §8 and doc 15 §1 with a concrete enforcement
  mechanism attached.
- **Steam Cloud can fight your saves.** If a mod changes save formats, Cloud sync can restore
  an un-modded save or create conflicts. Know where the Cloud toggle is (Properties → General)
  before modding save data.

## 7. The Platform Reality on macOS (Honest Version)

You're on a Mac, and this shapes what's practical:

- **Steam itself runs fine on macOS**, and Mac-native games mod like any other — find the
  files under `~/Library/Application Support/Steam/steamapps/common/`. Note that a Mac game's
  files are sometimes *inside* the `<Game>.app` bundle (right-click → Show Package Contents),
  which is a Mac-specific reconnaissance wrinkle.
- **But many of the most-modded titles are Windows-only.** They run on Mac only through a
  translation layer (CrossOver, or Whisky/Wine), and mod tools are frequently Windows-only
  even when the game runs. Linux users hit the same via **Proton** (Valve's compatibility
  layer), where mods often work but paths shift into a "compatdata" prefix.
- **Practical takeaway:** for learning, favor a Mac-native, Workshop-enabled, single-player
  game — you'll spend your time modding, not fighting compatibility. Depot works regardless: it
  reads the manifests on whatever's installed.

## 8. Lab Track

Builds on doc 15's labs; these are the Steam-specific reps. (Labs needing a real Steam install
are optional — Depot's core is verifiable without one, via fixtures.)

| # | Lab | Where | What it teaches |
|---|-----|-------|-----------------|
| 1 | Read Depot's VDF/ACF parser and its fixture tests — Valve's KeyValues format decoded | Depot | The manifest file format (§2), a real file-format exercise |
| 2 | Run Depot against your real Steam install (if present): list installed games, their App IDs, install paths, and any subscribed Workshop content | Depot + Steam | Reconnaissance (§2) on your actual machine |
| 3 | Right-click a game → Browse Local Files; compare what you see to what Depot reported | Steam | The GUI shortcut vs. the scripted path (§2) |
| 4 | Subscribe to one Workshop item for a game you own, then find its files under `workshop/content/<appid>/` and read how the mod is structured | Steam | The Workshop path (§3) + learning from a working mod |
| 5 | Read an `appmanifest_<appid>.acf` and an `appworkshop_<appid>.acf` by hand; identify install state and subscribed items | Steam | The manifests as your source of truth (§2–3) |
| 6 | Before any loose-file modding: set a game to "only update on launch" and note where Verify-Integrity lives — so you know the gotchas *before* they bite (§6) | Steam | Defensive modding habits |

**Research prompts:** the VDF / KeyValues format (text and binary variants — `appinfo.vdf` is
binary); SteamDB (how it maps App IDs, depots, and build history); the Steamworks UGC API and
SteamCMD `workshop_build_item`; how Proton's compatdata prefix relocates game files on Linux;
why script extenders (SKSE-style loaders) live on Nexus and not the Workshop; how large mod
managers (Vortex, Mod Organizer 2) handle load order and "virtual file systems" that mod
without touching the real game files (the cleanest answer to §6's integrity problem); Steam's
depot/manifest content system (where "Depot" gets its name).
