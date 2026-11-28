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

**`real + "string"` uses the Home number format.** `BtW + "bit"` prints `64.00000bit` under
Fixed 5, not `64bit`. Wrap every number that ends up in user-facing text in `STRING(x, 1)`
(Mode 1 = Standard): `BST()`, `zPX`'s `<base>:` tag and the `MSGBOX` bounds messages all do
(fixed 2026-11-25). The tag one was a real bug — with `BtP:=1` and a base outside 2/8/10/16 the
output carried `36.00000:ZZ`, which `zVR` could not parse back.

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
pairs, plus `CstMenu()` which renders it and `KEY K_Menu()` which binds it.

The Base pages and the `BST()` status strip (optional 3rd page element) were removed on
2026-11-27: BaseN is used through Base16C now, so `CST` no longer reaches any BaseN export.
The Probability page and its 14 distribution pages were removed the same day: ProbCalc covers them,
so `CST` no longer reaches `Probability.hpprgm`. Perm/Comb (nPr, nCr, n!) moved to Main → `Perm`.
**Adding exports to any
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
`HEX` (zero-argument, they act on the stored X), and `BN(x)` was deleted as byte-identical to
`BSTO(x)`. **`BSTO(x)` is still exported** (an earlier version of this note claimed both were
deleted — wrong; check line 406 of `BaseN.txt` before repeating it).

**`BSet()` no longer loads a value (2026-11-25).** It is settings only: Word size (edit field),
Base, Sign, Negative, Tag. Value entry is `BSTO(x)` / `BI2`/`BI8`/`BI10`/`BI16` / `BI(x,b)` /
`BX(x,b)` from Home — there is no dialog and no keypad-menu path for it any more.

**Base and Sign are radio groups, not choose fields (2026-11-25).** Per the guide, an `INPUT`
field spec `{var, real, {pos}}` is a **check box**, and a `real` > 1 pools it with the *next*
`real`-1 check boxes into a radio group — so group members must be **consecutive in the variable
list**, independent of where `pos` puts them on screen. `BSet()` declares `{b2,4}` + three
followers at `1`, and `{mU,3}` + two followers, then positions them as two columns (Base left,
Sign right). Two further constraints from the guide: `pos` (`{start %, width %, line from 0}`)
must be given for **either none or all** fields, and a page holds **at most 7 lines** of controls
(more spill onto extra pages and `title` then takes a list). The current layout is 6 lines.

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

That form is no longer used: `BSet()` dropped its Value field on 2026-11-25, and with it the
`zOK(s,b)` helper and the whole hazard of a field seeded in the *old* base but parsed in the *new*
one. For value entry from Home, based literals (`#FFh`, `#1010b`, `#377o`, `#255d`) need no quotes;
other bases still need a quoted string. `MB(n)`, `MUNS`, `M1C`, `M2C` and the `W*` family keep
their prefixes.

That reintroduced the one program-level clash: `BIN` is exported by both BaseN and `Gray Code
Conversion.hpprgm`. The latter is not synced to the device, so it is latent, not live.

## Base16C (added 2026-11-26 as BaseNApp, renamed Base16C 2026-11-27)

`Content/Application/Base16C.hpappdir/` packages BaseN as a full-screen app. Readable source:
`Content/Program/Base16C.txt`. Rebuild `Base16C.hpappprgm`
from the `.txt` with the plain-variant writer above, plus a trailing `\0` (see below). The `.hpapp`
is a byte copy of `Base48.hpapp`; the icon is generated (38×38 RGB, stdlib zlib PNG writer, no PIL here):
  BaseN's palette (#141C28 field, #5A6E8C border), green `BASE`, yellow `16C` in 5×7 glyphs at 2×3 px
  (replaced the old `BASE`/`N` icon 2026-11-28). The `.hpappnote` is UTF-16LE + trailing `\0`, English.

- **Why not `BaseN`:** app and program names share the `Name.var` namespace on the device, so an
  app called BaseN would clash with `BaseN.hpprgm`. `Base16C` was checked against the tree and the
  user guide (no hits); both can live on the calculator together.
- **Self-contained.** It carries a private copy of BaseN's word engine (`Bt*` renamed to `BN*`) and
  exports **no functions**, only the app variables `BNw BNb BNm BNs BNp BNz BNg BNc BNo BNstk`.
  It does not need `BaseN.hpprgm` on the device, and its state is separate from BaseN's. Engine fixes
  in `BaseN.txt` must be ported by hand.
- **The engine copy has no `MSGBOX`/`KILL` paths.** `KILL` would end the app. Errors go to `zMsg`,
  which shows once in red in the status strip. `zPR` replaces `zVR`: entry keys only produce
  digits valid in the current base, so there is nothing to reject.
