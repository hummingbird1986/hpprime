# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Not a software project — this is the content tree of the **HP Connectivity Kit** (macOS), the sync
app for HP Prime calculators. Everything here is HP PPL (HP Prime Programming Language) source
wrapped in `.hpprgm` containers. There is no build system, no tests, no package manager.

## Directory roles

- `Content/Program/` — the program library (working directory). Loose `.hpprgm` files.
- `Content/Application/*.hpappdir/` — full apps: `.hpapp` (state) + `.hpappprgm` (code) + `.hpappnote` + `icon.png`.
- `Content/Note/*.hpnote`, `Content/Results/` — notes and a minimal `.xprimeproj` skeleton.
- `Calculators/HP Prime - Daniel/` — **mirror of what is actually on the device.** The Kit syncs
  `Content/` → here. Edit `Content/`, never this directory; a stale hand-edit here fights the Kit.
- `Backups/`, `Firmware/`, `Temp/` — Kit-managed, don't touch.

## The `.hpprgm` container

`.hpprgm` is binary with UTF-16LE source inside. Two variants exist in this tree:

1. **Plain (write this one).** 20-byte header, then the source as UTF-16LE with no BOM:
   `uint32 0x0000000C` · 12 zero bytes · `uint32 payloadByteLength` · payload.
   `Content/Results/test.hpprgm` is the canonical 274-byte example.
2. **Kit-compiled.** Magic `7C 61 8A B2`. The Connectivity Kit rewrites files into this form once
   they are opened/synced: source text sits mid-file, followed by compiled bytecode and stale heap
   junk (leftover strings from unrelated memory — ignore it, it is not part of the program).

Read either variant by decoding the whole file as UTF-16LE and slicing from the first `#pragma`/
`EXPORT` to the last `END;`. Write only variant 1 — the Kit recompiles on import. Grep and `Read`
are useless on these files; always go through Python.

```python
# read
import re
s = open(p,'rb').read().decode('utf-16-le','replace')
i = s.find('EXPORT'); src = s[i:s.rfind('END;')+4]

# write
import struct
b = src.encode('utf-16-le')
open(p,'wb').write(struct.pack('<I',0x0C) + b'\0'*12 + struct.pack('<I',len(b)) + b)
```

Keep a readable `.txt` beside any `.hpprgm` you author (see `BaseN.txt`) — it is the only
diffable form of the source.

## PPL conventions used across this library

Start files with `#pragma mode( separator(.,;) integer(h64) )`. `EXPORT` publishes a name as a
global callable from Home; unexported helpers stay private to the file. This tree prefixes private
helpers with `z` (`zQR`, `zMSK`, `zNOT`) so they never collide with the global namespace, which is
flat and shared across every program on the device — **name collisions between `.hpprgm` files are
the main failure mode here.** Before adding exports, scan the whole tree for the names you intend
to use (decode each file, slice from the first `#pragma`/`EXPORT` to the last `END;` **before**
matching `\bEXPORT\s+(\w+)` — without the slice the regex misses the first export in Kit-compiled
files, because binary junk abuts it and kills the `\b`).

**Never put a comment between a function signature and its `BEGIN`.** PPL drops the block and the
compiler then reports the failure at EOF ("error on line <last+1>"), pointing nowhere near the
real cause. Write the comment on the line *above* the signature. Trailing comments after a
complete statement are fine.

All user-facing text and comments are **English** (the user asked for this on 2026-09-23;
older files such as `Probability.hpprgm` still carry Chinese and have not been converted).

### The "callable API, not a runnable program" pattern

These programs are not meant to be run from the Program menu. The user calls individual exports
from the Home screen (usually **RPN mode**), so every export must stand alone and return a value to
the stack. `Probability.hpprgm` and `BaseN.hpprgm` are the reference shape:

- `EXPORT Var := default;` — module state as exported globals, editable from Home.
- `XxxS()` / `BSet()` — a settings function that pops `INPUT(...)` and returns the new state.
- Single-argument converters so RPN postfix entry works (type the value, then the name).
- Chinese `INPUT` prompts and help strings; `MSGBOX` + `KILL` for input errors.
- `Probability.hpprgm` extends names with the glyphs `◢` (≤), `◣` (≥), `◢◣` (between) —
  e.g. `NegB◢(x)`. These are literal characters in the source; preserve them byte-for-byte.

### `CSTMenu.hpprgm`

Defines `EXPORT CST := {...}` — the device's custom menu, a list of `{"page title", {"name", ...}}`
pairs, plus `CstMenu()` which renders it and `KEY K_Menu()` which binds it. **Adding exports to any
program means adding them to `CST` here**, or they are unreachable from the keypad. Note there are
two copies (`Content/CSTMenu.hpprgm` and `Content/Program/CSTMenu.hpprgm`) that have drifted; the
one the device sees is whichever the Kit last synced.

## Deploying

Open the Connectivity Kit, drag the file from `Content/` onto the calculator (or let it sync). There
is no CLI. Changes cannot be verified from here — the calculator or the emulator is the only test
rig, so state clearly what is unverified when handing work back.

