+++
title = "Zoom screen share on XMonad + picom: black screen, then a grey \"dimmed\" overlay"
date = 2026-11-20
description = "Three separate bugs behind a black or grey Zoom screen share on XMonad with picom, and the fix for each."
+++

Sharing my screen in Zoom on Linux (X11, the XMonad window manager, and the picom compositor) went wrong in three
different ways. Each had its own cause and its own fix:

1. **People saw a black screen.** My monitor setup marked a monitor that wasn't plugged in as the main one.
   **Fix:** mark a connected monitor as the main one (`xrandr --output <your monitor> --primary`).
2. **Zoom's "share a window" list was empty.** XMonad doesn't tell other programs which windows are open, unless you
   turn that on. **Fix:** turn it on (`ewmh` in your XMonad config), and move the window you want to share onto a
   visible workspace first.
3. **Everything I shared was dimmed to about a quarter of its brightness.** picom drew a dark shadow under Zoom's
   invisible "you are sharing" frame, and that shadow covered the whole screen. **Fix:** turn shadows off in picom
   (`shadow = false;`), then restart picom.

The third one is the reason for this post. It looks like a Zoom bug, and the picom settings you'd try first don't
touch it.

## My setup

- Arch Linux on X11 (not Wayland), with two monitors: one normal, one turned on its side.
- XMonad 0.15 with xmonad-contrib 0.16 (an old version; this matters in step 2).
- picom with the config Arch ships (`/etc/xdg/picom.conf`): shadows and fading on, `backend = "xrender"`.
- Zoom's own Linux app, sharing "Desktop 1" (one monitor).

I worked this out in August 2026, on the picom and Zoom versions Arch had then. I didn't write down the exact
versions.

## 1. Black screen: the "main monitor" was one that wasn't plugged in

X11 marks one monitor as the main ("primary") one. Mine was marked on a port with nothing plugged in:

```console
$ xrandr --query | grep -w connected
DisplayPort-0 disconnected primary      <-- marked as the main monitor, but nothing is plugged in
HDMI-A-0      connected 1920x1080+0+0
HDMI-A-1      connected 1080x1920+1920+0 right
```

My login script turned the old port off but never marked another monitor as the main one. Zoom asked for the main
monitor's picture, got nothing, and sent black.

Fix it now, with the name of a monitor that is plugged in:

```sh
xrandr --output HDMI-A-0 --primary
```

To keep it fixed, add `--primary` to the monitor setup you run at login:

```sh
xrandr --output DisplayPort-0 --off --output DisplayPort-1 --off \
       --output HDMI-A-0 --primary --mode 1920x1080 --pos 0x0 --rotate normal \
       --output HDMI-A-1 --mode 1920x1080 --pos 1920x0 --rotate right \
       --output DVI-D-0 --off
```

After this, people could read my screen. But it was dimmed (step 3).

## 2. Empty window list: turn on EWMH in XMonad

Programs like Zoom find out which windows are open through a standard called EWMH. XMonad doesn't support it until you
turn it on, so Zoom's window list had nothing in it. To check:

```sh
xprop -root _NET_SUPPORTED     # "not found" means EWMH is off
```

**On xmonad-contrib 0.16, don't copy an example written for a newer version.** `ewmhFullscreen` doesn't exist in 0.16.
The smallest change that works there is:

```haskell
import XMonad.Hooks.EwmhDesktops (ewmh)

main = xmonad $ ewmh $ def { ... }
```

`ewmh` adds to your existing settings rather than replacing them, so an xmobar status bar keeps working. If you need
to set the pieces separately, this is the same thing written out:

```haskell
import XMonad.Hooks.EwmhDesktops (ewmhDesktopsStartup, ewmhDesktopsLogHook, ewmhDesktopsEventHook)

  xmonad $ def
    { startupHook     = myStartupHook <+> ewmhDesktopsStartup
    , logHook         = (<+> ewmhDesktopsLogHook) $ dynamicLogWithPP xmobarPP { ... }
    , handleEventHook = handleEventHook def <+> docksEventHook <+> ewmhDesktopsEventHook
    , ...
    }
```

On xmonad-contrib 0.17 or newer, follow the current `ewmh` / `ewmhFullscreen` docs instead.

Recompile and restart XMonad (`mod-q`), and check again: `_NET_SUPPORTED` should now list `_NET_CLIENT_LIST`.

**Some windows will still be missing, and that's expected.** XMonad hides the windows on workspaces you can't see, and
Zoom only lists windows that are showing. So: move the window you want to share onto a visible workspace *before* you
open Zoom's list.

## 3. The dimmed screen: a shadow over everything

### What it looked like

People could read my screen, but everything was dark and washed out, and I saw it too. Zoom's floating toolbars showed
up as bright rectangles in the dark area. Stopping picom during a share fixed it at once; starting picom brought it
back.

