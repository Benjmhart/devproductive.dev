+++
title = "keybind-find: the shortcuts for the window you're in, from one hotkey"
date = 2026-11-21
description = "Why I wrote a tool that lists the keyboard shortcuts that work right now, read from the real configs, and presses them for you."
draft = true
+++

I kept forgetting my own keyboard shortcuts. Not the ones I use every hour, but the ones I set up once and then never
remembered: a window split I need twice a month, a window manager command I wrote and never pressed again. Cheat
sheets didn't help, because mine were always out of date.

So I wrote [keybind-find](https://github.com/Benjmhart/keybind-find). You press one hotkey (I use `Super+/`) and get a
list of the shortcuts that work right where you are: your window manager's, and the ones for the program you're in
(nvim, tmux, kitty, helix, mpv, your browser: more than 40 programs). Type to search the list. Pick a shortcut and
keybind-find presses those keys for you, so you also learn the key for next time.

![keybind-find over nvim: the list narrows as you type, Enter sends the keys](https://github.com/Benjmhart/keybind-find/raw/main/docs/media/demo.gif)

## It reads your real settings

The list never goes out of date, because keybind-find doesn't keep its own copy. It reads each program's settings
the way that program does, including the shortcuts you changed yourself. Some programs can simply be asked:

- **i3** is asked for the settings it loaded.
- **WezTerm** is asked for its full list of keys.
- **kitty** gets its standard keys for your version, with your own changes on top.
- **xmonad** lists your own keys with their names (it needs a few lines in your config).

For programs that can't be asked, keybind-find knows their standard shortcuts and adds your changes on top.

It works on X11 and on Wayland. Version 1.2.0 added WezTerm, zellij, Konsole, niri, awesome and sxhkd (bspwm). On
GNOME and KDE under Wayland it can now press the keys for you too, once you allow it the first time.

## Try it

Install it with `cargo install keybind-find`, or get a ready-made binary or `.deb` from the releases page, or use Nix
(`nix run github:Benjmhart/keybind-find`, or NUR). Then set a key in your window manager to run it. For i3, that's one
line:

    bindsym Mod4+slash exec keybind-find

The README has a table of every program it reads and what each one needs.

## How it was built

An AI coding agent (Claude Code) wrote much of the code, to my design. I reviewed it, tested it, and use it every day.

Which program should it read next? Tell me in the [issues](https://github.com/Benjmhart/keybind-find/issues).
