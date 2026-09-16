# termios-nv

**termios** is the POSIX interface to a terminal: the attributes that
decide whether the kernel echoes what is typed, whether it waits for a
whole line before answering a read, and whether Ctrl-C becomes a signal
or a byte. It is specified in IEEE Std 1003.1 chapter 11, "General
Terminal Interface", and declared in
[`<termios.h>`](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/termios.h.html).
This package brings it to novo-lang, together with the rest of what a
full-screen program does to the terminal it was started on: the window
size, the alternate screen, the mouse, and the read loop that turns
bytes into key events. It is built on
[ansi-nv](https://novo-lang.org/packages/ansi-nv) for the escape
sequences and [keymap-nv](https://novo-lang.org/packages/keymap-nv) for
the keys.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What it is

By default a terminal is in **canonical mode**. The kernel collects
what the user types, applies the erase and kill characters, and answers
a read only when Enter is pressed. It echoes each byte as it arrives,
and it turns Ctrl-C, Ctrl-\ and Ctrl-Z into signals rather than
delivering them as bytes. That is what a shell wants and what a
full-screen program cannot use.

**Raw mode** is canonical mode with those services turned off, so every
keystroke reaches the program as it happens. It is not one setting but
a set of flags, and this package names twelve of them, which are the
ones raw mode changes and the ones a program turns back on one at a
time when full raw mode is too much.

| Flag | What it does when on |
| --- | --- |
| ECHO | The kernel writes each input byte back |
| ICANON | The kernel buffers a line and answers the read at the newline |
| ISIG | INTR, QUIT and SUSP become signals rather than bytes |
| IEXTEN | Ctrl-V quotes the next character, Ctrl-O discards output |
| IXON | Ctrl-S stops output and Ctrl-Q resumes it |
| ICRNL | A carriage return arrives as a newline |
| OPOST | The kernel post-processes output, so a lone LF becomes CR LF |
| BRKINT | A break condition raises SIGINT |
| ISTRIP | The eighth bit of every input byte is cleared |
| INPCK | Input parity is checked |
| VMIN | The fewest bytes a read will answer with |
| VTIME | The read's deadline, in tenths of a second |

Everything else in the platform's `struct termios` — the baud rates,
the control characters, whatever a kernel added last year — is carried
through unread in `TioAttrs.opaque` and written back exactly as it was
read.

Three raw modes are offered, because three different kinds of program
want different amounts taken away. `TioRawFull` is `cfmakeraw`:
everything off, which is what a terminal emulator or a multiplexer
wants, because every byte including Ctrl-C belongs to the program on
the other side. `TioRawKeepSignals` leaves ISIG on, so Ctrl-C still
raises SIGINT; that is what an editor wants. `TioCbreak` turns off
canonical mode and echo and leaves the rest alone, which is what a menu
or a pager wants.

A change of attributes takes effect at a moment the caller chooses.
`TioNow` is TCSANOW, immediately. `TioDrain` is TCSADRAIN, after
everything already written has been sent. `TioFlush` is TCSAFLUSH,
after output drains and with pending input discarded.

Besides the attributes, a full-screen program changes a set of terminal
modes, each of which is an escape sequence and each of which has to be
undone on the way out.

| What | Sequence | Why a program wants it |
| --- | --- | --- |
| Alternate screen | `CSI ? 1049 h` | A screen of its own, with the cursor saved and the user's scrollback untouched underneath |
| Hide cursor | `CSI ? 25 l` | The program draws its own |
| Bracketed paste | `CSI ? 2004 h` | A pasted block arrives wrapped, instead of as a hundred keystrokes |
| Focus events | `CSI ? 1004 h` | The program can stop animating when nobody is looking |
| Mouse clicks | `CSI ? 1000 h` | Button presses and releases |
| Mouse drag | `CSI ? 1002 h` | The above, and motion while a button is held |
| Mouse motion | `CSI ? 1003 h` | The above, and motion with no button down |
| SGR mouse encoding | `CSI ? 1006 h` | Implied by every mouse setting; the older encoding packs a coordinate into one byte and breaks past column 223 |
| Synchronised output | `CSI ? 2026 h` | A frame written in several writes is presented in one and cannot tear |

A terminal's size is rows and columns, and sometimes also the width and
height of the text area in pixels, which is what an image protocol
needs in order to size a picture to the cell grid. The kernel reports
it through the `TIOCGWINSZ` request and raises SIGWINCH when it
changes.

| Quantity | Value |
| --- | --- |
| Fallback size | 24 rows by 80 columns, the same fallback `screen` and `tmux` use |
| Default wait for the byte after a lone ESC | 25 ms, keymap-nv's `ESCAPE_TIMEOUT_MS` |
| Default wait for anything at all | 50 ms |
| Default wait for a terminal's answer to a query | 1000 ms |
| Named flags | 12 |

Every function here that touches the machine performs input or output
and nothing else. None of them reads a clock: a wait is a poll with a
deadline in milliseconds, and a poll that comes back empty is the
deadline expiring.

## Install

```
novo pkg add termios-nv
```

## Example

A full-screen program's whole terminal handling. There is no restore in
it, because `with_screen` does the restoring.

```novo ignore
use tioattrs
use tioinput
use tioscreen
use tiosize

fn draw_until_quit(t: TioFd) -> Int [io]
    var r = tioinput.reader(t)
    var size = tiosize.size_or_default(t).size
    var quitting = false
    while not quitting
        // True once per SIGWINCH, and clears when read.
        if tiosize.size_changed()
            match tiosize.size_of(t)
                Ok(s)  => size = s
                Err(_) => size = tiosize.default_size()
        match tioinput.read_step(r)
            Err(_)   => quitting = true
            Ok(step) =>
                // Carry on with the reader the step answered.
                r = step.reader
                if step.at_eof
                    quitting = true
                // …draw at `size`, act on `step.events`…
    0

fn main() [io]
    // Install the SIGWINCH handler that sets the flag.
    tiosize.watch_size()
    match tioattrs.stdin_terminal()
        Err(_) => println("termios-nv: no terminal on stdin")
        Ok(t)  =>
            // Raw mode and the screen go on, the body runs, and both
            // are undone before this answers.
            match tioscreen.with_screen(t, TioRawKeepSignals,
                                        tioscreen.full_screen(), draw_until_quit)
                Ok(code) => println("${code}")
                Err(_)   => println("termios-nv: could not take the terminal")
```

This program is marked `ignore` because it needs the bodies this
release does not have: running it reaches a `todo()` and panics.

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: termios-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `tioattrs` | A descriptor known to be a terminal, the attributes as a value, the twelve named flags, the three raw modes, entering raw mode and putting it back, and writing bytes to the terminal. |
| `tiosize` | The window size in cells and pixels, the three places it can come from and which one answered, setting it, and the SIGWINCH flag. Also the escape-sequence query for a terminal that will not answer an ioctl. |
| `tioscreen` | What a full-screen program wants the terminal to be, what was actually turned on, and the sequences that turn it on and off. |
| `tioinput` | Reading the terminal: the key decoder, the three deadlines, a step of progress, and the query that waits for an answer without dropping what the user typed meanwhile. |

Roughly half of the package performs no input or output. The flag
arithmetic in `tioattrs`, the size values and the query bytes in
`tiosize`, every sequence builder in `tioscreen`, and the decoding in
`tioinput` all work on values the caller already holds, and are
testable with no terminal anywhere near the test. What performs is the
`isatty`, the `tcgetattr`, the `tcsetattr`, the two size requests, the
writes, the poll and the read.

## How to choose an entry point

**`tioscreen.with_screen` is the one most programs want.** It takes the
terminal, a raw mode, what the screen should be and the program's own
loop as a named function. Raw mode and the screen go on, the loop runs,
and both are undone in the right order before it answers.

**`tioattrs.with_raw` is the same guarantee without the screen**, for a
program that manages its own.

**`tioattrs.enter_raw` and `tioattrs.restore` are the two halves**, and
`tioscreen.enter_screen` and `tioscreen.leave_screen` likewise. They
are for a loop that owns the process and cannot be handed to anything —
a multiplexer's, an emulator's. `enter_raw` answers a `TioRestore`, and
holding one is what says the terminal has been changed and not yet put
back.

**`tioinput.read_step` waits and then decodes.** It is the read a pump
makes each turn. `tioinput.read_available` takes what is there without
waiting, and `tioinput.feed_bytes` decodes bytes a caller read itself.

**`tioinput.ask` writes a query and waits for the answer.** It is the
only way to learn whether a terminal supports a query, and it hands
back the keys that arrived while it waited.

**`tioscreen.screen_bytes` and `tiosize.write_size_query` build the
sequences into a caller's buffer** and write nothing. They are for a
program that batches its output into one write.

## The rules a user needs

1. **A program that enters raw mode and exits without restoring leaves
   the user a shell with no echo and no line editing.** The same is
   true of the alternate screen and the mouse. `tioscreen.with_screen`
   and `tioattrs.with_raw` take the program's loop as an argument and
   undo everything before they answer, on every path the loop can leave
   by, so there is no teardown to forget.
2. **The order matters, and `with_screen` fixes it.** Raw mode goes on
   first, with TCSAFLUSH, so keys typed while the program was starting
   are discarded rather than echoed across the first frame. The screen
   is left *before* the attributes are restored, so the shell's prompt
   is painted on the main screen under the mode the shell expects. And
   `tioscreen.screen_restore_bytes` undoes in reverse order, because
   `CSI ? 1049 l` restores the cursor position the alternate screen
   saved, and showing the cursor first would flash it where it is about
   to be overwritten.
3. **`TioScreenState` records what was turned on, not what was asked
   for.** A program that asked for the mouse on a terminal where the
   mouse was already on must not turn it off on the way out, because it
   did not turn it on.
4. **Nothing can cover a signal that ends the process without
   unwinding.** SIGKILL is nobody's to catch. For SIGTERM and SIGINT,
   the standard library's `process.shutdown_requested()` is a
   test-and-clear poll: ask it each turn and return, and returning is
   what runs the restore.
5. **The attributes are read once and written back verbatim.** Only the
   twelve named flags are overlaid. A package that rebuilt the
   structure from the flags it knew would silently drop the rest, and a
   user whose terminal had a non-default VERASE would find it changed.
6. **A `TioFd` is the only argument these functions take, and
   `tioattrs.terminal` is the only way to make one.** It asks the
   kernel. `tcsetattr` on a pipe or a closed descriptor is a silent
   no-op on some platforms and an error on others, so the check happens
   once, at the start, rather than three calls later.
7. **`TioNotATerminal` is the ordinary answer in a test harness or a
   build.** `tioattrs.is_terminal` is the probe a program starts with,
   so it can take the non-interactive path rather than fail.
8. **SIGWINCH is polled, not handled.** `tiosize.watch_size` installs
   the handler and `tiosize.size_changed` is a test-and-clear read, so
   the program's own loop decides when a resize matters. It is the same
   shape the standard library uses for SIGTERM.
9. **`size_changed` says a SIGWINCH arrived, not that the size is
   different.** A window manager delivers one for a drag that rounds to
   the same grid of cells. Compare with `tiosize.size_eq` before
   laying out again.
10. **A size has three sources and `tiosize.size_or_default` says which
    one answered.** `TIOCGWINSZ` first, which is right and cheap but
    unavailable over a serial line and stale inside a multiplexer that
    has not propagated a resize. Then `$LINES` and `$COLUMNS`, which a
    shell updates on its own SIGWINCH and nothing else does. Then 24 by
    80. A program drawing narrow on a wide terminal can say in its log
    which source it used.
11. **Asking the terminal for its size is not in that chain.** `CSI 1 8
    t` and the cursor-position probe are questions whose answers arrive
    as *input*, so the program must already be reading, must have a
    deadline, and must expect the answer to interleave with typing.
    `tiosize.write_size_query`, `tiosize.write_size_probe` and
    `tiosize.size_from_reply` are the pieces, and `tioinput.ask` is the
    waiting.
12. **A lone ESC is resolved by a deadline, and this package owns it.**
    keymap-nv has no clock: a `0x1B` is either the Escape key or the
    first byte of a sequence, and only the party whose read timed out
    can tell. `TioTimeouts.escape_ms` is that deadline.
13. **While a decoder is holding an ESC, the shorter deadline wins.**
    `tioinput.read_step` waits `escape_ms` rather than `read_ms` in
    that state. A hand-written loop that always waits its own tick
    leaves the Escape key unresolved for a whole frame.
    `tioinput.poll_deadline_ms` gives a caller running its own poll the
    same arithmetic.
14. **A step that timed out is not an error.** It is how a loop gets
    its tick, and a step that timed out with a pending decoder has
    already flushed it, so the Escape key is in `events`.
    `TioInputStep.at_eof` is the one to act on: a loop that ignores it
    spins.
15. **Replies and keys are two parsers over one byte stream.** A
    terminal's answers arrive through the descriptor the user is typing
    into. keymap-nv reports a sequence it does not claim as
    `KeyUnknownSequence(final_byte)`, which loses the parameters, so
    when reply-watching is armed `tioinput` runs ansi-nv's parser over
    the same bytes and answers typed replies alongside the keys. No
    byte belongs to both. Reply-watching is off by default.
16. **`tioinput.ask` hands back the keys that arrived while it
    waited.** A program that dropped them loses the first characters a
    fast typist types at startup. `TioAsked.reply` being `None` is the
    answer to "does this terminal support this query": a terminal that
    does not recognise one is silent about it.
17. **A short write is an error here.** `TioWriteFailed` carries how
    many bytes went, because a half-written escape sequence leaves the
    terminal in a state the caller has to reason about.
18. **A restore token is spent once.** `tioattrs.restore_is_pending`
    and `tioscreen.screen_is_pending` say whether it still is. A second
    restore does nothing rather than writing stale attributes over a
    terminal some later caller has since configured.

## What is not included

- **Terminal emulation.** Parsing what a *child* writes is
  [novo-vte](https://novo-lang.org/packages/novo-vte)'s work.
- **Key decoding.** keymap-nv does it; this package feeds it bytes and
  supplies the deadline it does not have.
- **Building escape sequences.** ansi-nv does that, and every sequence
  this package writes comes from there.
- **Opening a pseudoterminal.**
  [pty-nv](https://novo-lang.org/packages/pty-nv) does, and depends on
  this package for the slave's attributes and for `TioWinSize`.
- **Reading a clock.** See rule 12: every wait is a poll with a
  deadline. A program that wants a clock takes
  [timer-nv](https://novo-lang.org/packages/timer-nv).
- **Windows.** `termios` is a POSIX interface and the Windows console
  API is a different design, so a Windows port is a separate package
  rather than a branch inside this one.
- **Running on a microcontroller.** Every function here is a system
  call on a terminal descriptor. A device's UART is a different shape
  entirely.

## Related packages

- [ansi-nv](https://novo-lang.org/packages/ansi-nv) builds and parses
  escape sequences and touches no descriptor. This package owns the
  descriptor and writes what ansi-nv built.
- [keymap-nv](https://novo-lang.org/packages/keymap-nv) turns bytes
  into key, mouse, paste and focus events. It reports a held ESC as
  pending and resolves it when its host says the read timed out. This
  package is that host.
- [pty-nv](https://novo-lang.org/packages/pty-nv) is the other side:
  the terminal a program *creates* for a child, rather than the one it
  was started on.
- [novo-vte](https://novo-lang.org/packages/novo-vte) keeps the grid of
  cells a terminal emulator draws.
- `std.tui` in the standard library writes escape sequences straight to
  standard output and reports the terminal size. It is for a program
  that wants a coloured line, not one that takes the terminal over.

## Reference implementations

POSIX `termios` is the attribute model. Rust's `crossterm` is the
reference for splitting raw mode from screen state, Python's `tty` and
`termios` modules for the three raw modes, and `linenoise` for the
minimum a program actually needs. vim's `term.c` and tmux's `tty.c` are
where the ordering rules in rule 2 come from.

`crossterm` keeps its terminal state in process-wide variables with a
reference count, so a library that enables raw mode and a caller that
also does share one counter; here the state is a value and the
descriptor is in it. Python's `termios` exposes the attributes as a
list of integers whose indices the caller has to know; here twelve
flags are named and the rest is opaque. Neither models the screen state
at all, which is what `TioScreenState` is for.

## Tests

```bash
novo test --isolate tests/tioattrs_tests.nv   # 10 tests: the flags and the raw modes
novo test --isolate tests/tioscreen_tests.nv  #  8 tests: the sequences and the restore
novo test --isolate tests/tioinput_tests.nv   # 12 tests: the decoder, the deadlines, the replies
novo test --isolate tests/tiosize_tests.nv    # 10 tests: the sizes and the queries
novo test --isolate tests/tiohost_tests.nv    # 17 tests: the descriptor, the ioctls and the reads
```

The behaviour asserted comes from POSIX chapter 11 for the attributes
and the three `tcsetattr` moments, from xterm's `ctlseqs` for every
private mode in the table above, and from vim and tmux for the order
raw mode and the screen go on and come off in.

The four suites that need no terminal assert on values: that full raw
mode turns off the eight flags `cfmakeraw` turns off and leaves the
opaque bytes alone, that `TioRawKeepSignals` differs from `TioRawFull`
in exactly one flag, that the alternate screen is 1049 and not 47, that
every mouse setting carries the SGR encoding, that a lone ESC is held
rather than reported and shortens the next deadline while it is held,
and that the same bytes are a reply when reply-watching is armed and an
unknown key when it is not. `tiohost_tests.nv` is the half that opens a
terminal: a redirected standard input is not one, attributes off a pipe
are refused rather than guessed, and entering and leaving the screen
spends the state exactly once.

No test leaves a terminal changed. The tests compile today and fail at
run, each on the `not implemented: termios-nv.<module>.<fn>` panic that
is its body. That is the expected state of an interface release. They
turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `tioattrs.is_terminal`, `.terminal`, `.stdin_terminal`, `.controlling_terminal`, `.close`, `.fd_of` | no |
| `tioattrs.attrs_of`, `.set_attrs`, `.raw_attrs`, `.flags_of`, `.with_flags`, `.attrs_eq` | no |
| `tioattrs.enter_raw`, `.restore`, `.restore_is_pending`, `.saved_attrs`, `.with_raw` | no |
| `tioattrs.write_bytes`, `.drain`, `.flush_input`, and `TioError`'s `message` | no |
| `tiosize.default_size`, `.win_size`, `.size_eq`, `.size_is_valid`, `.cell_pixels` | no |
| `tiosize.size_of`, `.set_size`, `.size_from_env`, `.size_or_default` | no |
| `tiosize.watch_size`, `.size_changed`, `.size_watch_is_armed` | no |
| `tiosize.write_size_query`, `.write_size_probe`, `.size_from_reply`, `.in_band_resize_mode` | no |
| `tioscreen.full_screen`, `.inline_screen`, `.no_screen`, `.mouse_modes` | no |
| `tioscreen.screen_bytes`, `.screen_restore_bytes`, `.screen_is_pending` | no |
| `tioscreen.enter_screen`, `.leave_screen`, `.set_mouse`, `.set_cursor_visible`, `.with_screen` | no |
| `tioinput.default_timeouts`, `.reader`, `.reader_with`, `.watch_replies` | no |
| `tioinput.read_step`, `.read_available`, `.input_is_ready` | no |
| `tioinput.feed_bytes`, `.flush_pending`, `.is_pending`, `.poll_deadline_ms` | no |
| `tioinput.decoder_of`, `.with_decoder`, `.timeouts_of`, `.with_timeouts` | no |
| `tioinput.ask`, `.terminal_answers` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