- **Screen:** rows 0-19 (system title bar and annunciators) are never drawn on. Strip y 20-37:
  `Base 16 · <page>` left, `|Size: 64|2's C G|T P Grp|` right (carry C in red, out-of-range
  G in green, no popup, as the HP-16C's annunciators; `Grp` = grouping). Panel BCD/HEX/DEC/OCT/BIN of X (no Tag prefix there), then the stack,
  menu at 220. Menu pages and numeric action codes are in `zPG`. Menu rearranged 2026-11-28:
  top `Conf. W.SZE Shift▶ Logic▶ Bits▶ IEEE▶` / `Code▶ UNDO` (a trailing `▶` marks a folder,
  CSTMenu's convention). **No base items on the menu — the user does not want them** (removed
  2026-11-27; wrongly re-added and removed again 2026-11-28). Base switching is keys + touch only. Shift: SL SR RL RR ASR /
  SLn SRn RLn RRn ASRn / RLC RRC RLCn RRCn LJ. Logic: AND OR XOR NOT / NAND NOR XNOR. Bits:
  SB CB B? #B / MASKL MASKR. Code: ▶Gray Gray▶ ▶ASCII ASCII▶. RMD is on Shift ÷ only (not
  HP-16C's position, where it is f 9; the user confirmed Shift ÷ on 2026-11-28). Base also
  switches with **Shift + letter key B / O / D / H** (key codes 15 / 29 / 17 / 22, the `zAK`
  letters; Shift + `,` is therefore OCT, plain `,` still swaps) and by **tapping a HEX/DEC/
  OCT/BIN panel row** (codes 102/108/110/116 via `zTAP`). **Shift W** (key 39, also digit 6)
  runs W.SZE (`zDO(4)`, 2026-11-28). `zDRAW` records
  each base's rows in `zRY` as `{y top, y bottom, base}` every frame, so wrapped rows (64-bit BIN)
  still hit; the working base's label is blue with a dot (`OCT •`; the user
  rejected a highlight band). BCD is not tappable.
  `W.SZE` (code 4) pops X as the word size, HP-16C WSIZE style. Conf. ("Base16C Setup") holds only
  Tag/Padding/Grouping/Carry, sign mode, Overflow (G) and Negative. IEEE (page 5) switches to hex
  on entry and restores the previous base on exit (`zOB`, checked in `zMAIN` by page change).
- **Entry line:** digits of the current base; `.`/EEX make a real (only f▶32/f▶64 take it);
  ALPHA / Shift ALPHA type letters (`zAK` key codes) as text, which only ASCII▶ takes.
- Entry follows Base48: `START()` → `STARTVIEW(-1,1)` and then the loop. `Num()` re-enters it.
  Home leaves the app to Home view, and Apps leaves it to the App Library.
- PPL runtime pitfalls found on the device: `INSTRING` on an empty string raises a runtime error
  (use `zHAS`, a MID scan); don't write `ᴇ` inside quotes, build it with `CHAR(7431)` (`zEE()`).
  An uncaught runtime error silently ends the app, so `zMAIN` wraps `zKEY`/`zTAP`/`zDRAW` in `IFERR`
  and shows "Error on key N" in the strip instead.
