# mpflipdot

![A Hanover Displays flip-dot sign driven by mpflipdot](flipdot_example.jpg)

[Watch a video demo on YouTube](https://www.youtube.com/watch?v=nHq_gT024Bs).

A minimal [MicroPython](https://micropython.org/) driver for [Hanover Displays](https://www.hanoverdisplays.com/) flip-dot signs, aimed at a [Raspberry Pi Pico](https://www.raspberrypi.com/products/raspberry-pi-pico/) (or Pico 2) with an RS485 adapter. No dependencies beyond MicroPython itself.

> **Status:** early but working. Verified on a 20x14 and an 84x7 sign from a Pico W, and on the 20x14 from a Pico 2 W, with the Waveshare Pico-2CH-RS485 (see
> [Tested hardware](#tested-hardware)). Tested on MicroPython 1.29.0; the non-wireless Pico and Pico 2 should work fine too. See [PLAN.md](PLAN.md) for protocol notes and design.

## Features

- Configurable bus address, width, height (and upside-down mounting) per sign
- Clear / fill the display
- Start / stop the manufacturer's test mode (broadcast, address 0)
- Set a single dot on/off while retaining all others
- Set many dots (or the whole display) in one update
- Optional drawing helpers (`mpflipdot_draw.py`): lines of any width, rectangles and circles, each an outline or solid
- Optional text (`mpflipdot_text.py`): a built-in 5x7 font (A-Z, 0-9 and most punctuation, including brackets and `|`), text at any x, y position (or centred), scaling, and scrolling
- Everything is clipped to the sign, and `on=False` erases a shape or text instead of drawing it
- A minimum gap between frames, so the sign doesn't drop updates, and repeat frames are not sent at all

## Usage

```python
from mpflipdot import FlipDot, open_uart

uart = open_uart(1, tx=4, rx=5)          # any UART id / pins (this is channel 1 of the Waveshare board)
sign = FlipDot(uart, address=6, width=20, height=14, row_offset=1)   # see "Sign quirks" below

sign.clear()
sign.set_pixel(10, 3, True)
sign.set_pixels([(0, 0, 1), (1, 1, 1), (2, 2, 1)])   # one frame sent
sign.test(True)                                      # manufacturer test mode
sign.test(False)
```

Other options: `row_offset=1` if the sign's top row is bit 1 instead of bit 0, `flip=True` (sign mounted
upside-down; rotates the picture 180° so (0, 0) stays your top-left), `auto_show=False` (batch changes, then call `sign.show()`), `de_pin=Pin(n, Pin.OUT)` for RS485
adapters without automatic direction control, `min_gap_ms` / `quiet_ms` (see "Update rate" below), `aspect=1.25` (how much wider than tall `circle` draws, for signs whose dots are taller than wide; see Drawing helpers). Every change resends the whole frame, because the protocol has no
per-dot command; a frame identical to the last one sent is skipped (`show(force=True)` sends it anyway, `skip_unchanged=False` turns skipping off). `set_pixels` sends just one frame for the whole batch. Addresses are 0-15 (one hex digit on the
wire). Several signs on one bus: create one `FlipDot` per address sharing the same UART.

## Drawing helpers

`mpflipdot_draw.py` is installed alongside the driver and has plain functions that return `(x, y, on)` dots for
`set_pixels`, so a whole shape goes out as one frame:

```python
from mpflipdot_draw import line

sign.set_pixels(line(sign, 0, 0, 19, 13))                # width 1
sign.set_pixels(line(sign, 0, 0, 19, 13, width=3))
sign.set_pixels(line(sign, 0, 0, 19, 13, on=False))      # erase the same line
```

```python
from mpflipdot_draw import rect, circle

sign.set_pixels(rect(sign, 2, 3, 10, 6))                       # top-left corner x, y, then width, height (outline)
sign.set_pixels(rect(sign, 2, 3, 10, 6, solid=True))
sign.set_pixels(circle(sign, 10, 7, 5))                        # centre x, y, then radius (outline)
sign.set_pixels(circle(sign, 10, 7, 5, solid=True, on=False))  # a hole: switch the dots off
```

Rectangles and circles are outlines one dot thick unless you pass `solid=True`. A circle is `2 * radius + 1` dots tall
(`radius=0` is a single dot), and its outline is exactly the edge of the solid one. Lines, rectangles and circles are
all clipped to the sign, and dots later in a list override earlier ones, so you can combine shapes in one frame:
`set_pixels(list(rect(...)) + list(circle(..., on=False)))`.

**Round circles.** Dots on some signs are taller than they are wide, so a circle drawn in dots looks tall and thin. On the 84x7
sign an `aspect` of 1.25 looked round (1.0, 1.25, 1.5 and 1.75 were compared side by side). Pass it to one call,
`circle(sign, 42, 3, 3, aspect=1.25)`, or give the sign a default with `FlipDot(..., aspect=1.25)` and every circle on it uses
that. The circle stays `2 * radius + 1` dots tall and comes out about `aspect` times as wide; the default 1 is a true
dot-for-dot circle. The 20x14 sign was not compared.

Lines are clipped to the sign, so one that runs past the edge draws the visible part. Width is added across the line
along its minor axis, so a diagonal comes out about 0.7 of the width you ask for, and an even width puts the extra dot
below or to the right of the centre line.

## Text

`mpflipdot_text.py` (also installed with the driver) has a 5x7 font and works the same way as the drawing helpers: its
functions return `(x, y, on)` dots for `set_pixels`, clipped to the sign.

```python
from mpflipdot_text import text, text_centered, scroll, text_width

sign.set_pixels(text(sign, "HI!", 1, 3))              # top-left corner of the text at x=1, y=3
sign.set_pixels(text(sign, "OK", 2, 0, scale=2))      # whole-number scale: 2 gives 10x14 characters
sign.set_pixels(text(sign, "HI!", 1, 3, on=False))    # erase it again
sign.set_pixels(text_centered(sign, "HI!", 3))        # centred left to right by the dots drawn, top at y=3
scroll(sign, "HELLO WORLD", y=3)                      # once right to left, one frame per step, ends blank
scroll(sign, "BIG", y=0, scale=2, step=3)             # step = dots moved per frame (default 2)
width = text_width("HELLO", scale=1)                  # in dots, e.g. for your own layout
```

- Characters are 5 dots wide with a 1-dot gap, so a 20-dot-wide sign fits three at scale 1. Scrolling is how you show more.
- `text_centered` centres on the dots drawn, ignoring the blank columns some characters (`I`, `1`, `.`, `!`) leave at their edges; an odd dot left over goes on the right.
- The zero is a plain oval (no slash), so it matches the letter O.
- The font covers ASCII space to `]`: `A-Z`, `0-9`, space, and `! " # $ % & ' ( ) * + , - . / : ; < = > ? @ [ \ ]`, plus `|`. Lowercase is
  drawn as uppercase, and any other character is blank. The table is about 315 bytes.
- To draw text on a blank sign in one frame, make the sign with `auto_show=False`, then
  `sign.clear(); sign.set_pixels(text(...)); sign.show()`. With `auto_show=True` each of those calls sends its own frame.
  `scroll` handles that itself.
- `scroll_frames(sign, string, y, scale, step)` yields the dots for each frame, if you want to do something between frames.
  The frame rate is set by the driver's `min_gap_ms`, so about 3 frames a second on the 20x14 sign (fewer on a bigger one): a larger `step` scrolls faster.

## Examples

- `examples/basic.py`: clear, single dots, a batch update and test mode.
- `examples/text.py`: centred and positioned text, a counter, and scrolling text at scale 1 and 2.
- `examples/bounce.py`: a ball bouncing round the sign forever; Ctrl-C clears the sign and stops it.
- `examples/shapes.py`: loops through rect and circle demos (growing and nested rectangles, circles, a target, a bouncing ball, a hole punched in a solid block).
- `examples/lines.py`: loops through line demos (widths, a cross, spinning lines, a fan, lines clipped at the edges).
- `examples/demo.py`: loops through patterns (sweeps, boxes, checkerboard, diagonal wipe, sparkle, flash) as fast as
  the sign allows. Run it with `mpremote run examples/demo.py`; edit `W`, `H` and `address` first for your sign.
  Stop it with Ctrl-C, or `mpremote soft-reset` if it's detached.

## Tested hardware

| | |
|---|---|
| Board | [Raspberry Pi Pico W](https://www.raspberrypi.com/products/raspberry-pi-pico/) and Pico 2 W, [MicroPython 1.29.0](https://micropython.org/download/) (the Pico 2 W with bare MicroPython and nothing else installed, driving the 20x14 with `examples/demo.py` on channel 1). The non-wireless Pico and Pico 2 are untested but should work fine, as the driver uses nothing wireless-specific |
| RS485 | [Waveshare Pico-2CH-RS485](https://www.waveshare.com/pico-2ch-rs485.htm), channel 0 (UART0, GP0/GP1) or channel 1 (UART1, GP4/GP5; the examples use this, verified on the 20x14), sign data red on A, black on B. No direction pin needed |
| Sign | Hanover Displays Flip Dot 20x14, rotary switch SW2 on 5 (so `address=6`), 24 V / 5 A supply. Also an 84x7 (`address=6`, `row_offset=0`, same supply) |
| Verified | `fill`, `clear`, `set_pixel` (state retained), `set_pixels`, `test(True/False)`, `flip=True` (an L at (0,0) appeared in the opposite corner), `examples/demo.py` and `examples/lines.py` on the sign, `mpflipdot_draw.line`, `rect`/`circle` (`examples/shapes.py`), `examples/bounce.py`, `examples/text.py` (scrolling, positioned and centred text, every punctuation glyph added so far, ink-centring), skipping of repeated frames, 0.3 s between frames and the fixed `min_gap_ms=300` guard (a 0.2 s loop was slowed to 0.3 s and every row showed). The automatic gap (frame send time + `quiet_ms`, 284 ms here) is verified on the 20x14: a 14-row sweep run twice, a single dot against a full 14-dot column, 20 alternating checkerboards (all 280 dots flip every frame) and 20 random frames alternating 5 and 275 dots, each frame showing in order with none dropped. Timings were as predicted (14 frames in 3.70 s, 20 frames in about 5.4 s) |
| Verified on the 84x7 | manufacturer test mode, `fill`, `clear` (no blank-frame problem), `row_offset=0` (all 7 rows light), the automatic frame gap (7-row sweep, twice, no dropped frames; 7 frames in 2.8 s, about 0.47 s each) `scroll` of `HELLO WORLD` at full width (29 frames in 13.6 s, legible, no missing frames), and at rest: centred text (`HELLO WORLD`, `FLIP DOT`, `12:34`), text at x = 0 and against the right edge, text clipped at the right edge, and the punctuation glyphs; `examples/lines.py` and `examples/shapes.py` (with the demos' dot lists replaced by drawing straight into the buffer, which fixed a `MemoryError` on this wider sign); circles looked too tall at `aspect=1`, and `aspect=1.25` looked right. Also on channel 1 (UART1, GP4/GP5) from a Pico W: a test sequence (fill, row sweep, column sweep, four alternating checkerboards, shrinking box), left-aligned text, the four corner dots only (so all 84 columns and 7 rows are addressed), and the same text with `flip=True` (rotated 180 degrees). Also with the sign's backlight on (extra load on the same 24 V / 5 A supply): 20 alternating checkerboards, 20 sparse frames (one dot, then a full column), a 7-row sweep and the corner dots, 0.47 s per frame, no dropped frames or missing rows |
| Not yet tested | the 84x7 from a Pico 2 W, the non-wireless Pico and Pico 2, `de_pin`, several signs on one bus, and on the 84x7: the remaining demos (`demo.py`, which also builds dot lists, and `bounce.py`) |

## Sign quirks

These came out of testing on the 20x14 sign and may differ on other signs, so check yours:

- **Address is the rotary switch plus 1.** Switch on 5 means `address=6`. Address 0 is broadcast, and test mode always uses it.
- **Row offset.** On the 20x14 sign the top row is bit 1 of each column, not bit 0, so use `row_offset=1`.
  With `row_offset=0` the bottom row stays off. If a row is missing on your sign, try a different offset.
- **All-zero frames are ignored.** A frame with no dots set does nothing, so a plain "clear" would never take effect.
  When a frame would be all zero, `show()` sets one bit that lights no dot (bit 0 with `row_offset`, or the first
  spare bit above the rows), and the display clears as expected. On the 84x7 `row_offset=0` is right and `clear` worked with the same workaround.
- **Power it properly.** An underpowered sign behaves unreliably: it drops frames at random, or looks completely
  unresponsive (no clicking, nothing changes) even though the wiring and code are fine, which looks just like a
  signalling problem. In testing a 24 V / 5 A supply was reliable and a 24 V / 1 A supply was not. If frames go
  missing or the sign seems dead, check the supply before the wiring.
- **Update rate.** The sign needs a quiet moment after each frame. On the 20x14 sign frames 0.3 s apart were fine and
  at 0.2 s it skipped them. One 88-byte frame takes about 0.18 s just to send at 4800 baud, so that leaves roughly
  0.1 s of quiet. The driver works the same way for any size: by default (`min_gap_ms=None`) a send waits until the
  previous frame has finished going out plus `quiet_ms` (100 ms), which is about 0.28 s on the 20x14 sign (verified) and
  about 0.47 s on an 84x7 (176 bytes, 0.37 s to send; verified on the sign, though only about 2 frames a second). A fast loop is slowed down, not dropped, and the last update
  always lands. `quiet_ms=100` is a guess from the 20x14 sign, so raise it if a bigger sign still drops frames. A
  number for `min_gap_ms` is a fixed gap from the start of one send to the start of the next, and `min_gap_ms=0`
  turns the guard off.
- **USB-RS485 adapters on a computer** (for debugging): keep the port open for a few seconds after the last write,
  or the final frame can be lost.

## Hardware

- Raspberry Pi Pico W or Pico 2 W running MicroPython (the non-wireless Pico and Pico 2 should work fine too, but are untested)
- [Waveshare Pico-2CH-RS485](https://www.waveshare.com/pico-2ch-rs485.htm) (SP3485): channel 0 = UART0 on GP0/GP1, channel 1 = UART1 on GP4/GP5
- Hanover Displays flip-dot sign, 4800 baud serial, powered from a **24 V / 5 A** supply. A 24 V / 1 A supply made it drop frames
  at random or seem unresponsive, so don't skimp on current (see [Sign quirks](#sign-quirks))
- MicroPython **1.29.0** (tested on the Pico W and Pico 2 W). Older releases are untested; roughly 1.20+ is expected to work
  (`mip` install and `UART.flush()`, which is only used with `de_pin`), but that hasn't been verified.

### Pins used by the Waveshare board

The Pico-2CH-RS485 plugs onto the Pico's headers and passes every pin through, so all of the Pico's other GPIOs stay available to
your own code (sensors, buttons, LEDs and so on). Only these four GPIOs are taken, assuming both channels are in use:

| Channel | UART | TX | RX |
|---|---|---|---|
| 0 | UART0 | GP0 | GP1 |
| 1 | UART1 | GP4 | GP5 |

Keep GP0, GP1, GP4 and GP5 free of anything else, including other UART, I2C or SPI functions on those pins. If you only use one
channel, the other channel's two pins are free to reuse. The board has automatic direction control, so it needs no direction pin
(`de_pin` is only for other adapters). Apart from these four, it only draws power and ground from the Pico.

## Install

```
mpremote mip install github:simonprickett/mpflipdot
```

(verified on a Pico W: it installs `mpflipdot.py`, `mpflipdot_draw.py` and `mpflipdot_text.py` into `/lib`, and a script importing from them drove the sign), or, on the board: `import mip; mip.install("github:simonprickett/mpflipdot")` (not tried yet; needs Wi-Fi). Or just copy `mpflipdot.py` to the board; the drawing and text modules are optional extras you copy the same way.

Run the tests (CPython, no hardware): `python3 -m unittest discover -s tests`.

## Acknowledgements

Protocol details were worked out by studying [pyflipdot](https://pypi.org/project/pyflipdot/).

## License

[MIT](LICENSE)