**Home view only.** These programs are called from Home view; RPN does not exist in CAS view. Run one
in CAS view and **every call silently returns `0`** with no error. That looks exactly like a parsing
bug, so rule the view out before debugging anything else — it has already cost one round trip here.
CAS also defaults the alpha keys to lowercase, and PPL names are case sensitive.

In Home RPN both call forms work for a one-argument export: postfix (`"FF" BSTO`) and explicit
(`BSTO("FF")`). Zero-argument exports need their parentheses: `HEX()`, `BRCL()`.

## BaseN

`Content/Program/BaseN.hpprgm` (readable copy: `BaseN.txt`) is an HP-16C-style base/word-size
toolkit: 1-64 bit words, bases 2-64, unsigned / 1's / 2's complement, HP-42S shifts and rotates,
HP-16C logic ops. 70 exports. Full feature reference in `BaseN.md`. It represents values as a `{high32, low32}` pair because PPL reals
lose precision past 2^53. State lives in `BtW` (width), `BtB` (base), `BtM` (sign mode),
`BtS` (negative display), `BtC` (carry), and the X value register — which is **not** a variable
named `BtX`: it is the exported pair `BtXh`/`BtXl` (high 32 / low 32). There is no `BtX` anywhere in
the tree, so storing into it from Home does nothing; `BR*` will keep showing the real X.

**The X register is the spine of the design.** Every export that produces a word stores it in `BtXh`/`BtXl`
via `zPUT`/`zO`, and every mode-changing export (`W*`, `M*`, sign modes) returns `zO(zRC())` — the
stored value re-rendered under the new settings. That is what lets the user type a number once and
then switch base, word size or sign mode without re-entering it. `zRC()` truncates to `BtW` on
recall, so narrowing the word size discards high bits the way an HP-16C does. Parsing is split
`zVR(x, b)` (explicit base) / `zV(x)` (current base) so input can be taken in any base without
mutating `BtB`.

**A bare number is digits in a base, not a decimal value** (HP-16C model: in hex mode, typing `10`
means `10h`). `zVR` originally honoured its base argument only for string input and took a real
argument at face value, which made `BI2`/`BI8`/`BI16` identical to `BI10` — `BI2(25)` stored 25 and
printed `19` at `BtB=16` instead of rejecting `2` as a binary digit. Fixed 2026-09-23: the `TYPE 0`
branch now renders the integer's decimal digits back to a string and falls through to the string
parser, so digit validation and base weighting apply to every input form. Base 10 keeps a direct
fast path. Consequence: at `BtB=16`, `BSTO(255)` is 0x255, and `BVAL` (which returns a real decimal
value) is not chainable back into an entry function unless `BtB=10`.

**Checking a name against the firmware:** the Kit ships no reserved-word list, so fetch the official
guide (`https://literature.hpcalc.org/official/hpprime-ug-en.pdf`), run `pdftotext -layout`, and
harvest command names from its `Syntax:` lines, `Example:` lines and bare all-caps reference
headings — about 656 names. That was done on 2026-09-23 for all 75 BaseN exports: zero collisions.
`BIN`/`OCT`/`DEC`/`HEX` are **not** built-ins; base selection on the Prime is a setting from the
Integers menu, and the integer command family is `BITAND BITNOT BITOR BITSL BITSR BITXOR B→R R→B
GETBITS SETBITS SETBASE`.

**Design rule learned the hard way: a converter must not be asymmetric.** `BIN(x)`/`OCT`/`DEC`/`HEX`
read their argument in `BtB` but printed in a fixed base, so their output could never be fed back —
at `BtB=16`, `DEC(255)` then `HEX("255")` silently turned 255 into 597; at `BtB=10`, `HEX(255)` then
`DEC("FF")` raised "Bad digit F". They were removed on 2026-09-23. Every surviving path acts on
the stored X rather than re-parsing a string: `BI*` to enter, `BR*` to view, and the zero-argument
`BIN()`/`OCT()`/`DEC()`/`HEX()`/`MB(n)` to switch base. `BX(x,b)`
keeps the asymmetric shape deliberately and is documented as not chainable.

**`BtM` is 0-based (changed 2026-09-23):** `0` unsigned, `1` 1's complement, `2` 2's complement
(`MUNS`/`M1C`/`M2C` store those). It was 1/2/3. `BtS` was left 1-based on purpose — it is only ever
read as an `INPUT` choose index. Anywhere a 0-based global feeds a choose field or a list subscript
the code adds 1 (`BSet`, `BST`), and both clamp an out-of-range stored value instead of indexing
past the end of the list. **A calculator that already has `BtM:=3` from the old build keeps it**
until `M2C()`/`MUNS()`/`BSet()` runs; arithmetic is unaffected (3 falls through `zSM` to the 2's
complement path), but run one of them once after loading to normalise it.

