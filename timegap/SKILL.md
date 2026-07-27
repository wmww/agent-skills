---
name: timegap
description: Timestamp a program's output and measure the gaps between lines with `tg` (timegap). Use to find which step of a slow command eats the time, spot stalls in live logs, and debug timing-dependent behaviour.
---

# Measuring where the time goes with timegap

`tg` prefixes each line of output with seconds since it started, and marks every pause longer than 200 ms. The log becomes a timeline: whatever printed the line *above* a gap marker is what was running while that time went.

```
$ tg ./build.sh
0.000: resolving deps
     - 0.301s -
0.302: compiling
     - 1.402s -
1.704: linking
     - 0.202s -
1.906: running tests
     - 0.902s -
2.808: done
```

`tg` is https://codeberg.org/sphi/timegap. If it isn't installed, ask the user to install it rather than building it yourself.

## Use

```sh
tg CMD ARGS...      # run a command under tg
CMD | tg            # or time a stream you already have
```

Prefer `tg CMD`. It runs the command on a pty, so lines are timed as they are produced, stderr is timestamped along with stdout, and `tg` exits with the command's status.

Reach for the pipe form only for output you can't launch yourself (`journalctl -f | tg`). There the command's stdout is a pipe, so anything using stdio block-buffers and dumps everything at exit — a run where every line shares one timestamp is measuring the buffer, not the program. Fix it with `python3 -u`, `stdbuf -oL CMD`, `grep --line-buffered`, or by switching to `tg CMD`. Pipe form also leaves stderr untimed unless you add `2>&1`, and exits 0 regardless, so check `${PIPESTATUS[0]}`.

Options go before the command; anything after the command name belongs to it, and `--` ends the option list.

- `-m MS` — gap threshold, default 200. `-m 0` shows every gap, `-m 1000` only the egregious ones.
- `-g` — gaps only, no per-line timestamps. `-t` — timestamps only, no gap markers.
- `-C -R` — no colour, no live-updating clock. Pass both when the output is captured rather than watched by a human.

Reading the result:

- A gap before the first line is startup latency; a gap after the last line is teardown before exit. `-m 0` always shows both.
- A command that prints nothing gives a single gap: its total runtime.
- Times stay in plain seconds past a minute (`62.003:`), so they subtract directly.
- A gap means *nothing was printed*, which is usually the slow step but can also be a program that just doesn't log while it works. If a gap is suspicious, add logging and measure again.

## Recipes

Profile every command in a shell script — each gap lands on the exact `+ ...` line that was slow:

```sh
tg -m 100 bash -x ./build.sh
```

Just the stalls:

```sh
tg -g -m 500 ./cmd
```

Watch a live log for hangs, bounded by `timeout` since it won't return on its own:

```sh
timeout 60 tg -g -m 1000 tail -f app.log
```

Keep the timed log while still reading it:

```sh
tg ./cmd | tee timing.log
```

To compare a fast and a slow run, capture both the same way and diff them — timestamps drift after the first divergence, but the gaps stay comparable line by line.

## Notes

- The command sees a terminal, so tools that auto-detect one turn on colour and `\r` progress bars; each redraw gets its own timestamp. Pass the tool's own `--no-color` or quiet flag if that's noisy.
- Output is split on elapsed time as well as on newlines, so a partial line left hanging across a gap is flushed with its own timestamp instead of being swallowed.
- Input reaches the command normally, and killing `tg` takes the command with it.
- Overhead is negligible: 200k lines cost under 100 ms.
