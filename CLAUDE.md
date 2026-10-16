# hydrogen — working notes

## What this repo is

`hydrogen.c` and `hydrogen.h` are [libhydrogen](https://github.com/jedisct1/libhydrogen)
flattened by hand into two files, so it can be dropped into a project without
carrying a source tree. That is the *whole* purpose: same library, one
translation unit.

It is **a vendor, not a fork.** Nothing here is meant to differ from upstream
except the flattening itself. If you find yourself wanting to change
behaviour, the change belongs upstream in jedisct1/libhydrogen, or in
[proton](https://github.com/mas-bandwidth/proton) — not here.

## The standing job: track upstream

Upstream moves; this file does not, unless someone carries the changes across.
Because the flattening is manual, **upstream commits do not merge** — each one
has to be read and applied by hand into the corresponding place in
`hydrogen.c`.

That means the risk here is silent staleness: the file keeps compiling and
keeps working, while fixes accumulate upstream that nobody notices. This is
crypto, so that matters more than it would elsewhere.

**This check now runs itself.** `.github/workflows/upstream-drift.yml` does it
on every push and pull request and once a day on a schedule, which is the beat
that matters: the change happens in *their* repo, so nothing here can trigger
it. A commit touching `impl/` fails the job, because those are buckets 1 and 2
below and have to be carried. Docs and CI commits are reported and pass.

**To check by hand, or to see what the job saw:**

```
git clone --depth 100 https://github.com/jedisct1/libhydrogen.git /tmp/libhydrogen
cd /tmp/libhydrogen && git log --oneline --since=<date the source last changed>
```

Use the date of the last commit that touched `hydrogen.c` here, not the last
commit in this repo — README edits do not move the source. The job derives that
date the same way, so a red run is telling you the carry is owed, not that
something here is broken.

**Then triage each upstream commit into one of three buckets:**

1. **Core** — touches `impl/hydrogen_p.h`, `impl/gimli-*`, `impl/core*`,
   `impl/hash*`, `impl/kx*`, `impl/secretbox*`, `impl/sign*`. These affect
   every consumer. Carry them.
2. **Platform RNG** — `impl/random/<platform>.h`. This flattening carries
   ALL platform branches (Linux, Windows, Linux-kernel, ESP32, STM32,
   Zephyr, nRF52832, RT-Thread, AVR, Arduino, pico-sdk), so these apply even
   though the projects using this file today are desktop and server. Carry
   them, and say in the review log which platform each affects.
3. **Docs, CI, badges, version strings in `library.properties`** — skip, and
   note that you skipped them.

**Record what you did.** A review log — every upstream release read, what was
in it, what was applied, and what was deliberately *not* applied and why — is
the only thing that makes the next check cheap. Keep exactly one copy of that
log, here, in the repo that owns the vendoring. Two copies of a review log is
two copies of one truth, and the copy nobody updates is the one people read.

## Review log

### 2026-08-13 — carried three, verified bit-exact

Baseline: the flattened source last changed **2025-10-31**; upstream had **26
commits** since. Carried:

**Core:**
- `4bcc4b4` **Optimize Gimli for aarch64.** Upstream adds a new file,
  `impl/gimli-core/aarch64.h`, and branches to it from `impl/gimli-core.h`. In
  a flattened vendor there is no new file: the body goes inline as a third
  `#elif` arm between the SSE2 and portable ones, keeping upstream's guard
  verbatim — `#elif defined(__aarch64__) || defined(_M_ARM64)`. It must be
  `#elif` and not a fresh `#ifdef`, or a target satisfying both guards gets
  two `gimli_core` definitions. `#undef GIMLI_ROUND` stays inside the branch,
  where upstream puts it. The inserted body is byte-identical to upstream's
  file; the patch is +71/−0, a pure insertion.

  **This is the permutation everything here is built on**, so a one-bit
  difference would change every hash, ciphertext and signature — silently, and
  only on the machines that take the new path. It was verified rather than
  reviewed: a differential harness drove hash at 13 lengths across block
  boundaries, an 8 KB hash, secretbox, deterministic sign and kdf through both
  the old (portable) and new (aarch64) builds on Apple Silicon. Identical
  output on every vector. The harness is worth keeping for the next such
  carry.

  Note the branch is **pure scalar C** — no `<arm_neon.h>`, no intrinsics.
  That is what makes it safe for proton's in-kernel build, which could not
  tolerate a userspace intrinsics header.

- `cb202c9` **Remove duplicate `hydro_secretbox_MACBYTES`.** Upstream removed
  the `impl/hydrogen_p.h` one; the flattened file carried both (lines 424 and
  1796). Removed the first, after checking the macro is not used between the
  two definitions — in a flattened file, ordering is a real constraint that
  does not exist upstream.

**Platform RNG:**
- `44fddd3` **linux_kernel** — `get_random_bytes(&ctx.state, …)` →
  `get_random_bytes(ctx.state, …)`. Type correctness, not a behavioural bug:
  `state` is `uint8_t state[gimli_BLOCKBYTES]`, so both spellings pass the
  same address. Carried because this is the `__KERNEL__` path, which is
  exactly what **proton** builds against.

**Not carried, deliberately:**
- The build-system hunks of `4bcc4b4` (CMakeLists.txt, Makefile,
  Makefile.arduino, Makefile.particle) add the new header to source lists. A
  single-file vendor has no source list. Nothing is lost; do not invent a
  counterpart.
- `5469506` + `319313d` pico-sdk. The flattening carries no pico-sdk branch at
  all, so this is a new platform rather than a fix to an existing one. Only
  worth adding if someone wants this on a Pico. **Still outstanding.**

### 2026-08-14 — the platform RNG backlog, cleared

The nine platform-RNG fixes deferred on 2026-08-13 are carried. They were
deferred as unreachable from a desktop, server or kernel build — true, and the
wrong reason to leave them: this flattening carries ALL platform branches, so
a consumer on any of them inherits whatever the vendor holds, and "nobody we
know builds for it" is exactly how a vendor rots.

| commit | platform | what it fixes |
|---|---|---|
| `24b3f52` | STM32L4 | an RNG init failure fell through to the entropy loop, which then spun forever on an uninitialised peripheral |
| `dd93aa5` | STM32F4 | the data-ready busy-wait had no timeout, so a stalled peripheral hung the caller |
| `f3ab14c` | STM32L4 | `continue` on a HAL error retried forever without advancing the entropy count |
| `d00b0c5` | RT-Thread | a stuck hardware RNG returning one word forever seeded the PRG deterministically, silently |
| `5eabaff` | ESP32 | `esp_random()` without `bootloader_random_enable()` is a PRNG never seeded from entropy — init failed OPEN |
| `2de57c6` | ESP32/Arduino | an unconditional `delay(10)` broke the plain-ESP32 contract, where `delay` is Arduino's |
| `1c23dbc` | nRF52832 | hashed `total_bytes` when only `available_bytes` had been read — hashing uninitialised buffer |
| `33bce64` | AVR | cleared the watchdog registers to zero instead of restoring them, so a sketch relying on the watchdog lost it |
| `cc776d5` | Zephyr | `zephyr/random/rand32.h` was removed upstream in 2024; the header is `random.h` |

Three of these are fail-open or hang bugs in an RNG, which is the worst place
to have one. None touches the desktop, server or kernel paths: the differential
harness (`gimli_diff.c`) produces identical output before and after, which is
what says so rather than inspection.

**Also filed upstream:** jedisct1/libhydrogen#165 — upstream's own `__KERNEL__`
branch suppresses `<stdint.h>` and supplies nothing back, so an in-kernel build
fails on the first `uint32_t` it declares. proton has carried those lines
locally to build at all; the PR offers them back so the divergence can end.
- CI permissions, badges, links, copyright year, `library.properties`.

**Also done this pass — proton parity.** `warnings_fuck_off` gained its
forward declaration and `(void)` parameter list here, which is what proton's
copy already had. That symbol is a **flattening artifact, not upstream**
(upstream has no such function), so changing it maintains the flattening
rather than forking. `hydrogen.c` and `proton/hydrogen.c` are now
**byte-identical**, so parity is checkable with a checksum instead of a diff
review. Both were re-vendored into flow, which builds clean and passes its
crypto tests.

**Open decision for Glenn.** proton's `hydrogen.h` carries a `__KERNEL__`
block — `linux/types.h`, `linux/string.h`, `linux/random.h` and the
`uint*_t` typedefs — that hydrogen's does not. Adding it would make the
headers byte-identical too, and there is a real argument for it: hydrogen's
kernel branch is currently incoherent on its own, since `hydrogen.c` has the
`__KERNEL__` branch and the `get_random_bytes` path while `hydrogen.h`
suppresses the libc includes and supplies nothing back. But that block does
**not exist upstream**, so adding it trades a hydrogen↔proton delta for a
hydrogen↔upstream one, which is exactly what the vendor-not-a-fork rule
forbids. Left alone. The clean fix is a PR to jedisct1/libhydrogen.

**Resolved 2026-08-15 — upstream answered.** jedisct1/libhydrogen#165 was fixed
upstream in `617036a` (2026-08-14, credits @gafferongames), and the fix is
carried in the entry below. The decision is moot: the block now exists
upstream, so carrying it is tracking, not forking. Upstream's form has no
typedefs — `linux/types.h` supplies `uint8_t` through `uint64_t` itself
(lines 109–114 of the kernel header) — so proton's typedef lines are
redundant and its header can re-vendor down to byte parity.

### 2026-08-15 — upstream answered #165: the kernel headers, carried

Baseline: one upstream commit since the 2026-08-14 pass.

**Core (the `__KERNEL__` build):**
- `617036a` **Include required header files for linux `__KERNEL__` builds.**
  The exact fix this repo's own log asked upstream for: the guard at the top
  of `hydrogen.h` suppressed `<stdint.h>`/`<stdlib.h>`/`<stdbool.h>` for
  in-kernel builds and supplied nothing back. Upstream adds an `#else` arm —
  `linux/random.h`, `linux/string.h`, `linux/types.h` — and `linux/types.h`
  provides the `uint*_t` typedefs, so nothing else is needed. Carried
  byte-identical (the inserted block diffs empty against upstream's).
  Verified: the userspace branch is untouched by construction (the new lines
  live entirely in the `#else` arm) and `clang -c hydrogen.c -Wall -Wextra`
  is clean on this bench. The kernel branch itself is not compile-testable
  here (macOS); proton's build is the real test and its re-vendor is the
  follow-up.

### 2026-10-16 — `264bf42` "Fix typos", carried: six comment sites

Baseline: the flattened source last changed **2026-08-15** (`hydrogen.h`;
`hydrogen.c` 2026-08-14). Three upstream commits since, one of them touching
`impl/`. The drift job has been red every day since 2026-09-28, the day this
commit reached upstream — eighteen consecutive scheduled failures, firing on the right
day for the right reason.

**Carried — `264bf42` (2026-09-28), comments only.** Every hunk in the code
files changes comment text; no executable line moves. Six sites apply to the
flattening, two of them in bucket 1 and four in bucket 2:

| upstream file | bucket | platform | change |
|---|---|---|---|
| `impl/random.h` | 1 (core) | all | the `hydro_random_uniform` clamping comment reworded — "size mod upper_bound == 0" → "size modulo upper_bound is zero", and the second sentence punctuated |
| `impl/x25519.h` | 1 (core) | all | ladder comment corrected, `// x3 = BC` → `// x2 = BC`. The comment was wrong and the code was right; this is the one hunk worth carrying on its own merit, since a wrong comment in the ladder is a future reader's trap |
| `impl/random/stm32.h` | 2 | STM32F4 | "RNG Peripheral enable" → "Enable the RNG peripheral" |
| `impl/random/ch32.h` | 2 | CH32V30x | the same reword, identical text |
| `impl/random/esp32.h` | 2 | ESP32/ESP8266 | "activated on ESP board" → "activated on the ESP board" |
| `impl/random/mbed.h` | 2 | mbed / mbedTLS | the `MBEDTLS_ENTROPY_BLOCK_SIZE` comment reflowed into three sentences, and "Forces mbedTLS" → "Force mbedTLS" |

`// RNG Peripheral enable` is one string appearing twice in the flattened file
— the STM32F4 arm at line 874 and the CH32 arm at line 989 — because upstream
changes both files to the same new wording. Both were replaced; that is two
hunks, not an over-broad substitution.

Verified three ways, because "it's only a comment" is exactly the claim that
should still be checked: every changed line is comment text, and the single
changed line that carries code (`hydro_x25519_mul1(x2, z2);`) is
character-identical before and after the `//`; `clang -E -P` output is
**identical** between the old and new file, so the preprocessor sees no change
at all on the compiled path; and the differential harness `gimli_diff.c`,
built against both versions, prints **byte-identical** output across hash at
13 lengths, an 8 KB hash, secretbox, deterministic sign, verify and kdf.
`clang -c hydrogen.c -Wall -Wextra` is clean. Behavioral risk: nil, and
measured rather than asserted.

**Not carried, deliberately:**
- The two `cmake/*` hunks of `264bf42` are build-system files. A single-file
  vendor has no counterpart; standing rule, nothing lost.
- `f45478a` (2026-09-25) "Fix stale documentation links", `README.md` only —
  bucket 3, skipped and reported.
- `9819a7c` (2026-08-26) "Add AI policy", `AI-POLICY.md` only — bucket 3,
  skipped and reported.
- `5469506` + `319313d` pico-sdk, still outstanding from the 2026-08-13 pass:
  a new platform rather than a fix to a carried one.

**The baseline moves itself.** There is no pinned upstream sha anywhere in the
repo; `upstream-drift.yml` derives its since-date from
`git log -1 --format=%cI -- hydrogen.c hydrogen.h`, so this commit *is* the new
baseline and the job goes green on it without a second file to remember.

**Worth knowing for the next pass:** the bucket-2 platform list above does not
name CH32 or mbed, yet the flattening demonstrably carries both branches and
both took a hunk here. Triage by grepping the flattened file for the upstream
text, not by reading that list.

**proton follows in the same hour.** Parity is byte-identical with a tolerated
delta of zero, so any edit here reddens `hydrogen-parity.yml` until the mirror
is copied across. The flow re-vendor (`flow_hydrogen.cpp`) is the remaining follow-up.

## Who uses this

The flattened file is vendored into the private **flow** networking library as
`flow_hydrogen.cpp` — byte-for-byte the same 3,116 lines. flow is where the
consequences of a stale vendor would actually be felt, so re-vendor there
after carrying anything across.

[proton](https://github.com/mas-bandwidth/proton) is a *different* derivation:
hydrogen built as a **Linux kernel module**, exposing kfuncs that XDP programs
call. Note what that is NOT — the crypto is not compiled to eBPF bytecode and
does not face the verifier; it is ordinary native kernel code. The discipline
there is the **kfunc contract**: BTF registration, the rules for pointers
arriving from a BPF program, `__arg` annotations, and lifetime.

That is also the real reason libsodium cannot follow this path while
libhydrogen can. It is not about the verifier — it is that libhydrogen already
builds inside the kernel (upstream carries `impl/random/linux_kernel.h`, and
this file has the `__KERNEL__` branch at lines 24 and 677), whereas libsodium
assumes libc, `mmap` and `mlock` and has no in-kernel build at all.

An upstream fix landing here may or may not apply to proton — check proton's
own build rather than copying the patch across. But upstream changes to the
`__KERNEL__` RNG path are the ones most likely to matter to it.
