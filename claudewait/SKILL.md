---
name: claudewait
description: Wait for background Bash tasks to finish with `claudewait ID...`. Use whenever you need the result of a command you ran with run_in_background (or one that was moved to the background), instead of an until/while/sleep loop or polling its output file.
---

# Waiting for background tasks with claudewait

To get the result of a background task, run `claudewait` in the foreground with the task ID Bash gave you:

```
$ claudewait b09xrd4qp boxltsd4x
boxltsd4x  exit 0  after 15s  (output: /tmp/claude-1006/.../tasks/boxltsd4x.output)
b09xrd4qp  exit 2  after 41s  (output: /tmp/claude-1006/.../tasks/b09xrd4qp.output)
```

It returns when every task has finished (`--any`: the first one), printing each one's exit code, how long it ran and its output file; read that file for the output. After 110 s it lists the tasks still running and exits 124: run it again to keep waiting. For a longer `--timeout`, raise the Bash timeout to match.

Never wait with an `until`/`while` loop, `sleep`, `pgrep` or a `grep` of the log. They match the wrong thing or wait for text that never comes. `claudewait` waits for the task's own process to exit.

To wait for a line a task prints while it keeps running (a server's "listening on"), that is not this tool: use Monitor.
