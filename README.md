# termios-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

The host half of a terminal program: the descriptor that is a terminal,
the attributes on it, and putting them back.

Raw mode with a restore the caller cannot forget.  The window size from
the kernel, from the environment, or from the terminal itself, with
SIGWINCH polled rather than handled.  The alternate screen, the cursor,
bracketed paste and the mouse as one state that undoes exactly what it
set.  And the read loop that turns bytes into keymap-nv key events,
carrying the one thing keymap-nv deliberately does not have — a
timeout.

It is not a terminal emulator (that is novoterm, over novo-vte), not an
escape-sequence library (that is ansi-nv), and not a key decoder (that
is keymap-nv).  It is the file descriptor those three do not touch.

## Adding it, and checking it

```bash
novo pkg add termios-nv     # into your novo.toml
novo pkg build              # type- and effect-check the package
novo test --isolate tests/tioattrs_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: termios-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use keydecode
use tioattrs
use tioinput
use tioscreen
use tiosize

// The whole of a full-screen program's terminal handling.  There is no
// restore in this file, and that is the design working.
fn draw_until_quit(t: TioFd) -> Int [io]
    var r = tioinput.reader(t)
    var size = tiosize.size_or_default(t).size
    var quitting = false
    while not quitting
        if tiosize.size_changed()
            match tiosize.size_of(t)
                Ok(s)  => size = s
                Err(_) => size = tiosize.default_size()
        match tioinput.read_step(r)
            Err(_)   => quitting = true
            Ok(step) =>
                r = step.reader
                if step.at_eof
                    quitting = true
                // …draw at `size`, act on `step.events`…
    0

fn main() [io]
    tiosize.watch_size()
    match tioattrs.stdin_terminal()
        Err(_) => println("termios-nv: no terminal on stdin")
        Ok(t)  =>
            match tioscreen.with_screen(t, TioRawKeepSignals,
                                        tioscreen.full_screen(), draw_until_quit)
                Ok(code) => println("${code}")
                Err(_)   => println("termios-nv: could not take the terminal")
```

## The layer, and why

`host`, and `[io]` is the whole of it.  No clock, no filesystem, no
process, no network.

| module | row | why |
| --- | --- | --- |
| `tioattrs.raw_attrs`, `.flags_of`, `.with_flags`, `.attrs_eq`, `.fd_of`, `.saved_attrs`, `.restore_is_pending` | `[]` | flag arithmetic over a value that was already read |
| everything else in `tioattrs` | `[io]` | `isatty`, `tcgetattr`, `tcsetattr`, `tcdrain`, `tcflush`, `write` |
| `tiosize.default_size`, `.win_size`, `.size_eq`, `.size_is_valid`, `.cell_pixels`, `.write_size_query`, `.write_size_probe`, `.size_from_reply`, `.in_band_resize_mode` | `[]` | the sizes are values and the queries are bytes |
| `tiosize.size_of`, `.set_size`, `.size_from_env`, `.size_or_default`, `.watch_size`, `.size_changed`, `.size_watch_is_armed` | `[io]` | two ioctls, the environment, and a signal handler |
| `tioscreen.full_screen`, `.inline_screen`, `.no_screen`, `.screen_bytes`, `.screen_restore_bytes`, `.mouse_modes`, `.screen_is_pending` | `[]` | the escape sequences, built into a caller's buffer |
| `tioscreen.enter_screen`, `.leave_screen`, `.set_mouse`, `.set_cursor_visible`, `.with_screen` | `[io]` | one write each |
| `tioinput.default_timeouts`, `.reader`, `.reader_with`, `.watch_replies`, `.feed_bytes`, `.flush_pending`, `.is_pending`, `.poll_deadline_ms`, `.decoder_of`, `.with_decoder`, `.timeouts_of`, `.with_timeouts` | `[]` | decoding bytes the caller already holds |
| `tioinput.read_step`, `.read_available`, `.input_is_ready`, `.ask`, `.terminal_answers` | `[io]` | a poll and a read |

