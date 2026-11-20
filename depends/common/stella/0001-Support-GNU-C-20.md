# 0001-Support-GNU-C-20.patch

Notes for whoever rebases this patch onto a newer Stella. They cover why it exists, what it changes, how the current version was checked, and the prompt it was made from.

## Why it exists

Stella master builds as C++23. Kodi's Jenkins builds this add-on with toolchains that can't, and Jenkins can't be changed:

- AppleClang 15.0 (Xcode 15.2) with a macOS 11 deployment target. It doesn't accept `-std=c++23`, and its libc++ either lacks some C++20 library features or only allows them from macOS 13.3.
- The Android NDK r28c used for Kodi 22, whose libc++ has the same gaps.

The patch builds the libretro core with `-std=gnu++20` on every platform. The version here applies to Stella `da5a002c318843bea1d233932508b9439798de0d` (see `stella.txt`).

## What it changes

It touches 100 files. The core's behaviour is meant to stay exactly the same, and the checks below found no difference.

C++23 language and library features, replaced with C++20 equivalents:

- `#elifdef` becomes `#elif defined(...)` (6 sites).
- The `UZ` literal suffix becomes an explicit `size_t{...}` (71 lines).
- Lambdas with a trailing return type but no parameter list, such as `[&] -> string_view {`, get an empty `()` (4 sites).
- `std::to_underlying` becomes `BSPF::toUnderlying` (42 sites).
- `std::unreachable` becomes `BSPF::unreachable` (16 sites).
- `std::byteswap` becomes `BSPF::byteswap` (4 sites).
- `basic_string::contains()` and `string_view::contains()` become a `find() != npos` check (3 sites).

Library features the libc++ of Xcode 15.2 lacks, or only allows from macOS 13.3:

- `std::format` becomes `BSPF::format` (207 call sites). It's a small formatter in `src/common/bspf.hxx` that supports the format specifiers Stella uses.
- Floating-point `std::from_chars` and `std::to_chars` go through `BSPF::parseFloat` and `BSPF::formatFloat`. These use the standard functions where the platform has them, and a fallback where it doesn't.
- `basic_stringstream::view()` becomes `str()` (4 sites).

One compiler gap:

- `BitmaskEnum` arguments are deduced through the class template, because Clang before 19 can't deduce them through an alias template.

The new helpers all live in `src/common/bspf.hxx`, so most of a rebase is finding new uses of the standard names and pointing them at the helpers.

## Known difference

The fallback in `BSPF::formatFloat` only runs where the platform has no floating-point `std::to_chars`, which in practice means Xcode 15.2 with a macOS 11 target. It always round-trips, but it isn't always the shortest form. In a differential test of 3.3 million values against `std::to_chars`, 49 came out one digit longer than the shortest representation, and none differed in any other way. Where the standard function exists it's used, and it matched in every case.

## How this version was checked

On `da5a002c3`:

- **Applying it:** `patch -p1` applies it to the tarball layout the add-on downloads with no fuzz, and the patched tree is identical to the branch it was made from.
- **gcc build:** g++ builds the patched tarball with the add-on's Linux recipe. All 153 compiles use `-std=gnu++20`, with no warnings, and the core has no undefined symbols outside libc, libstdc++, libm and libgcc. A clang++ build also succeeds.
- **Syntax checks:** every translation unit (152) passes with g++ and clang++ at `-std=gnu++20`. It also passes with clang++ against libc++ 16 headers with macOS availability markup for 11.0 and 10.13, which stands in for Xcode 15.2, and against libc++ 19.
- **Formatter tests:** a differential test of `BSPF::format`, `parseFloat` and `formatFloat` against `std::format`, `std::to_chars` and `std::from_chars`, plus a compile-time check that every `BSPF::format` call site's format string matches its arguments.
- **ROMs loaded from memory, the way Kodi loads them:** for generated 2K and 4K images, and for 41 synthetic images across 21 bank-switching types (2K 3E 3F 4K AR CV DPC DPC+ E0 E7 F0 F4 F4SC F6 F6SC F8 F8SC FA FA2 FE UA), the core reports the same size, MD5 and cartridge type as unpatched master built as C++23. With the wall clock pinned so the random seed matches, 600 frames of video and audio hash identically too. This was also checked with the float fallbacks forced on.
- **Jenkins:** green on all six platforms on the first push.

## Rebasing onto a newer Stella

1. Update `stella.txt` to the new commit and apply the patch with `patch -p1` to that commit's tarball. Fix any rejects by hand, keeping the same replacements.
2. Look through the code Stella added since for anything newer than C++20, including:
   - `#elifdef`, `#elifndef` and integer literals ending in `uz`/`UZ` or `z`/`Z`;
   - `std::to_underlying`, `std::unreachable`, `std::byteswap` and `.contains(` on strings;
   - `std::format`, `std::print` and `std::println`;
   - floating-point `std::from_chars`/`std::to_chars`, and `.view()` on string streams;
   - `std::ranges::to`, `std::ranges::contains`, `std::ranges::fold_left`, `std::expected`, `std::flat_map` and `std::mdspan`;
   - `if consteval`, `auto(x)`, explicit object parameters (`this auto&`), multidimensional `operator[]` and `static operator()`.
3. Build with `-std=gnu++20` using g++, clang++ with libstdc++, and clang++ with libc++. Also compile against libc++ 16 headers with `-mmacosx-version-min=11.0`: that catches the library features Apple only allows on newer macOS, which a Linux build won't show.
4. Check that ROMs loaded from memory still report the same size, MD5 and cartridge type as unpatched master, since that's the path Kodi uses.
5. Push and let Jenkins build all six platforms.

## The prompt it was made from

This version was a one-shot from the prompt below, run as a single Claude Fable 5.2 agent at max effort. It took about two and a half hours, most of it the agent building and checking its own work on the compilers above, and about 0.6 million output tokens. For a rebase, change the Stella commit and give it these notes.

> Convert latest Stella master to **GNU C++20 (`-std=gnu++20`)**, preserving all existing functionality.
>
> The compatibility baseline is **AppleClang 15.0 / Xcode 15.2 / macOS 11 deployment target**, plus Kodi 22's Android NDK r28c toolchain. Jenkins cannot be modified.
>
> Perform a complete C++23-to-GNU-C++20 compatibility conversion across the entire transitive libretro source set. Account for compiler and standard-library limitations, not just language syntax. Do not rely on newer libc++ facilities unavailable in Xcode 15.2 or runtime functionality unavailable on macOS 11.
>
> Use four parallel agents with separate source ownership. Preserve runtime behavior, especially memory-loaded ROM sizing, hashes, and cartridge detection. Make the smallest necessary changes and validate all supported platforms. Continue resolving build failures before declaring completion.
