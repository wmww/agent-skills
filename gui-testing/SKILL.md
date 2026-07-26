---
name: gui-testing
description: Run a Linux GUI program in a disposable session, take screenshots and inject mouse/keyboard events. Use to test rendering/interaction/etc of a Wayland/X11 app.
---

# GUI testing in a headless Wayland session

`guibox` starts a private headless sway on its own `WAYLAND_DISPLAY`, runs a program in it, and deletes everything on stop (or after 10 minutes, whichever comes first). Sessions are fully independent, so several agents can run at once.

## Requirements

`sway`, `grim`, `wdotool`. Optional: `wl-clipboard` for pasting text.

If a tool is missing, **ask the user to install it** and wait - don't run the install yourself, don't work around a missing tool, and don't fall back to the user's live session. Arch: `pacman -S sway grim wl-clipboard`; wdotool: `cargo install --git https://github.com/wmww/wdotool wdotool`.

## Use

```sh
DIR=$(gui-testing/guibox start -- myapp --flag)   # returns once the app's window is up
. $DIR/env                                        # session env; needed in every later shell
grim $DIR/1.png                                   # screenshot the output, then read it
wdotool mousemove 200 249 click 1                 # act on it, perhaps screenshot again
gui-testing/guibox stop $DIR                      # kill the app, delete the dir
```

`start` options: `-s WxH` output size (default 1280x720), `-t SECS` TTL (default 600), `-x` enable Xwayland. Everything else is plain `grim`/`wdotool`/`swaymsg`.

Without `. $DIR/env` those tools talk to the user's real desktop instead. Keep screenshots in `$DIR` so they vanish with the session; copy out anything you need to persist.

## Interacting

The window is tiled to fill the output with no decorations, so screenshot pixel coordinates are exactly what you click.

```sh
wdotool mousemove X Y click 1          # buttons 1=left 2=middle 3=right
wdotool click 1 --repeat 2             # double-click
wdotool scroll 0 5                     # <DX> <DY>; positive dy scrolls down
wdotool type 'hello'                   # chords: wdotool key ctrl+shift+t
wl-copy 'long text' && wdotool key ctrl+v
```

Subcommands chain in one invocation, which is what anything holding a button or modifier needs - a chain runs against one set of virtual devices:

```sh
wdotool mousemove 742 147 mousedown 1 mousemove 800 147 mousemove 880 147 mouseup 1
wdotool mousemove 120 249 keydown Shift_L click 1 keyup Shift_L
```

Use real keysym names when pressing a modifier alone (`Shift_L`, `Control_L`), not the chord spellings that `key` accepts. Avoid `2>/dev/null` - it hides errors and failures. Touch isn't available: no Wayland protocol exposes virtual touch, and uinput events don't reach a headless compositor. Touch-only gesture handling stays untested.

## Compositor control

`swaymsg` is available once `$DIR/env` is sourced:

```sh
swaymsg -t get_tree                       # windows, app_ids, geometry
swaymsg exec 'other-app'                  # launch more programs in the session
swaymsg '[app_id=foo] kill'               # X11 windows have no app_id; match [class=foo]
```

Launch extra programs with `swaymsg exec` rather than backgrounding them from your own shell - that way they are tagged as part of the session and get cleaned up with it.

## Notes

- The app's stdout/stderr and the compositor log go to `$DIR/log`. If no window appears, `start` prints the tail of it immediately instead of waiting.
- `start` returns as soon as the window maps, which for Wayland means it has painted. Browsers, Electron apps and X11 clients may map earlier - if a screenshot looks blank or half-drawn, sleep a second and retake.
- Windows that can't shrink to the output are clipped; give a larger `-s`.
- The session has no D-Bus and no portals; apps generally log warnings about this and carry on.
- The headless compositor should advertise `linux-dmabuf` and GPU rendering should work fine.
- In some cases it may be helpful to give apps a disposable profile so they don't collide with other instances (e.g. `firefox --profile /tmp/ff --no-remote`).
- `-x` runs X11 clients under Xwayland. Prefer Wayland, only use X11 support with X-only apps.
- All the same tools work on X11. `wdotool type` sends ASCII only there - non-ASCII characters are dropped, so paste those with `wl-copy 'text'` then `wdotool key ctrl+v`.
- Cleanup is automatic: `stop`, the TTL, and a sweep of dead sessions on the next `start` all remove the processes and the directory. `guibox list` shows live sessions, which may belong to other agents - don't stop dirs you didn't start or blindly killall by name.
