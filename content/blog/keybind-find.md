+++
title = "keybind-find: the shortcuts for the window you're in, from one hotkey"
date = 2026-11-21
description = "Why I wrote a tool that lists the keyboard shortcuts that work right now, read from the real configs, and presses them for you."
draft = true
+++

I kept forgetting my own bindings. Not the ones I use every hour: the ones I set up once, for a split I make twice a
month or a window manager command I wrote and never pressed again. Cheat sheets didn't help, because mine never matched
my config for long.

So I wrote [keybind-find](https://github.com/Benjmhart/keybind-find). Press one hotkey (I use `Super+/`) and you get
a searchable list of the keys that work where you are right now: your window manager's, and the focused program's
(nvim, tmux, kitty, helix, mpv, a browser: more than 40 programs in all), plus your espanso snippets. Pick an entry and
keybind-find presses those keys for you, so it doubles as a command palette that shows you the key you could have
pressed instead.

![keybind-find over nvim: the list narrows as you type, Enter sends the keys](https://github.com/Benjmhart/keybind-find/raw/main/docs/media/demo.gif)

## Read from the real config, never kept by hand

The point is that nothing goes out of date. Each list is read from the program itself, the way the program reads it:

- **i3** is asked for the config it loaded (`i3-msg -t get_config`), so includes and variables come out as i3 sees
  them.
- **WezTerm** is asked through `wezterm show-keys`, so the list is WezTerm's own resolution of your `wezterm.lua`.
- **kitty** starts from kitty's defaults for its version and puts your `kitty.conf` on top, as kitty does.
- **xmonad** reads your keymap with each binding's name (a few lines in `xmonad.hs`).

Where a program can't print its keys, keybind-find carries that version's defaults and applies your config over them,
and says so.

It works on X11 and on Wayland. Version 1.2.0 added WezTerm, zellij, Konsole, niri, awesome and sxhkd (bspwm), and on
GNOME and KDE Plasma under Wayland it can now press the keys too, through the desktop's remote-desktop portal after a
one-time approval.

## Using it

Install it with `cargo install keybind-find`, a release binary or `.deb`, `nix run github:Benjmhart/keybind-find`, or
from NUR,
then bind it to a key in your window manager. For i3, one line:

    bindsym Mod4+slash exec keybind-find

The README's "Works with" table lists every program it reads and what each needs.

## How it was built

Much of the code was written with an AI coding agent (Claude Code), to my design; I reviewed it, tested it, and use it
every day under xmonad.

Which program should it read next? Requests and bug reports go to the
[issues](https://github.com/Benjmhart/keybind-find/issues).
