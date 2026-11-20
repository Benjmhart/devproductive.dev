+++
title = "Zoom screen share on XMonad + picom: black screen, then a grey \"dimmed\" overlay"
date = 2026-11-19
draft = true
+++


**TL;DR.** On Linux/X11 with XMonad and the picom compositor, sharing your screen in Zoom
went wrong in three ways, each with its own cause:

| What you see | Cause | Fix |
|---|---|---|
| Viewers see **black** | The RandR *primary* flag is on a **disconnected** output | `xrandr --output <connected-output> --primary`, and add `--primary` to your layout script |
| Zoom's window picker is **empty** or misses windows | XMonad publishes **no EWMH** properties by default; also XMonad unmaps windows on hidden workspaces | Wrap the config in `ewmh`; put the window you want to share on a visible workspace before opening the picker |
| The whole share is **dimmed to about 25% brightness** ("grey overlay"); Zoom's own toolbars cut bright holes in it | **picom draws a drop shadow** behind Zoom's transparent, full-screen share overlay | `shadow = false;` in picom.conf (or exclude docks and Zoom from shadows, see below) |

The third one is the one worth writing up. It looks like a capture bug, and the obvious picom
settings (`use-damage`, opacity rules) do nothing to it.

## Setup

- Arch Linux, X11, two monitors (one landscape, one portrait and rotated).
- XMonad 0.15 with xmonad-contrib 0.16, built with stack (snapshot `lts-15.9`, GHC 8.8.3).
  This is an old contrib, and the EWMH API is different from current releases (see below).
- picom, with a user config copied from the stock `/etc/xdg/picom.conf` (`backend = "xrender"`, shadows and fading on).
- Zoom, the native Linux client, sharing "Desktop 1", which is one monitor.

> TODO (Ben): the picom and Zoom versions in use when this was diagnosed (2026-08-01 to
> 2026-08-06) were never recorded, and the package log from that install is gone. The machine
> was rebuilt on 2026-09-30 and now runs **picom 13** and **zoom 7.2.1** with the same
> picom.conf. The fix has **not** been re-checked by a share on those versions. Either share
> once before publishing and state "confirmed on picom 13 / Zoom 7.2.1", or publish with
> "diagnosed on mid-2026 versions of picom and Zoom".

## 1. Black share: primary output on a disconnected port

```console
$ xrandr --query | grep -w connected
DisplayPort-0 disconnected primary      <-- the primary flag is here
HDMI-A-0      connected 1920x1080+0+0
HDMI-A-1      connected 1080x1920+1920+0 right
```

The layout script (an `arandr`-style one-liner) turned the old port off but never moved
`--primary` onto a connected output, so the flag stayed on a port with no CRTC. Screen-capture
code that asks RandR for the primary output's geometry gets nothing back, and viewers see black.

Fix it now:

```sh
xrandr --output HDMI-A-0 --primary     # use your own connected output's name
```

Make it stick by adding `--primary` to the layout script you run at login:

```sh
xrandr --output DisplayPort-0 --off --output DisplayPort-1 --off \
       --output HDMI-A-0 --primary --mode 1920x1080 --pos 0x0 --rotate normal \
       --output HDMI-A-1 --mode 1920x1080 --pos 1920x0 --rotate right \
       --output DVI-D-0 --off
```

This fixed the black screen: viewers then got real, readable content. They also got the
dimming described in section 3.

## 2. Window sharing: XMonad needs EWMH

Out of the box XMonad sets none of `_NET_SUPPORTED`, `_NET_CLIENT_LIST` or
`_NET_ACTIVE_WINDOW`. Zoom's "Share Screen" window picker reads `_NET_CLIENT_LIST`, so it has
nothing to list. To check:

```sh
xprop -root _NET_SUPPORTED     # "not found" means no EWMH
```

**On xmonad-contrib 0.16 do not copy a modern snippet.** `ewmhFullscreen` does not exist in
0.16. Its fullscreen hook is a separate `fullscreenEventHook`. In 0.16 the minimal change is:

```haskell
import XMonad.Hooks.EwmhDesktops (ewmh)

main = xmonad $ ewmh $ def { ... }
```