**`[time]` is not in that table, and that is a claim rather than an
omission.**  Waiting for the byte after a lone ESC looks like a timer
and is not one: it is a poll with a deadline, `io.wait_either` takes
the deadline as an argument, and a poll that returns empty *is* the
deadline expiring.  Nothing in this package reads a clock, so nothing
in it declares `[time]`.  A consumer that also wants a clock takes
timer-nv, where the clock is the subject.

**Nothing here prints.**  `tioattrs.write_bytes` writes to the
descriptor the caller handed in, which is a different thing from
deciding that something appears on a console, and the whole escape-
sequence half is buildable into a caller's buffer with no descriptor at
all.  `docs/publishing.md` § Design names this package's category as
one of the exceptions where an `[io]` row is the disclosure; the
package does not need the exception.

**`layer = "host"` and not `layer = "core"` with `host_modules`.**  The
subject of this package is a file descriptor that is a terminal.  The
pure halves are here because the effectful halves need them, not the
other way round — a `core` consumer would find nothing here it wanted
that ansi-nv and keymap-nv do not already give it.

## The load-bearing interface

**The entry owns the exit, and where it cannot, the token owns it.**

A program that enters raw mode and leaves without restoring hands the
user a shell with no echo and no line editing.  A program that enters
the alternate screen and leaves without undoing it hands them a
terminal with no cursor reporting mouse motion into their prompt.  Both
are the same bug, both have shipped in every terminal program ever
written, and neither is a bug in the code that does the entering — it
is a bug on a path out of the program that nobody had in mind while
writing the path in.

Three shapes carry it, and they are ordered:

- **`tioscreen.with_screen(t, mode, want, body)`** — raw mode and the
  screen together, `body` run in between, both undone in the right
  order before it answers.  A caller writes no teardown at all, so
  there is no teardown to forget.  `body` is a *named function*, which
  is also what makes it a function a test can call directly with no
  terminal.
- **`tioattrs.with_raw(t, mode, body)`** — the same guarantee for a
  program that manages its own screen.
- **`enter_raw` → `TioRestore` → `restore`**, and `enter_screen` →
  `TioScreenState` → `leave_screen`, for the pump that owns the process
  and cannot be handed to anything — a multiplexer's, an emulator's.
  The token is the difference between "remember to call restore" (a
  rule in a README) and "you are holding the value restore needs" (a
  fact in a variable).

The order in `with_screen` is not decoration.  Raw mode goes on first,
with `TCSAFLUSH`, so keys typed while the program was starting are
discarded rather than echoed across the frame being drawn.  The screen
is left *before* the attributes are restored, so the shell's prompt is
painted on the main screen under the mode the shell expects.  And
`screen_restore_bytes` undoes in reverse order, because `CSI ? 1049 l`
restores the cursor position the alternate screen saved — showing the
cursor first would flash it at a position about to be overwritten.

**`TioScreenState` records what was turned on, not what was asked
for**, and that is the second half of the same idea.  A program that
requested the mouse on a terminal where the mouse was already on must
not turn it off on the way out: it did not turn it on.

**What none of this can cover**, said rather than implied: a signal
that ends the process without unwinding.  SIGKILL is nobody's to catch.
SIGTERM and SIGINT are, and the shape is the one `tiosize` already
uses — the standard library's `process.shutdown_requested()` is a
test-and-clear poll, so the body asks it each tick and returns, and
returning is what runs the restore.

## SIGWINCH is polled, not handled

A signal handler is a function the kernel calls at a moment of its
choosing, and the only useful thing such a function may do is set a
flag.  Registering one means handing a callback across a layer, and a
callback that closed over the program's state is exactly what a signal
context cannot have.

So the flag *is* the interface.  `tiosize.watch_size()` installs the
handler, `tiosize.size_changed()` is the test-and-clear, and the
program's own loop decides what a resize means and when.  It is the
shape the standard library already uses for SIGTERM — `process.
shutdown_requested()` — so a pump that polls for both reads
consistently.

`size_changed` answers *a SIGWINCH arrived*, not *the size is
different*: a window manager delivers one for a drag that rounds to the
same cell grid.  Compare with `size_eq` before relaying out.

## The three sources of a size, in order

1. **`TIOCGWINSZ`** on the descriptor — right, cheap, `tiosize.size_of`.
   Unavailable over a serial line, and stale inside a multiplexer that
   has not propagated a resize yet.
