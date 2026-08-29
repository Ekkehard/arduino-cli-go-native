# arduino-cli-go-native

Install `arduino-cli` on Apple Silicon Macs **without any dependency on
Rosetta 2** — no Arduino IDE required, just the CLI native.

## The problem this solves

`arduino-cli` itself ships native arm64 builds, and Homebrew's `avr-gcc`,
`avrdude`, and `arm-none-eabi-gcc` are all natively compiled for Apple
Silicon too. So it would be reasonable to assume a `brew install arduino-cli`
plus `arduino-cli core install arduino:avr` gets you a fully native toolchain.
It doesn't.

Arduino's own board-manager packages — the `avr-gcc`, `avrdude`, and
`arm-none-eabi-gcc` builds that `arduino-cli core install` downloads into
`~/Library/Arduino15/packages/` — are still published for macOS as
**x86_64-only**. The official package index even says so explicitly: Apple
Silicon Macs are simply pointed at the Intel build, to be run under Rosetta.
This is a known, long-open issue
([arduino/toolchain-avr#91](https://github.com/arduino/toolchain-avr/issues/91))
with no sign of a native build arriving any time soon.

Since Apple has announced Rosetta 2 is headed for deprecation, that's not a
future reasonable people may want to depend on.

## What this script does

1. Installs native arm64 builds of `avr-gcc` (via the `osx-cross/avr` tap),
   `avrdude`, `arm-none-eabi-gcc`, `bossa`, and `arduino-cli` through
   Homebrew, and verifies each one is actually arm64 (not just installed).
2. Wipes any existing `~/Library/Arduino15` state and does a clean
   `arduino-cli core install` of the platforms avr, sam, and samd. This step 
   still downloads Arduino's x86_64 toolchain packages — that's unavoidable, 
   since `arduino-cli` always fetches a platform's declared tool dependencies.
3. Walks every binary Arduino actually downloaded and symlinks it to the
   matching native Homebrew binary in place, so the *same* file paths that
   `platform.txt` expects now resolve to native arm64 code.

The result: `arduino-cli compile` / `upload` never touches an x86_64 binary,
and never needs Rosetta, for the boards below.  Others should be easy to add.

## Boards covered

| Board(s) | Arduino core | Toolchain |
|---|---|---|
| Uno, Leonardo, Pro/Pro Mini, Mini, Mega, Nano | `arduino:avr` | avr-gcc, avrdude |
| Due | `arduino:sam` | arm-none-eabi-gcc, bossac |
| MKR Zero | `arduino:samd` | arm-none-eabi-gcc, bossac |

## Side note

Each downloaded package does still carry some genuinely-unused x86_64
content afterward: GCC's internal single-purpose binaries (`cc1`,
`cc1plus`, `collect2` and similar) live in a nested directory outside
`bin/`, so the by-name swap never touches them. That's harmless — once the
top-level tool is the native Homebrew one, it calls its own internal
helpers from inside its own Homebrew keg, never the Arduino package's.

## Requirements

- Apple Silicon Mac
- Xcode Command Line Tools
- *native** Homebrew install at `/opt/homebrew`
  (not `/usr/local`, and not a Terminal session running under Rosetta —
  check with `arch`, should print `arm64`)

## Usage

```bash
./arduino-cli-go-native
```

Flags:

| Flag | Effect |
|---|---|
| `-h`, `--help` | Show usage info |
| `-V`, `--Version` | Show script version |
| `--coreUpgrade` | Skip the brew installs and `core install` steps; only re-run  the symlink swap. Use this after you've manually run `arduino-cli core  upgrade`, which can pull in a new toolchain version (and therefore a fresh set  of x86_64 binaries) under a new version-numbered folder. **This has not been tested yet**|

Verify it worked:

```bash
arduino-cli compile --fqbn arduino:avr:uno --verbose /path/to/sketch
```

The invoked compiler path in the verbose output should resolve into your
Homebrew prefix, not into an `Arduino15` tools path.

## A gotcha worth knowing about

If you hit compiler errors that look like macOS SDK header conflicts (e.g.
`error: architecture not supported` coming from deep inside
`/Library/Developer/CommandLineTools/SDKs/...`) even after this script
reports everything as arm64 — that's very likely a **different** problem:
a `CPATH` or `CPLUS_INCLUDE_PATH` environment variable set globally in your
shell profile (often left over from getting some other GNU/Homebrew package
building on macOS) leaking the host macOS SDK into the cross-compiler's
header search path. Check with:

```bash
env | grep -i -E "cpath|include"
```

and scope any hits to the specific build that needs them rather than your
whole shell profile.  Cross compilers need clean include path environment
variables.

## License

MIT — use, modify, and redistribute freely.