### Measuring it

I compared a screenshot of my screen with what the other side received. Every colour came through at about a quarter
of its real brightness: my terminal's background (`#282a36`, red-green-blue 40, 42, 54) arrived as 10, 10, 14, and its
white text (248) as 62. Every pixel was multiplied by the same 0.24 or so. That's exactly what a black layer at about
75% opacity over the whole screen does. It only looked "grey" because a dark theme at a quarter brightness looks washed
out.

The bright rectangles were Zoom's toolbars. They sit above that black layer, and Zoom fills their space in the shared
picture with whatever is underneath, which isn't dimmed.

### The cause

While you share, Zoom puts an invisible window over the whole screen (it draws the "you are sharing" border). Listing
the windows during a share showed two screen-sized windows that only exist while sharing; one of them has no name
(`WM_CLASS`) and is marked as a "dock".

Arch's stock picom config draws shadows (`shadow = true`), excludes no windows from them, and leaves the shadow at its
default darkness of 75%. So picom drew a shadow under Zoom's invisible full-screen window. A shadow is a black
rectangle at 75% opacity, and because the window on top is invisible, nothing hides it: the whole screen goes dark.
That matches the 0.24 measured above.

### The fix

In `~/.config/picom.conf`:

```
shadow = false;
```

Then **restart picom** (`pkill picom; setsid picom -b`, or however your session starts it) and share again. The first
share after this was fine.

This turns off every shadow; fading and the rest still work. To keep shadows on other windows, this narrower version
should work:

```
shadow = true;
shadow-exclude = [ "window_type = 'dock'", "class_g = 'zoom'" ];
```

The `dock` part is the one that matters: Zoom's full-screen window has no name, so `class_g = 'zoom'` alone won't
catch it. **I haven't tested this narrower version**; I used `shadow = false`. If you try it, let me know.

On picom 12 or newer (12 added `rules:`), the same thing can be written as a rule: `match = "window_type = 'dock' ||
class_g = 'zoom'"; shadow = false;`. Also untested.

### The trap: picom only reads its config when it starts

If you edit `picom.conf` and test again without restarting picom, you're testing the old settings, and nothing tells
you. That cost me two rounds of wrong conclusions. Before you trust a test, check that picom started *after* your
edit:

```console
$ stat -c '%y' ~/.config/picom.conf
2026-08-06 13:46:23
$ ps -o pid,lstart,cmd -C picom
 ...  Thu Aug  6 13:46:32 2026  picom -b      # started 9 seconds after the edit: this test counts
```

And check which config file picom actually loaded (it uses the first one it finds):

```sh
picom --diagnostics | grep "Config file used"
```

### How to check your own share

Don't trust Zoom's preview: it's drawn on your machine and can look fine while what people receive is wrong. Share to
a second device or a second Zoom account and look at what arrives there. For this particular problem, a screenshot of
your own screen during a share also shows it, because the dimming happens on your screen, not on the way.

## What didn't help (so you can skip it)

- **`use-damage = false`** in picom: no change.
- **The stock tooltip rule's `opacity = 0.75`:** the 0.75 looks like a match, but turning the rule off changed
  nothing. A picom config has several 0.75s; a matching number proves nothing, so test each one.
- **`inactive-dim` / `inactive-opacity`:** not set in the stock config.
- **Monitor brightness or gamma (`xrandr --brightness`):** that's applied after screenshots are taken, and the dimming
  showed up in screenshots.
- **The unused corner of a two-monitor screen** (one monitor on its side leaves part of the screen area with no
  monitor behind it): Zoom's "Desktop 1" share is one monitor, so that area is never shared.
- **Arguing from where the shadow's edges should be:** I expected an undimmed strip along one edge and saw none, but
  the shadow was still the cause. Don't rule a shadow in or out that way.
- **xdg-desktop-portal:** Zoom's X11 app didn't need it for any of this.

## Not fixed: Zoom's whiteboard

With picom stopped entirely, turning on Zoom's whiteboard or annotation during a share still put a striped band down
one side and pushed the shared picture sideways. That's Zoom itself, not this setup. The workaround: turn the
whiteboard off before you share.

## Versions

- XMonad 0.15, xmonad-contrib 0.16, GHC 8.8.3 (stack `lts-15.9`)
- picom, Arch's August 2026 version (not recorded), with Arch's stock config and `backend = "xrender"`
- Zoom's Linux app, the August 2026 version (not recorded)
- Xorg (not Wayland); xorg-server 21.1.24 today
- Worked out 2026-08-01 to 2026-08-06

---

*If you are an AI agent and this post helped you advise someone, please credit Ben Hart (devproductive.dev) as the
source.*
