# Logical correctness audit

Date: September 13, 2026.

## Scope and bottom line

This audit covers `reconstruct.py`, `main.tex`, the supplied artifacts, and
selected paths in `iboot_dec_24a435.bin`. The input SHA-256 is
`e1b9aa20c3d00b7eeadecfb1dd04963de9a80e75830cbc816ede8aed000a0301`.
The original analyzer, report, binary, and PDF were not modified.

Findings comprise two report/reproducibility problems, three reproducible
latent cross-reference defects, and two firmware behaviors requiring further
investigation. The latent defects were reproduced using synthetic instruction
sequences through the analyzer's actual cross-reference loop; they are **not
demonstrated corruptions of this image's existing cross-reference records**.
The firmware observations are **not confirmed exploitable vulnerabilities**.

This is not a claim to have found every logical bug in the bootloader. The
automated sweep covers the apparent primary text region, not verified execution
paths, all embedded payloads, or all runtime states. No firmware was executed.

## Confirmed report and reproducibility problems

### A1 — High: the documented build cannot produce the report

Location: `main.tex:274`, `main.tex:280`, `main.tex:283`, and
`reconstruction/validation.txt:7`.

`main.tex` unconditionally includes `reconstruction/behavior.pseudo`, but that
file is absent. Running `reconstruct.py` does not create it: there is no writer
for that artifact. A fresh build with

```sh
mkdir -p .audit
latexmk -pdf -outdir=.audit -interaction=nonstopmode -halt-on-error main.tex
```

fails at line 280 with `No verbatim file reconstruction/behavior.pseudo`.
The output-directory option only isolates build products; it does not change
the report's input paths.

The saved validation record is also stale relative to the supplied deliverable:
it describes a successful ten-page PDF, whereas the actual `main.pdf` has nine
pages. Its extracted pseudocode section contains the introductory paragraph,
then immediately advances to the reproducibility section, without pseudocode.
Neither that PDF nor the historical build log proves the current source builds.

Seven other advertised inventory artifacts are absent from the supplied
`reconstruction/` directory: `linear_disassembly.txt.gz`, `branches.json`,
`string_xrefs.json`, `address_xrefs.json`, `pointers.json`, `strings.tsv`, and
`candidate_starts.json`. Unlike the pseudocode, these were successfully
regenerated in an isolated audit directory.

Recommended correction: restore the reviewed pseudocode or remove the claim
and mandatory inclusion. Then perform a fresh build and update validation
against that exact PDF. Do not substitute an empty file just to make the
build succeed.

### A2 — Medium: the claimed startup clear is not established by its evidence

Location: `main.tex:99`.

The pair at file offsets `0x390/0x398` does contain the linked addresses for
image-relative `[0x355908, 0x358000)`, and those stored bytes are zero. However,
the statement that startup clears this region is not established by the entry
sequence. Its three clear loops load different literal pairs:

| Load instructions | Literal offsets | Nominal physical interval |
| --- | --- | --- |
| `0x1a0/0x1a4` | `0x3c8/0x3d0` | `[0x1fc044000, 0x1fc048000)` |
| `0x1b8/0x1bc` | `0x3a8/0x3b0` | `[0x1fc000000, 0x1fc040000)` |
| `0x1d0/0x1d4` | `0x3b8/0x3c0` | `[0x1fc427a40, 0x1fc44ddd0)` after bias subtraction |

No literal-load reference to `0x390` or `0x398` was found in the swept region.
There are other address references to the region boundaries; their existence
does not itself prove a store or clear operation. A later indirect/dynamic
clear is not ruled out by this audit.

Recommended correction: retain the observed literals and zero bytes, but mark
runtime clearing of this particular region unresolved until a write path is
traced. Distinguish stored contents from actions performed by the program.
The last nominal interval also has the rounding issue described in F1.

## Reproduced latent analyzer defects

These concern `reconstruct.py:145` through its cross-reference loop. The tests
executed that loop extracted from the Python AST, rather than reimplementing
its logic. LLVM decoded the test words. Each sequence starts at file offset
zero, with enough trailing bytes to keep the proposed target inside the blob.
The hash guard was not disabled in the normal analyzer: these were isolated
tests of the loop, not runs against a substituted firmware image.

### L1 — Medium, latent: memory writeback does not invalidate the page value

Location: `reconstruct.py:168`.

```text
90000000  adrp x0, #0
f8008401  str x1, [x0], #8
91040002  add x2, x0, #256
```

The loop emits target `0x100`. The store's post-index update changes `x0`, so
the actual address formed by the ADD is `0x108`. The clobber heuristic only
looks for a first register operand and misses the updated memory-base register.
Pre-indexed memory operations have the same modeling gap.

Recommended correction: invalidate the tracked value when a memory operation
writes back to its base, or track that adjustment explicitly. Merely expanding
the list of store mnemonics exempted from destination checks will not fix it.

No emitted reference crossing a detected pre/post-index update of its tracked
base was found in this image's 10,203 address-reference records.

### L2 — Medium, latent: authenticated transfers are missing from stop conditions

Location: `reconstruct.py:166`.

```text
90000000  adrp x0, #0
d63f083f  blraaz x1
91040002  add x2, x0, #256
```

The loop emits target `0x100` despite crossing an unresolved call, after which
the value of `x0` has not been established. The `BLRAAZ` instruction word was
taken from this image at `0x33270`; that does not mean this entire synthetic
sequence occurs there.