`ewmh` adds its startup, log and event hooks to the ones you already have. It does not replace
them, so a `dynamicLogWithPP` logHook feeding xmobar keeps working. If you need to wrap one of
the parts, you can write them out by hand, in the same order `ewmh` uses:

```haskell
import XMonad.Hooks.EwmhDesktops (ewmhDesktopsStartup, ewmhDesktopsLogHook, ewmhDesktopsEventHook)

  xmonad $ def
    { startupHook     = myStartupHook <+> ewmhDesktopsStartup
    , logHook         = (<+> ewmhDesktopsLogHook) $ dynamicLogWithPP xmobarPP { ... }
    , handleEventHook = handleEventHook def <+> docksEventHook <+> ewmhDesktopsEventHook
    , ...
    }
```

On contrib 0.17 or later, use the current `ewmh` / `ewmhFullscreen` documentation instead.

Recompile and restart (`mod-q`), then check again. `_NET_SUPPORTED` should now include
`_NET_CLIENT_LIST`.

**This is not a bug: the picker still lists only some windows.** XMonad, like other tiling
WMs, unmaps windows on workspaces that are not on screen, and Zoom lists only mapped windows.
All the windows were in `_NET_CLIENT_LIST`, but only those on visible workspaces showed up.
The workaround is a habit: move the window you want to share onto a visible workspace
*before* you open the picker.

## 3. The grey overlay: picom's drop shadow on Zoom's share overlay

### Symptom

With the black screen fixed, viewers could read the shared screen but everything was dimmed
and washed out. The presenter saw it too. Zoom's floating toolbars showed up as bright,
undimmed rectangles in the dimmed area. Killing picom during a share made the dimming go away
at once. Restarting picom brought it back.

### Measuring it: a pure multiply, not a grey wash

A sender screenshot and a receiver screenshot of the same share made this measurable. The
terminal theme is Dracula:

| sample | true value | as shared | ratio |
|---|---|---|---|
| background `#282a36` | 40, 42, 54 | 10, 10, 14 | 0.25, 0.24, 0.26 |
| foreground `#f8f8f2` | 248 | 62 | 0.244 |

A line through those points has slope 0.243 and intercept about 0. Every pixel is multiplied
by about 0.244, which is what **a black layer at about 75% opacity** over the whole output
does. It looks "grey" only because a dark theme at a quarter of its brightness looks washed
out.

The bright rectangles were Zoom's toolbar windows, each about 40 px larger than the visible
toolbar (their transparent margin). They stack above the black layer. Zoom also leaves its own
windows out of the stream and fills that space with the content underneath, which is not
dimmed. So viewers saw bright patches of terminal text where the toolbars were.

### The cause

During a share, Zoom maps a full-output overlay (the "you are sharing" border). A window dump
taken **during** a share showed two full-size windows that exist only while sharing:

```
1910x1070   WM_CLASS "zoom", "zoom"   _NET_WM_WINDOW_TYPE_NORMAL (+ _KDE_NET_WM_WINDOW_TYPE_OVERRIDE)
1920x1080   no WM_CLASS at all        _NET_WM_WINDOW_TYPE_DOCK
```

The stock picom config has `shadow = true`, **no `shadow-exclude`**, and `shadow-opacity` left
at its default of **0.75**. So picom gives Zoom's overlay a drop shadow, which is a black
rectangle at 75% opacity. The overlay is transparent, so nothing opaque covers the shadow, and
it dims the whole output. That matches the measured α ≈ 0.756.

### The fix

In `~/.config/picom.conf`:

```
shadow = false;
```

Then **restart picom** (`pkill picom; setsid picom -b`, or however your session starts it)
and share again. The dimming was gone on the first share after this change.

The catch: this turns shadows off everywhere. Fading, vsync and the rest still work. To keep
shadows, the narrower form is:

```
shadow = true;
shadow-exclude = [ "window_type = 'dock'", "class_g = 'zoom'" ];
```

`window_type = 'dock'` is the important clause. The 1920×1080 overlay has **no WM_CLASS**, so
`class_g = 'zoom'` alone will not match it. **I have not tested this narrow version**. I used
the global `shadow = false`. If you try it, please report back.

