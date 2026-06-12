# Build Subsystem Details

## Python Compatibility

Enforcing Python 2 compatibility creates false positives. The Lustre build system and auxiliary scripts require Python 3.

- Assume Python 3 for all `.py` files (`contrib/`, `lustre/`, `Documentation/`)
- Do not report Python 2 incompatibilities (type annotations, `print()`, f-strings) as defects

## C Standard and Compilation Flags

Evaluating kernel C code against ISO C or user-space compiler defaults causes false defect reports on GNU extensions, data types, and aliasing.

- **Language standard**: Written in GNU C11 (`gnu11`). Do not flag GNU extensions (statement expressions, `typeof`, zero-length arrays, case ranges) as non-standard.
- **Unsigned char (`-funsigned-char`)**: Top-level `Makefile` enforces unsigned `char` on all architectures. Checking `char < 0` is dead code or incorrect logic.
- **Strict aliasing (`-fno-strict-aliasing`)**: Core kernel code disables strict aliasing; do not report type punning or pointer casting as undefined behavior in kernel space. Conversely, files under `{lnet,lustre}/utils/` assume standard `-fstrict-aliasing`; warn about type punning or incompatible pointer casting in tools.

## C language and type conventions

Unlike the aliasing rules above (which suppress false positives), these are
real issues to flag:

- **Don't cast pointer arguments to silence a warning.** Casting a pointer
  argument so a type-mismatch warning goes away hides a genuine mismatch and can
  corrupt memory. The fix is the correct type, not the cast.
- **Use real function-pointer prototypes, not `void *`.** A `void *` where a
  typed function pointer belongs defeats the compiler's argument checking and can
  cause stack corruption or crashes.
- **`BIT(n)` is for bitmasks only.** Using `BIT(n)` for an ordinal, count, or
  other numeric value is wrong; write the plain integer.

## Quick Checks

- **Python 3**: Do not enforce Python 2 compatibility on scripts
- **C standard and CFLAGS**: Respect `gnu11`, unsigned `char`, and domain-specific aliasing (`-fno-strict-aliasing` in kernel, `-fstrict-aliasing` in `tools/`)
- **Type/macro conventions**: flag pointer-argument casts that hide warnings,
  `void *` in place of function-pointer prototypes, and `BIT()` used as a number
