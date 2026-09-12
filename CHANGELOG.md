# Changelog

All notable changes to termios-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `tioattrs` — `TioFd`, the descriptor that is known to be a terminal;
  `TioAttrs` with twelve named flags and every other byte carried
  opaquely; three raw modes; `with_raw` for the guarantee and
  `enter_raw` / `TioRestore` / `restore` for the pump that cannot be
  handed to anything; the one write in the package.
- `tiosize` — `TioWinSize` with pixel dimensions, the three sources in
  the order a program wants them and the source named alongside the
  answer, SIGWINCH as `watch_size` plus the `size_changed`
  test-and-clear, and the query pair for a terminal with no ioctl.
- `tioscreen` — the alternate screen, cursor visibility, bracketed
  paste, focus events, four mouse settings and synchronised output as
  one `TioScreen` request answering a `TioScreenState` that records
  what was actually turned on; `screen_bytes` and
  `screen_restore_bytes` for a caller with its own writer; and
  `with_screen`, the one call a full-screen program makes.
- `tioinput` — the reader that feeds keymap-nv and supplies the
  timeout it does not have; `TioTimeouts` with the three deadlines;
  `feed_bytes` for a caller whose keystrokes arrive off a socket; and
  `ask`, which asks the terminal a question and keeps every key that
  arrived while it waited.

### Known

- **The entry owns the exit.**  `with_screen` and `with_raw` run the
  caller's loop and undo everything before answering, so there is no
  restore in the caller's code to forget.  The explicit token pair is
  there for a pump that owns the process.
- **`TioScreenState` records what was turned on, not what was asked
  for**, so a program does not turn off a mode it found already on.
- **SIGWINCH is polled, not handled** — the same test-and-clear shape
  the standard library's `process.shutdown_requested()` uses.
- **`[time]` is not declared** and no clock is read: a deadline on a
  poll is an argument to a syscall, not a reading of a clock.
- **Two parsers over one byte stream.**  keymap-nv's
  `KeyUnknownSequence` carries a final byte and no parameters, so a
  query reply cannot be read out of it; with reply-watching armed the
  reader runs ansi-nv's `vtparse` alongside and answers typed
  `AnsiReply`s.
- **Widening asked of keymap-nv**: if `KeyUnknownSequence` carried the
  CSI parameters, the second parser would be unnecessary for the
  common case.  Raised, not worked around.
- **The attributes go back verbatim.**  Twelve named flags, and every
  other byte of the platform structure untouched.
- **POSIX only.**  A Windows console port is a separate package with a
  separate name.
- **No device claim.**  The package is `host`.
- **Two `core` dependencies**: ansi-nv and keymap-nv.
