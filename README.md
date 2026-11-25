# rpgmaker-player for xbox 

An experimental effort to run **RPG Maker XP fan games**, such as Pokémon Infinite Fusion, on an **Xbox in Developer Mode** by porting the open-source [mkxp-z](https://github.com/mkxp-z/mkxp-z) engine to the Universal Windows Platform (UWP).

> **Status: research phase. Nothing builds or runs yet.**
> No port exists today. This repo currently holds a reconnaissance workflow and the plan below. If you are looking for a playable app, there isn't one yet.

---

## What this project is

Many popular fan games (Pokémon Reborn, Rejuvenation, Insurgence, Infinite Fusion and others) are built on **Pokémon Essentials**, which runs on **RPG Maker XP**. On Windows they ship with a `Game.exe` that bundles an RPG Maker XP runtime.

Xbox consoles can't run a normal Windows `.exe`. They can run **UWP apps**, but only in Developer Mode. This project's goal is to turn the mkxp-z engine into a UWP app so the same game files can be played on an Xbox with a controller, keyboard, and mouse.

The longer-term idea is a small **JoiPlay-style launcher**: pick a game folder, see a library, and launch it.

## What this project is not

- Not affiliated with, endorsed by, or connected to Nintendo, Game Freak, The Pokémon Company, the mkxp-z team, or the developers of any fan game.
- Not a way to play official commercial games. It is only meant to run **fan games and content you own or have the right to use**, copied onto your console yourself.
- Not a Minecraft, PlayStation, or Android emulator. Those were considered and set aside.

---

## UWP and Xbox Developer Mode, explained

**UWP (Universal Windows Platform)** is Microsoft's app model that runs on Windows, Xbox, and some other devices. A UWP app runs inside a **sandbox** (an "app container"), which limits what it can do:

- It can only use a restricted subset of Windows APIs. Many classic desktop APIs, including some used for windows, process control, and DLL loading, are blocked or replaced.
- It reads and writes files only in its own storage, plus folders the user explicitly grants through a file or folder picker.
- It is delivered as a signed package (`.msix` or `.appx`), not a loose `.exe`.

**Xbox Developer Mode** lets you install your own UWP apps on a retail Xbox:

- You turn it on with the *Dev Mode Activation* app from the Microsoft Store. It requires a Microsoft Partner Center developer account, which has a one-time registration fee. Check Microsoft's site for the current price.
- Dev Mode runs in a separate environment from retail games. Apps you sideload only run there.
- You install packages through the **Xbox Device Portal**, a web page served by the console. You can open it from a phone on the same Wi-Fi.
- Dev Mode apps have limited resources. Setting an app's type to **Game** instead of **App** (done on the console) is reported to give it more.
- Dev Mode allows a few things retail doesn't, such as generating code at runtime, which is why emulators can work there.

**Why this matters for mkxp-z:** mkxp-z is written for desktop operating systems. To run in the UWP sandbox, every part of it (graphics, windowing, audio, the Ruby interpreter, file access) has to work within UWP's restrictions.

---

## How the port is expected to work

```
Xbox (Dev Mode)
 └─ UWP app container
     ├─ Window + input layer (SDL2, UWP/WinRT variant)
     │    ├─ Controller input
     │    └─ Keyboard and mouse
     ├─ Graphics: OpenGL ES calls -> ANGLE -> Direct3D 11
     ├─ Audio (OpenAL Soft or an SDL-based backend)
     ├─ mkxp-z engine (RPG Maker XP reimplementation)
     │    └─ Ruby 3.1 interpreter running the game's scripts
     └─ Game files you provide (Data/, Graphics/, Audio/, Game.ini)
```

The game's own data (`Data/*.rxdata`, `Graphics/`, `Audio/`) stays as it is. Only the engine that replaces `Game.exe` is ported.

---

## What the recon found

A scan of the mkxp-z source (224 source files) showed the following.

**Encouraging**
- The code has **no `CreateProcess`, `LoadLibrary`, or `VirtualAlloc` calls**. `GetProcAddress` appears 5 times and `CreateFile` 7, so the engine's own use of restricted Windows APIs is small.
- **ANGLE is already vendored**, and the build has a Direct3D 11 option. ANGLE translates OpenGL ES to Direct3D, which Xbox supports.
- The source asks SDL for an OpenGL ES context when ANGLE is used, so an ES path already exists.
- **Gamepad support** through SDL's controller API is already in the code.
- The Steam and Discord integration is in 4 files and can be removed.
- It has components that emulate the Windows API calls Pokémon Essentials makes (`miniffi`, `binding-mri-win32.h`).

**Risks and unknowns**
- **Ruby 3.1** is built from source as part of the build. Ruby on Windows depends on many desktop APIs, and how much of that survives UWP is the largest unknown.
- The existing Windows build uses MinGW-style clang and gcc cross files and links desktop libraries (`user32`, `shell32`, `shlwapi` and others) that UWP restricts. A UWP-compatible toolchain is needed.
- SDL2 reportedly has a WinRT/UWP port, but it hasn't been tested with mkxp-z.
- OpenAL Soft's behavior on UWP is untested.
- Memory limits in Dev Mode may matter for large games.
- Graphics performance through ANGLE on Xbox is unmeasured.

---

## Roadmap

| Phase | Goal | Status |
|-------|------|--------|
| 0 | Recon: scan mkxp-z's structure, dependencies, and API usage | Done |
| 1 | Reproduce mkxp-z's normal Windows build in GitHub Actions | Not started |
| 2 | Build the pieces for UWP: SDL2 (WinRT), ANGLE (D3D11), OpenAL, and Ruby 3.1 | Not started |
| 3 | Link the engine into a UWP app, package it as `.msix`, and run a test game on an Xbox | Not started |
| 4 | Add a launcher: game library, folder picker, controller-friendly menus | Not started |
| 5 | Optional: a player for RPG Maker MV/MZ games (they run on web technology, so a web view may work) | Not started |

Any phase may reveal a blocker, and the project could stop at phase 2.

---

## Planned features

Everything here is a goal. None of it is built or tested yet.

| Feature | Details | Phase |
|---------|---------|-------|
| **Controller support** | Xbox controller as the main input, using SDL's gamepad API, which mkxp-z already uses | 3 |
| **Keyboard and mouse support for all games** | USB keyboard and mouse (wired or 2.4 GHz dongle; Bluetooth ones generally don't work on Xbox). Keyboard play should work like on Windows, and the mouse should only show a cursor when a game needs one | 3 |
| **Key rebinding menu** | mkxp-z has its own settings menu. The plan is to make sure it works with both a controller and keyboard under UWP | 3-4 |
| **Game library** | A home screen listing your games, with a cover image and last-played time | 4 |
| **Folder picker** | Choose where your games live, using UWP's file and folder picker. What the picker can reach on Xbox, such as USB storage, needs testing | 4 |
| **Per-game settings** | Saved controls, scaling, and audio choices for each game | 4 |
| **Save management** | Keep saves in a known folder and let you back them up or copy them off the console | 4 |
| **Scaling and display options** | mkxp-z already ships scaling shaders (bicubic, Lanczos, xBRZ). The plan is to expose them in a menu | 4 |
| **Per-game compatibility fixes** | Small script patches for games that need them, kept in this repo and applied by the launcher | 4+ |
| **RPG Maker MV/MZ player** | A separate web-based player for MV and MZ games | 5 |

---

## Supported games

"Supported" will only mean tested on a real Xbox. Until then, every entry below is a target.

| Game or engine | Built on | Status |
|----------------|----------|--------|
| **Pokémon Infinite Fusion** | RPG Maker XP, Pokémon Essentials | Primary target. Untested |
| Pokémon Reborn, Rejuvenation, Insurgence, Uranium | RPG Maker XP, Pokémon Essentials | Candidates. They use the same engine family, so they may work, but none are tested |
| Other RPG Maker XP games | RPG Maker XP | Candidates. Untested |
| RPG Maker VX and VX Ace games | RPG Maker VX / VX Ace | Candidates. mkxp-z includes VX bindings, but untested |
| RPG Maker MV and MZ games | JavaScript | Planned for phase 5 through a separate player |
| HTML games | Web technology | Possible in phase 5 |
| Ren'Py games | Python | Not planned. They need a Python runtime, which is a separate big project |

Game compatibility depends on each game's scripts. Games that call Windows features the port doesn't cover may need patches, and some may not work.

To suggest a game, open an issue with the game's name and the engine it uses (a `Game.ini` plus `.rxdata` files means RPG Maker XP).

---

## Running the recon yourself

You need a GitHub account. A phone browser is enough.

1. Add the recon workflow file to `.github/workflows/` in your repo.
2. Open the **Actions** tab, choose **mkxp-z UWP recon**, and tap **Run workflow**.
3. When it finishes, open the run page and download the `recon` artifact at the bottom.
4. `report.txt` inside it holds the full results.

The workflow only reads the source. It doesn't compile anything, so it takes a few minutes.

---

## Playing games once this works

When a build exists, the expected flow is:

1. Enable Developer Mode on your Xbox and note the console's Device Portal address.
2. Download the package from this repo's Actions artifacts or Releases.
3. Sign it if needed, then upload it through the Device Portal.
4. Copy your own game folder to the console (a USB drive or the Device Portal file tools). How well each of these works in Dev Mode still needs testing.
5. Launch the app and choose the game folder.

Controller and keyboard and mouse should both work. Games that rely on Windows-only features may need script fixes.

---

## Legal and licensing

- mkxp-z is open source. Check its `COPYING` file for the license terms. Any work derived from it must follow them.
- Fan games belong to their authors. Follow each game's terms.
- Don't use this project with pirated, ripped, or otherwise unauthorized game files.
- All trademarks belong to their owners.

## Credits

- The [mkxp-z](https://github.com/mkxp-z/mkxp-z) contributors, and the original mkxp project it forks.
- The SDL, ANGLE, OpenAL Soft, and Ruby projects.
- Projects such as DuckStation-UWP, which show that emulator-style apps can run in Xbox Dev Mode.