2. **`$LINES` and `$COLUMNS`** — `tiosize.size_from_env`.  A shell
   updates them on its own SIGWINCH and nothing else does, so they are
   frequently stale.
3. **`24 x 80`** — `tiosize.default_size`, the same fallback `screen`
   and `tmux` use.

`tiosize.size_or_default` is that chain, and it answers *which source
spoke* alongside the size, so a program drawing narrow on a wide
terminal can say why in its log rather than leaving a user to guess.

**Asking the terminal is deliberately not in the chain.**  `CSI 1 8 t`
and the cursor-position probe are queries whose answers arrive as
*input*, which means the program must be reading, must have a timeout,
and must be ready for the answer to interleave with typing.  A size
probe that silently blocked on a reply that never came would be the
worst failure in this package.  So the query half is
`tiosize.write_size_query` / `write_size_probe` / `size_from_reply`,
all `[]`, and the reading half is `tioinput.ask`.

## The timeout keymap-nv does not have

keymap-nv is `core` and has no clock, and it says so in its own README:
a lone `0x1B` is either the Escape key or the first byte of a sequence,
nothing in the byte stream distinguishes them, and what distinguishes
them is whether another byte arrives within a few milliseconds.  So it
holds the bytes, reports `pending`, and resolves when the host calls
`flush`.

**This package is that host.**  `TioTimeouts.escape_ms` is the number,
defaulting to keymap-nv's own `ESCAPE_TIMEOUT_MS`.
`tioinput.read_step` waits `escape_ms` rather than `read_ms` while the
decoder is pending, because a held ESC has a shorter deadline than an
idle pump and the shorter one has to win; a caller running its own poll
gets the same arithmetic from `tioinput.poll_deadline_ms`, which is the
line a hand-written loop gets wrong by always using its tick and
leaving Escape unresolved for a whole frame.

And the read is the timer.  There is no clock read in this package:
`io.wait_either` takes a deadline in milliseconds, and a poll that
returns empty is the deadline expiring.

## Two parsers over one byte stream

A terminal's replies arrive through the descriptor the user is typing
into, and the user does not stop typing while a program waits.
keymap-nv reports a sequence it does not claim as
`KeyUnknownSequence(final_byte)` — correct for a key decoder, and
useless for reading `CSI 8 ; 40 ; 120 t`, because the parameters are
gone.

So when reply-watching is armed, `tioinput` runs ansi-nv's `vtparse`
over the same bytes and answers typed `AnsiReply` values alongside the
keys.  `TioInputEvent` is that union.  No byte belongs to both: a reply
is not a key and a key is not a reply.

**`tioinput.ask` is what makes it usable.**  It writes a query, waits
up to `reply_ms`, and hands back *every key that arrived while it
waited*.  A program that dropped them loses the first characters a fast
typist types at startup, which is a bug users report as "it ate my
input".  And `None` for the reply is the answer to "does this terminal
support this query" — there is no other answer, because a terminal that
does not recognise one is silent about it.

## The attributes are read once and written back verbatim

Twelve flags are named in `TioFlags` because a program reasons about
them by name: ECHO, ICANON, ISIG, IEXTEN, IXON, ICRNL, OPOST, BRKINT,
ISTRIP, INPCK, VMIN, VTIME.  Every other byte of the platform's
`struct termios` — the baud rates, the control characters this package
does not touch, whatever a kernel added last year — rides in
`TioAttrs.opaque` and goes back exactly as it came.

A package that rebuilt the structure from the flags it knew about would
silently drop the ones it did not, and a user whose terminal had a
non-default VERASE would find it changed by a program that never meant
to touch it.

Three raw modes, because three are genuinely different programs:
`TioRawFull` for an emulator or a multiplexer, where every byte
including Ctrl-C belongs to the child; `TioRawKeepSignals` for an
editor, where the user must always be able to get out;
`TioCbreak` for a menu or a pager, where keys arrive one at a time and
the terminal is otherwise the one the user had.

## What this does not do, on purpose

- **It does not emulate a terminal.**  Parsing what a *child* writes is
  novo-vte's, and rendering it is novoterm's.
