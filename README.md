# IW4x Animated Camos

Animated weapon camos for **Modern Warfare 2 (2009)** running on the [IW4x](https://github.com/iw4x/iw4x-client) community client. The stock game has no animated camos. This adds a small client module that plays looping animations on the classic camos, plus a browser tool so anyone can make their own from a GIF.

> **Offline / private-match use.** This is a cosmetic, client-side change. It is not meant for public servers, and a modified `iw4x.dll` may not be welcome there. You need your own legitimate copy of the game. This repo contains no game files.

## What you get

- **A one-file installer** for players. It backs up your original `iw4x.dll` and the uninstaller restores it.
- **Camo Maker** (`camo-maker/index.html`): drop in a GIF, several pictures, or one picture with a scroll/colour-cycle effect, preview how it tiles on a gun, and download the file the mod reads. Runs entirely in your browser, nothing is uploaded.

## Install 

1. Download `AnimatedCamosSetup.exe` from the [Releases](../../releases) page.
2. Run it and pick your IW4x folder (the one containing `iw4x.exe`).
3. **Start the game by running `iw4x.exe` directly.** The IW4x launcher can replace the modified `iw4x.dll` with the stock one. If that happens, run the installer again.

Uninstall from Windows "Installed apps"; your original `iw4x.dll` is restored.

**Version note:** the included `iw4x.dll` is built from IW4x client **r5154** (commit `d43b9306`). If your IW4x is a different version, don't use the prebuilt DLL; build your own with the patch (see below).

## Make your own camo

Open `animcamo/CamoMaker.html` (installed with the mod, or `camo-maker/index.html` from this repo) in Chrome or Edge.

| Input | Result |
|---|---|
| One animated GIF / WebP | Becomes the animation (timing is preserved) |
| Several pictures | Used as frames, in file-name order |
| One normal picture | Pick a scroll direction and/or colour cycling |

Pick which camo it replaces, check the 3x3 tiling preview for seams ("make it tile smoothly" is on by default), download the `.bin`, and drop it in the `animcamo` folder next to `iw4x.exe`. Delete a `.bin` to go back to the default for that camo. Camos are 128x128 (pictures are cropped to a square).

## How it works

**The problem.** Every gun camo in MW2 is one 128x128 DXT1 texture (for example `weapon_camo_red_tiger`) that all guns share. Weapon materials use it as a "detail map" through a fixed shader, and there is no scroll or animation parameter to turn on.

**The approach.** A new IW4x client module, `AnimatedCamo`, replaces the picture itself, frame by frame:

1. At startup it reads `animcamo/<image name>.bin` files from the game folder.
2. About once a second, on the main thread, it looks up the camo image in the game's asset pool (`DB_EnumXAssets`). This keeps working if the image is unloaded and reloaded (for example after `vid_restart`).
3. On the render thread, once per frame, before anything is drawn (`Renderer::OnBackendFrame`), it points the image's base texture at the pre-built Direct3D 9 texture for the current frame, chosen from elapsed time and the animation's frame rate.

All frames are uploaded once as `D3DPOOL_MANAGED` DXT1 textures with full mip chains, so there is no per-frame upload cost. Nothing else is touched: no materials, shaders, models or weapon files. If the in-game image size differs from the animation (low texture quality can shrink images), that animation is skipped and a line is written to the log.

Notes and limits:

- Requires `r_useD3D9Ex` off (the default).
- The module skips itself on dedicated servers and in ZoneBuilder.
- Animation is client-side only. Other players see whatever camo you have equipped, not the animation.
- The camo selection menu thumbnails are still static.

### `.bin` file format (little endian)

```
char[4]  "ACAM"
uint16   version (1)
uint16   width
uint16   height
uint16   mip level count
uint16   frames per second
uint16   frame count
frames   each frame = every mip level, largest first, as raw DXT1 blocks
```

The default animations (`tools/make_frames.py`) are 128x128, 8 mip levels, 48 frames at 24 fps (about 513 KB each).

## Build it yourself

The module is a small patch on top of IW4x client r5154:

```
git clone https://github.com/iw4x/iw4x-client.git
cd iw4x-client
git checkout d43b9306
git apply path\to\installer\animated-camo.patch   # or: powershell -File path\to\tools\install_module.ps1 -RepoPath .
generate.bat                                       # then build Release / Win32
```

The result is `build\bin\Win32\Release\iw4x.dll`. Full steps (Visual Studio setup, swapping the DLL, undoing it) are in [`docs/HOW_TO_BUILD.txt`](docs/HOW_TO_BUILD.txt).

To build the installer: install [Inno Setup](https://jrsoftware.org/isinfo.php), copy your built `iw4x.dll` into `installer/`, open `installer/AnimatedCamos.iss` and compile (Ctrl+F9). The installer appears in `installer/Output/`.

To regenerate the default animations: `pip install numpy pillow`, then `python tools/make_frames.py`.

## Repository layout

```
src/Components/Modules/   AnimatedCamo.cpp / .hpp (the IW4x client module)
installer/                Inno Setup script, source patch, player readme
camo-maker/index.html     Browser tool for making your own camos
animcamo/                 The four default animations (.bin)
tools/                    Frame generator, DXT encoder, patch helper
docs/HOW_TO_BUILD.txt     Build guide
```

## Known limitations

- Prebuilt DLL matches IW4x r5154 only; other versions need a rebuild.
- The IW4x launcher can overwrite the modified `iw4x.dll`.
- Gun camos only cover the parts of each gun that the game's own texture mask allows; this mod does not change that.
- GIF import in Camo Maker needs Chrome or Edge (other browsers can use the several-pictures option).

## License and credits

This project is a modification of the IW4x client and is licensed under the **GNU General Public License v3.0**. The corresponding source for the distributed `iw4x.dll` is `installer/animated-camo.patch` applied to IW4x client commit `d43b9306`. See `installer/SOURCE_AND_LICENSE.txt`.

Built on the work of the [IW4x team](https://github.com/iw4x). This project is not affiliated with or endorsed by Activision or the IW4x project. *Call of Duty* and *Modern Warfare 2* are trademarks of Activision. Contains no game assets.
