+++
title = "keybind-find: a command palette for your whole desktop"
date = 2026-11-22
description = "One hotkey searches everything you can do where you are, in the window manager and the app you're in, and does it for you by pressing its keys."
draft = true
+++

VS Code has a command palette: one key, type what you want, press Enter. My desktop doesn't. Everything I can do in
it is spread over the window manager, nvim, tmux, the terminal and my text snippets, and the only way in is a
keyboard shortcut I have to remember. I kept forgetting them: not the ones I use every hour, but the window split I
need twice a month, or the window manager command I set up once and never pressed again.

So I wrote [keybind-find](https://github.com/Benjmhart/keybind-find), a command palette for the whole desktop. You press
one hotkey (I use `Super+/`) and get a searchable list of everything you can do right where you are: your window
manager's commands, the ones for the program you're in (nvim, tmux, kitty, helix, mpv, your browser: more than 40
programs), and your espanso snippets. Search by name or by key. Pick one and keybind-find does it by pressing its keys
for you, and it shows you the key, so next time you can press it yourself.

![keybind-find over nvim: Super+/ opens the list, typing "light" puts Toggle light theme (Space u l) on top, and Enter sends the keys: the editor turns from dark to light](https://raw.githubusercontent.com/Benjmhart/keybind-find/a88c250/docs/media/demo.gif)

## It reads your real settings

The list never goes out of date, because keybind-find doesn't keep its own copy. It reads each program's settings
the way that program does, including the shortcuts you changed yourself. Some programs can simply be asked:

- **i3** is asked for the settings it loaded.
- **WezTerm** is asked for its full list of keys.
- **kitty** gets its standard keys for your version, with your own changes on top.
- **xmonad** lists your own keys with their names (it needs a few lines in your config).

For programs that can't be asked, keybind-find knows their standard shortcuts and adds your changes on top. When it
lists a program's built-in shortcuts, the top of the list says which version they come from, and whether that's the
version you have installed.

It works on X11 and on Wayland. Version 1.2.0 added WezTerm, zellij, Konsole, niri, awesome and sxhkd (bspwm), and on
GNOME and KDE under Wayland it can now press the keys for you too, once you allow it the first time. Version 1.4.0 adds
lazygit, qutebrowser, lf, nnn, Kakoune, Openbox, GIMP, Inkscape and LibreOffice.

## Try it

Get a ready-made binary or `.deb` from the [1.4.0 release](https://github.com/Benjmhart/keybind-find/releases/tag/v1.4.0),
or use Nix (`nix run github:Benjmhart/keybind-find/v1.4.0`, or NUR). Then set a key in your window manager to run it. For i3, that's one
line:

    bindsym Mod4+slash exec keybind-find

The README has a table of every program it reads and what each one needs.

## How it was built

An AI coding agent (Claude Code) wrote much of the code, to my design. I reviewed it, tested it, and use it every day.

Which program should it read next? Tell me in the [issues](https://github.com/Benjmhart/keybind-find/issues).