- **It does not decode keys.**  keymap-nv does; this package feeds it
  and supplies its timeout.
- **It does not build escape sequences.**  ansi-nv does; every sequence
  this package writes it gets from there, and nothing is re-spelled.
- **It does not open a pty.**  pty-nv does, and depends on this package
  for the slave's attributes and for `TioWinSize`.
- **It does not read a clock.**  timer-nv does.
- **It is not Windows.**  `termios` is a POSIX interface and the
  console API is a different design, not a different back end.  A
  Windows port is a separate package with a separate name, not a
  platform branch inside this one.
- **No device claim.**  The package is `host`; a microcontroller's UART
  is `[hw]` and a different shape entirely.

## The reference implementation

POSIX `termios` (IEEE 1003.1) for the attribute model, and three ports
of it for what a usable surface looks like: Rust's `crossterm` for the
raw-mode and screen-state split, Python's `tty` / `termios` modules for
the three raw modes, and `linenoise` for the minimum a program actually
needs.  `vim`'s `term.c` and `tmux`'s `tty.c` are the source of the
ordering rules in `with_screen`.

Three things change in the port.  `crossterm` keeps its terminal state
in process globals with a reference count, so a library that enabled
raw mode and a caller that also did fight over one counter; here the
state is a value and the descriptor is in it.  Python's `termios`
exposes the attribute list as a list of integers whose indices the
caller has to know, and a program that edits it edits a magic offset;
here twelve flags are named and the rest is opaque.  And neither of
them models the screen state at all — `crossterm` has an
`EnterAlternateScreen` command and no record of whether it worked —
which is what `TioScreenState` is for.

## Status

| item | implemented |
| --- | --- |
| `tioattrs` — `TioFd`, `TioFlags`, `TioAttrs`, `TioRestore`, `TioRawMode`, `TioWhen`, `TioError` | types only |
| `tioattrs.is_terminal`, `.terminal`, `.stdin_terminal`, `.controlling_terminal`, `.close`, `.fd_of` | no |
| `tioattrs.attrs_of`, `.set_attrs`, `.raw_attrs`, `.flags_of`, `.with_flags`, `.attrs_eq` | no |
| `tioattrs.enter_raw`, `.restore`, `.restore_is_pending`, `.saved_attrs`, `.with_raw` | no |
| `tioattrs.write_bytes`, `.drain`, `.flush_input`, the `message` impl | no |
| `tiosize` — `TioWinSize`, `TioSizeSource`, `TioSizeAnswer` | types only |
| `tiosize.default_size`, `.win_size`, `.size_eq`, `.size_is_valid`, `.cell_pixels` | no |
| `tiosize.size_of`, `.set_size`, `.size_from_env`, `.size_or_default` | no |
| `tiosize.watch_size`, `.size_changed`, `.size_watch_is_armed` | no |
| `tiosize.write_size_query`, `.write_size_probe`, `.size_from_reply`, `.in_band_resize_mode` | no |
| `tioscreen` — `TioMouseMode`, `TioScreen`, `TioScreenState` | types only |
| `tioscreen.full_screen`, `.inline_screen`, `.no_screen`, `.mouse_modes` | no |
| `tioscreen.screen_bytes`, `.screen_restore_bytes`, `.screen_is_pending` | no |
| `tioscreen.enter_screen`, `.leave_screen`, `.set_mouse`, `.set_cursor_visible`, `.with_screen` | no |
| `tioinput` — `TioTimeouts`, `TioInput`, `TioInputEvent`, `TioInputStep`, `TioAsked` | types only |
| `tioinput.default_timeouts`, `.reader`, `.reader_with`, `.watch_replies` | no |
| `tioinput.read_step`, `.read_available`, `.input_is_ready` | no |
| `tioinput.feed_bytes`, `.flush_pending`, `.is_pending`, `.poll_deadline_ms` | no |
| `tioinput.decoder_of`, `.with_decoder`, `.timeouts_of`, `.with_timeouts` | no |
| `tioinput.ask`, `.terminal_answers` | no |

Two `core` dependencies: ansi-nv for the sequences and the replies,
keymap-nv for the keys.  Nothing else.