- **Never name a function parameter `i` or `e`** (imaginary unit / Euler's number): "Syntax error
  in program line N" at the function header (ConfMatrix `zGET(i)`, ProbCalc `zPW(b, e)` line 81).
  `LOCAL i` is fine.
- **At most 8 names per `LOCAL` statement.** 9 gave "Syntax error in program line N" pointing at the
  LOCAL line (seen twice: `zFD` line 340, `zSET` line 374). Split longer lists over two LOCAL lines.
  (The first time it was misdiagnosed as the names `e`/`bias`; they were renamed `ex`/`bs` anyway.)
- **× ÷ RMD (added 2026-11-27, Base16C only — BaseN has no BMUL/BDIV).** Keys × (40) and ÷ (35),
  Shift ÷ (codes 64/65/66, `zARI`; RMD left the menu 2026-11-28). Work on magnitudes from `zSM`:
  `zMUL` multiplies in 16-bit limbs (partial sums < 10^12, exact in PPL reals) to a 128-bit
  product; `zDV` is bitwise restoring division over `BNw` bits. HP-16C rules: × keeps the low
  word, G = overflow (signed range in 1's/2's), C untouched; ÷ truncates toward 0, C = remainder
  ≠ 0, G = 2's MIN/-1; RMD has Y's sign; X = 0 → "Divide by 0", stack untouched. Checked against Python big
  ints (all modes, widths 8-64). First `FOR … DOWNTO` in the tree (`zDV`).
- **C / G flags (2026-11-27, HP-16C flags 4 / 5).** `BNc` = carry/borrow (+ −, shifts, ÷
  remainder); `BNo` = out of range: + − via `zOV` (UNS: same as C; 1's/2's: signed overflow),
  × and ÷ in `zARI`, CHS of 2's MIN. RMD, logic, MASK, #B leave both alone. Not verified on device.
- **UNDO (2026-11-27).** Shift Del and top menu page 2 `UNDO` (code 3). `zMAIN` snapshots
  `{BNstk, BNc, BNo, BNw, BNm, BNb}` (`zSN`) before each event and, if the event changed it (`zSQ`),
  keeps the snapshot in `zU`. `zUN` restores `zU`; the replaced state becomes the new `zU`, so
  UNDO twice = redo. One level only. Entry line untouched. Shift Esc still clears the stack.
- **SB / CB (2026-11-27).** Codes 56 / 57: Y with bit n (X, 0 to `BNw`-1) set / cleared via `zSB`;
  X is consumed, no flags.
- **RRCn / MASKL / MASKR / #B (2026-11-27).** Codes 37 / 53 / 54 / 55. Shift page 2 is
  `SLn SRn RLn RRn` (same slots as their 1-bit forms on page 1), 3 is `RLC RRC RLCn RRCn LJ`; Logic page 2 is `NOR XNOR B? #B`, 3 is `MASKL MASKR SB CB`
  (bit ops grouped by function, not by HP-16C key position). MASK takes n (0-`BNw`)
  from X and replaces it (`zMR`); #B replaces X with its 1-bit count (`zNB`).
- Stack entries are `{h, l}`, or `{h, l, text}` after 32▶f/64▶f/▶ASCII (shown as text; ops drop the tag).
- **Append a `\0` (UTF-16 NUL) to the plain payload** (`(src + '\0').encode('utf-16-le')`, length
  field includes it). With it, the Kit's compiled copy ended exactly at the source (verified
  2026-11-26); without it the Kit appended stale text / `0xFFFF` filler. Still verify after compiling.
- After **every** plain rewrite, wait for the Kit to compile it (magic `7C 61 8A B2`) and check what
  follows the source: the Kit may append stale text or `0xFFFF` filler up to the next `\0\0` (seen
  even when the new source was longer; once it grew the text area to 64 KB). Blank that span with
  spaces in the compiled file (same length, so header sizes stay valid). Small edits that fit the
  space-padded region can be patched straight into the compiled file instead of a plain rewrite.
- The Kit accepts a plain-variant `.hpappprgm` and rewrites it to the compiled form within ~1-3 min;
  the device copy under `Calculators/` follows shortly after.

## ConfMatrix (added 2026-11-27)

`Content/Application/ConfMatrix.hpappdir/` is a 2x2 confusion-matrix app; readable source
`Content/Program/ConfMatrix.txt`, built the same way as Base16C (plain variant + trailing `\0`,
`.hpapp` copied from Base16C, generated 38x38 icon). Exports only `CMtp CMfn CMfp CMtn`.

- Matrix view: four pastel boxes (TP green, FN red, FP orange, TN blue) with a 1px border in
  the darker colour, on the same grid as the pale-lavender (`#DCD6F0`) header cells: columns
  x 0-38 / 40-178 / 180-319, rows y 38-54 / 56-135 / 137-217, 1px white gap between every cell
  (`zBOX`). Tap a box, its menu key, or arrows + Enter to
  enter a count (INPUT dialog). Clear zeroes all four.
- Num key (11) or the Num menu key: table Metric | Formula | Value (Accuracy, Precision,
  Sensitivity, Specificity, F1, FPR, FNR, NPV, Balanced acc., MCC); "n/a" on a zero denominator.
  Counts print with `STRING(v,1)` and percentages with `STRING(x,2,2)`, so the Home number
  format (often Fixed) does not leak in.

## ProbCalc (added 2026-11-27)

`Content/Application/ProbCalc.hpappdir/` is the probability app; readable source
`Content/Program/ProbCalc.txt`, built like Base16C (plain variant + trailing `\0`, `.hpapp` copied
from ConfMatrix, generated 38x38 icon). Self-contained (does not need `Probability.hpprgm`), exports
only state: `PCd` (distribution 1-14, order in `zDN`), `PCp` (parameter list per distribution),
`PCx`, `PCy` (upper bound for Between), `PCa` (α), `PCt` (tail 1 Left, 2 Right, 3 Two, 4 Between).
`ProbCalc` and the `PC*` names were checked against the tree and the guide: no hits.