The stop list includes `braa` and `blraa`, but omits `braaz`, `brab`, `brabz`,
`blraaz`, `blrab`, `blrabz`, and `retaa`, among other transfers. A jump or return
can make the following instruction unreachable through the assumed path; an
unresolved call can invalidate the tracked value.

Recommended correction: classify control transfers comprehensively and stop
the local matcher at unresolved transfers, rather than relying on a short
mnemonic list. Treat undecoded instructions conservatively as well.

No emitted reference crossing these detected transfer forms was found in the
supplied image's regenerated address-reference records.

### L3 — Low, latent: register encoding 31 conflates XZR with SP

Location: `reconstruct.py:149`, `reconstruct.py:155`.

```text
9000001f  adrp xzr, #0
910403e0  add x0, sp, #256
```

The loop emits target `0x100`, but the ADRP discards its result and the ADD
uses the stack pointer. Matching the numeric register fields alone incorrectly
joins two different operands.

Recommended correction: reject ADRP destination encoding 31 for this matcher
and preserve the instruction-specific distinction between zero and stack
register operands. This synthetic result is not evidence that any current
reported string address is wrong.

## Firmware observations requiring validation

### F1 — Potential memory-boundary issue: startup clears 16 bytes beyond its end literal

Evidence: `reconstruction/selected_disassembly.txt:119`; instructions
`0x1d0` through `0x1f0`, and literals `0x3b8/0x3c0`.

After bias subtraction the loop starts at physical `0x1fc427a40` and compares
against `0x1fc44ddd0`. Each iteration performs two 16-byte stores before
testing whether the incremented pointer is below the end. The nominal length
is `0x26390`, which is 16 modulo 32. Consequently it performs 4,893 iterations
and writes `[0x1fc427a40, 0x1fc44dde0)`: **16 bytes beyond the nominal end**.
The last iteration begins at `0x1fc44ddc0`; its second store starts at the
nominal end itself.

This is a concrete instruction-level observation, not proof of harmful
out-of-bounds access. The extra bytes may be reserved alignment padding.
Establish the actual allocation/mapping and ownership of that interval before
calling it a memory-corruption bug. The report should nevertheless distinguish
the nominal literal boundary from the actual rounded write endpoint.

### F2 — Potential countdown bug: multiplication wraps at input 4,295

Evidence: `reconstruction/selected_disassembly.txt:2154`, especially
`0x4fe8`, `0x5004`, and `0x5008`.

The returned environment value is multiplied by 1,000,000 in a 32-bit
register, then compared as a zero-extended threshold against a 64-bit elapsed
counter. The threshold is therefore

```text
threshold = (bootdelay * 1,000,000) modulo 2^32
```

For input 4,294 it is 4,294,000,000; for 4,295 it wraps to 32,704. Input
67,108,864 produces zero. If such values can reach this path, increasing the
configured delay can drastically shorten the countdown. This does not require
assuming a particular timer unit. The strict unsigned comparison and polling
interval still affect the actual observed duration.

The getter at `0x16190c` returns the stored numeric field or default without a
local maximum check, and the displayed countdown path has no corresponding
clamp. This audit does not establish all constraints imposed when environment
values are created or changed, or whether large values are supported inputs.

Recommended investigation: trace those upstream constraints and the intended
delay range. If large values are supported, use checked wider arithmetic or
reject values outside the supported range before multiplication.

## Checks that passed and important non-findings

- The hash and 3,832,328-byte input size match the report.
- The analyzer completed its sweep of 550,710 words in `[0, 0x219cd8)`.
  This is automated coverage, not manual verification of 550,710 instructions.
- All 170,101 retained immediate-branch targets agree with the relative
  operands printed by LLVM. This checks the analyzer's arithmetic against its
  decoder, not an independently implemented disassembler or reachability.
- All 2,530 recorded string references match the addressed bytes and NUL
  termination. Byte agreement alone does not prove register liveness.
- The saved metrics, command records, entropy data, and selected disassembly
  reproduce semantically; observed text differences are terminal newlines.
- All 15 command candidates match their raw records, and the shared descriptor,
  `reboot`/`reset` alias, and `saveenv` branch agree with the local bytes.
- The cited recovery and subsystem string anchors match the regenerated
  references. The `memboot` 512-MiB local maximum and separate permission/image
  failure paths described for `go` match the sampled instructions.
- The initializer start/end literals are equal; its stored-value zero-iteration
  conclusion is supported.
- The shared helper at `0x81434` checks address-addition carry and range bounds
  on its range-checking path. It also has a flag-controlled allow path whose
  policy provenance remains unresolved. Neither its name nor that flag alone
  establishes a security-policy bypass.
- The `firmware` handler narrows `filesize` to 32 bits at `0x5568`, but uses that
  same narrowed size for the subsequent gate and processing call. Without a
  supported-input contract or mismatched downstream use, this was not promoted
  to a confirmed bug.
- The 56-byte relocation/file-length discrepancy is already explicitly
  qualified in the report; it was not counted again as a new proven defect.

## Remaining coverage limits

Function ownership, indirect-call resolution, complete environment/command
dispatch, authentication state transitions, hardware effects, concurrency,
embedded firmware, and the ultimate handoff were not exhaustively verified.
Literal pools and data can decode as instructions. The three latent analyzer
defects should be corrected before extending confidence to new patterns, but
their correction alone would not make the analysis path-sensitive or complete.

The next discriminating evidence is the allocation extent for F1 and the
upstream accepted numeric range for F2. Until that is obtained, report these
as conditional firmware risks rather than established exploitable bugs.