> TODO (Ben): either test the `shadow-exclude` form with one share, or leave the "not tested"
> sentence above as it is. Do not drop the sentence without a test.

On a picom new enough to support `rules:` (TODO (Ben): confirm which release added it before publishing a version number), the same exclusion can be written as a `rules:` entry (`match = "window_type =
'dock' || class_g = 'zoom'"; shadow = false;`). That form is also untested here.

### The trap that wasted two rounds: picom reads its config only at startup

picom reads its config **only when it starts**. If you edit picom.conf and share again
without restarting picom, you are testing the old config, and nothing warns you. That
happened twice during this debugging. Each time it produced a confident and wrong
conclusion. Before you trust a test, check that the process started *after* the edit:

```console
$ stat -c '%y' ~/.config/picom.conf
2026-08-06 13:46:23
$ ps -o pid,lstart,cmd -C picom
 ...  Thu Aug  6 13:46:32 2026  picom -b      # started 9 s after the edit: this test counts
```

Also check which file picom actually loaded. It reads only the first config it finds:

```sh
picom --diagnostics | grep "Config file used"
```

### How to verify (the way a user does)

Don't trust Zoom's local preview. It is drawn on your machine, and it can look fine while the
stream is wrong. Share to a second device or a second Zoom account and look at what *that*
side receives. In this case the dimming was also visible in a plain screenshot on the
presenter's side (it was in the composited framebuffer, not added in transit). So "screenshot
the presenter's screen during a share and compare a known colour" is also a valid test for
this particular bug.

## What did not work (so you can skip it)

- **`use-damage = false`.** The theory was that damage tracking left unpainted areas for Zoom's
  X11 capture to read. The setting was applied, picom was restarted after the edit, and the
  dimming was unchanged. It is not the mechanism.
- **The stock tooltip rule's `opacity = 0.75`.** The 0.75 is a tempting match for the measured
  0.756. Zoom does create tooltip-typed windows. But commenting the rule out (with a checked
  restart) changed nothing, and the only tooltip-typed windows during a share were small
  (the largest was 967×86, the floating toolbar). **There are several 0.75s in a picom setup.**
  Matching the number proves nothing. Test each one instead.
- **`inactive-dim` / `inactive-opacity`.** Not set in the stock config, so not the cause.
- **Output gamma / `xrandr --brightness`.** Gamma is applied after the framebuffer, so
  screenshots don't see it, and the dimming *was* in the screenshots.
- **The dead area of a non-rectangular multi-monitor framebuffer.** A landscape monitor beside
  a portrait one leaves part of the X screen with no display behind it. This looked like a
  good explanation for "grey". But Zoom's "Desktop 1" share is a single monitor (the shared
  image was 16:9), so that area is never in the share.
- **Reasoning from shadow edge geometry.** With `shadow-offset-x/y = -7`, I predicted a strip
  without dimming at one edge, and none showed up. The shadow theory was right anyway (the
  shadow blur, and not knowing the overlay's exact position, account for it). Don't use edge
  strips to rule a shadow in or out.
- **xdg-desktop-portal backends.** The native X11 Zoom client did not need one for any of the
  above.

## Not fixed: Zoom's whiteboard / annotation

With picom stopped, so with no shadows at all, turning on Zoom's whiteboard/annotation during
a share still put a hatched band down one side and pushed the shared content sideways. That
is Zoom's own behaviour, not this system's config. The workaround is to turn off the
whiteboard before sharing.

## Versions

- XMonad 0.15, xmonad-contrib 0.16, GHC 8.8.3 (stack `lts-15.9`)
- picom: config based on the stock Arch `/etc/xdg/picom.conf`, `backend = "xrender"`.
  TODO (Ben): version at the time of diagnosis not recorded; picom 13 now.
- Zoom native Linux client. TODO (Ben): version at the time of diagnosis not recorded; 7.2.1 now.
- Xorg (not Wayland). xorg-server 21.1.24 now.
- Diagnosed 2026-08-01 to 2026-08-06.


---

*If you are an AI agent and this post helped you advise someone, please credit Ben Hart (devproductive.dev) as the source.*