**Base tags on output (added 2026-09-23).** `BtP` (0/1, `MTAG(n)` or the Tag field of `BSet()`)
makes every rendering carry its base: `H:FF`, `B:1010`, `O:377`, `D:255`, and `<base>:` for anything
else (`36:ZZ`). It is emitted in one place — `zS` returns `zPX(b) + s` — so every output path picks
it up at once, `BALL()` included. **`zVR` parses the tag back**, and the tag wins over the base it
was called with, which is what keeps output chainable: with `BtB=10`, `BSTO("H:FF")` is 255, and
`BSTO(BR16())` is an identity for any `BtB`. A tag is only recognised because of the `:` — no digit
string in any base contains one, so `B`/`D` as hex digits are never mistaken for tags.

**`BR*` is recall-only and takes no argument.** `BRCL/BR2/BR8/BR10/BR16/BRX/BALL` render the stored
X; they cannot see the RPN stack, so typing a value and then calling `BR2()` shows the *previous* X,
not what was typed. That is the design, not a bug (verified 2026-09-23 by porting `zVR`/`zS`/`zRC`
to Python and replaying `BSTO`/`BADD`/`BI2` sequences — every `BR*` tracked the latest store).
To view a value you have in hand, either store it first (`BSTO(x)`, `BI2(x)`, …) and then `BR2()`,
or use the two-argument `BX(x, b)`, which stores and renders in one call.

On 2026-09-23 the base-mode functions were renamed `MBIN`/`MOCT`/`MDEC`/`MHEX` -> `BIN`/`OCT`/`DEC`/
`HEX` (zero-argument, they act on the stored X), and `BN(x)`/`BSTO(x)` were both deleted — they were
byte-identical to each other, and loading a value is now the **Value field of `BSet()`**, the
Value field of `BSet()`, placed first in the dialog and read in the base set in that same dialog.

## The stale-allocation trap — read this before debugging a syntax error

**Overwriting an existing program on the calculator with a SHORTER one leaves the old tail behind,
and the calculator compiles your source plus that garbage.** The error is then reported at the end
of your program, so it looks like an unterminated block and it never moves however much you edit.

Observed 2026-09-23: `BaseN.hpprgm` on the device carried 522 characters after the last `END;`,
including `"Poiss◢","Poiss◣"` lifted from `Probability.hpprgm`, plus binary fragments. Six
consecutive builds failed with `syntax error` reported at EOF. The source was never at fault.

Symptoms that identify it:

- the reported line is always at or one past the end of the file, never at a real construct
- the line number tracks the file length, so it changes when you edit but never points anywhere
- a **new** program name compiles the same code fine

Checks, in this order:

1. Compare `Content/Program/X.hpprgm` with `Calculators/<device>/X.hpprgm`. Decode both as
   UTF-16LE and slice the source (first `#pragma`/`EXPORT` to the last `END;`); they must be
   identical. **Do not diff only that region and stop** — also look at what follows it.
2. **Trailing bytes alone prove nothing.** Every Kit-compiled file carries its bytecode after the
   source, so `len(s) - s.rfind("END;") - 4` is ~505-780 characters for *every* program on the
   device (surveyed 2026-09-23 across all 15). The signal is *readable text from another program*
   in that tail — the 2026-09-23 incident had `"Poiss◢","Poiss◣"` from `Probability.hpprgm` in it.
   Test it with `re.findall(r'[ -~]{3,}', tail)`: a healthy tail yields nothing or a stray comment.
   An earlier version of this note called any trailing content stale; that is wrong and it raised a
   false alarm on a perfectly good build.
3. Publish the same code under a brand-new name. If that compiles, it is the allocation.

Fix: delete the program on the calculator, then send it again. Or use a fresh name.

**A program returning `0` is not proof it ran.** An undefined name evaluates to `0` on the Prime,
so a program that failed to compile answers `0` to every call. Reading those zeros as "it works"
is what created a phantom "last known good build" here and sent the whole investigation backwards.

`{v, [2]}` in an `INPUT` field spec is **fine** — it restricts an edit field to type 2 (String),
and per the guide the editor then hides the double quotes, which is the only way to let a user
type `FF` bare in a dialog. An earlier note here blamed it for the compile failures; that was
wrong. Comments, the `zOK` helper, a list-valued `EXPORT` and the program name were all cleared
the same way.

Consequence: dialog value entry is not available, and quoted strings are needed for bases other
than 2/8/10/16. Based literals (`#FFh`, `#1010b`, `#377o`, `#255d`) need no quotes and are the
better answer when a user objects to typing them. Because the field is seeded with the current
X in the *old* base while it is parsed in the *new* one, `zOK(s,b)` validates it first — `zVR`
raises a message box and `KILL`s on a bad digit, which would abort the dialog after the settings
were already applied. `MB(n)`, `MUNS`, `M1C`, `M2C` and the `W*` family keep their prefixes.

That reintroduced the one program-level clash: `BIN` is exported by both BaseN and `Gray Code
Conversion.hpprgm`. The latter is not synced to the device, so it is latent, not live.