- Landing page (`zM == 0`, where `START()` opens): F1-F6 are distribution names, 3 pages (6/6/2,
  Left/Right), no Back. Body `zDRAWL` lists all 14 in three boxes, one per page, coloured blue/green/orange; the
  current page's box is strong (double border), the others pale. Selected distribution: solid bar,
  white text. Left/Right on the landing page also moves the selection to the new column's first row. F-key or Enter opens a distribution. Touch
  (`zTAP`): tap a name in any box → select it (highlight, its box becomes the current page);
  tap the selected name again → open (Enter too). **Inside a distribution F6 is Back** to the landing page on every calc
  page; the Plot view has **no Back** (F6 blank, user asked 2026-11-27: mis-taps); Esc / Num return to Calc; Esc, Symb and Menu go there too.
- Calc view: strip + parameter/E/Var lines + result tape + entry line; menu pages (Left/Right):
  Set pdf P(X≤x) P(X≥x) P(a,b) Back / x-L x-R p-val Tail α Back / Crit CI Test · Clear Back (no x= on calc pages: typed number + ENTER sets x) (Plot only on the Plot key). A typed number is
  the argument; with no entry the key uses the current value (`PCx`, `PCa`, a/b) — **no dialogs** for
  single values (user asked 2026-11-27; `0.05 α` is the way to set α). Dialogs remain only for
  multi-field input: Set, CI, Test, and the Plot a,b= key. `P(a,b)` is `a ENTER b P(a,b)`.
  Up/Down switch distribution and reset x to the critical value at α for the current tail.
  **Tail changes only by the Tail key / Plot menu** (and P(a,b) → Betw, Test: Betw → Two). P(X≤x) P(X≥x) x-L x-R
  leave it alone (changed 2026-11-27, so a Two-tailed test survives table lookups); ◢ ◣ compute
  zCDF / zSF directly, not zPV. p-val takes a typed number as x (as b under Betw).
  x-L / x-R (quantile from a left / right tail p) are labelled by `zQS()`: z (Std Normal), t, χ², F,
  else x — also in the tape (`z-R(0.025)`).
  With no typed p they take the **last probability on the tape** (`zLP`, set by `zADP` from
  P(X≤x)/P(X≥x)/P(a,b)/p-val/Test p/α key), falling back to α; no dialog.
  p-value tape rows (p-val key, Test) are labelled by `zPLB`: `p = P(X≤x)`, `p = P(X≥x)`, or for
  Two `p = 2·P(X≥x)` / `2·P(X≤x)` (the smaller tail).
  The red strip message (`zMsg`) is for errors only (bad input / parameters); the Tail key sets
  no message — its label and the `<tail> · α` line already show the new tail.
  Under Betw the calc view shows `a = <PCx>   b = <PCy>` top right instead of `x = …`.
- Plot view (Plot key): pdf curve (bars if discrete), shaded tail: **red if p ≤ α (reject), green if not
  (keep H0)**, Between always blue; α critical values as dashed purple lines. Menu Left/Right/Two/Betw/a,b=/(blank) (a,b= only under Betw: a dialog for both bounds; otherwise typed number + ENTER sets x) (α is on calc page 2);
  arrows move x (Up/Down ×10). Two-tailed: Crit uses α/2 per tail and the screen shows
  `α/2 = …` (`zAT`); p = 2·min(P(X≤x), P(X≥x)) is compared with α (same test); the far tail is shaded
  from the quantile holding the same probability, so for discrete distributions the shaded area
  can differ slightly from p.
- CI (code 15, `zCI`, added 2026-11-27): two-sided 1-α interval, ignores the tail setting. Normal /
  Std Normal → z-interval for μ (σ seeded from the Normal's σ), t → t-interval (df = n-1, not the t
  parameter), Binomial / Bernoulli → proportion; on other distributions the key is blank and
  inert (`zCIOK` in `zMENU`/`zCODE`). Dialog
  seeded from the last inputs (`zCIm zCIs zCIn zCIp`, not exported); a typed number is the mean / p.
  Adds two tape rows: `MoE z* = …` then `95% CI [lo, hi]`.
- Test (code 16, `zTEST`, shown for Normal / Std Normal / t / χ² / F via `zKOK`): tests from
  summary data. N/StdN: `CHOOSE` list 1-Z, 2-Z, 1-P, 2-P; t: 1-T, 2-T (pooled/Welch choose field
  `{pl, {"Pooled","Welch"}}`), Paired; χ²: one-variance test straight away; F: two-variance test.
  Each test sets `PCx` = statistic and calls `zTFIN`, which switches to the statistic's
  distribution (z tests → Std Normal; t/χ²/F set df, Welch df fractional), Between → Two, and adds
  4 tape rows: header (SE / sp / p / df), statistic, p for the tail, 1-α CI (μ, μA-μB, p, pA-pB,
  μd, σ², σA²/σB²). 1-P / 2-P take counts x (statistic uses p0 / pooled p, CI unpooled).
  Last inputs per test in `zTI` (index = test code 1-9). 2-T verified on the device 2026-11-27.
- Unverified on the device as of 2026-11-27 (Kit was not running when it was written).
