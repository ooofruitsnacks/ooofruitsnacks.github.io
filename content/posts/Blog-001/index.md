+++
date = '2026-09-10T19:07:41-04:00'
draft = true
title = 'Blog 001'

How I Accidentally Created Odin's First SDK While Making a Videogame

An "under the hood" deep-dive into my video game written in Odin that ships eight binaries across four operating systems, two CPU architectures, and builds it's own compiler from source and bootstraps that compiler to build the game. It's truly madness! Odin is young. There's no __Cargo__ or __go__ build cross-compilation ecosystem of prebuilt runners, no __crates.io__ of packaging helpers, and no __Flathub__ SDK extension. Most Odin projects handle this by not handling it at all, no offense to those other projects.

My repo instead:

-Builds the Odin compiler itself from source inside a network-isolated Flatpak sandbox on two architectures

-Detects and works around architecture dependent LLVM target availability with a generated llvm-config interceptor and force included stub macros

-Rebuilds a vendored static library to backport across a glibc symbol-versioning change

-Rebuilds raylib from source to replace a prebuilt binary blob with a sandbox native one

-Asserts its own ABI floor in CI so a compatibility regression can't reach users

-Walks a dependency graph at build time to assemble a self contained .app

-Ships eight verified artifacts with checksums from a single git tag

## Repository Structure

```
FuzzyBuddyFarms/
│
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.md
│   │   └── feature_request.md
│   └── workflows/
│       └── build.yml
│
├── assets/
│   └── bill_painting.png
│
│
├── packaging/
│   │
│   ├── flatpak/
│   │   ├── <app-id>.yml
│   │   ├── launcher.sh
│   │   ├── <app-id>.desktop
│   │   ├── <app-id>.metainfo.xml
│   │   ├── icon.png
│   │   └── odin-llvm-target-guards.sh
│   │
│   ├── macos/
│   │   ├── Info.plist
│   │   └── icon.icns
│   │
│   └── raspberrypi/
│       ├── RUN_ME.sh
│       ├── <app>.desktop
│       └── isoc23_compat.c
│
│
├── fuzzybuddyfarms.odin
│
├── net.odin
│
├── README.md
├── SECURITY.md
├── LICENSE
└── .gitignore
```

| Path | Role | Consumed by |
|:--|:--|:--|
| `.github/workflows/` | The entire build pipeline — typecheck gate with a 5-way platform matrix, Docker container Pi build, aarch Flatpak, tagged release | GitHub Actions |
| `.github/ISSUE_TEMPLATE/` | Structured issue intake | GitHub UI |
| `assets/` | Runtime image assets. Deliberately near empty because the game renders itself from raylib | Game at runtime |
| `packaging/flatpak/` | Flathub manifest with commit pinned Odin + raylib and the LLVM target guard script that lets the compiler build inside a sandbox with a partial LLVM | `flatpak-builder` |
| `packaging/macos/` | `.app` bundle metadata and icon | CI bundle assembly |
| `packaging/raspberrypi/` | GL version override wrapper and the `__isoc23_*` compatibility shim that backports raylib across glibc 2.36 | CI Pi job |
| `fuzzybuddyfarms.odin` | The whole game, one Odin package lol. | `odin build` |
| `net.odin` | Networking module/package | `odin build` |

Actions Workflow

```
         ┌─────────┐
         │  check  │
         └────┬────┘
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
      ┌──────────────┐  ┌─────────┐  ┌────────────┐
      │    build     │  │   Pi    │  │  flatpak   │
      │  (5 way      │  │ (Debian │  │  (aarch    │
      │   matrix)    │  │  12 in  │  │  container │
      │              │  │  Docker)│  │   matrix)  │
      └──────┬───────┘  └────┬────┘  └─────┬──────┘
      └───────────────┼─────────────┘
        ▼
       ┌──────────┐
       │ release  │
       └──────────┘
```

---

__Other Cool Highlights:__

The game ships and runs on __Raspberry Pi OS (Bookworm)__. This shouldn't be possible and really there was no need to support __bookworm__. However I am stupid. __Bookworm__ was shipped with **glibc 2.36** but **Odin's** prebuilt toolchain and vendored **libraylib.a** are both compiled on __Ubuntu 24.04__ which ships with **glibc 2.39.** This can't be cross compiled to run on __Bookworm__ but between 2.36 and 2.39 **glibc** introduced symbol versioning aliases **__isoc23_strtol** and **__isoc23_sscanf**. When you compile against a modern **glibc** header with a C23-ish standard mode, any calls to **strtol** get redirected at compile time to **__isoc23_strtol**. So **libraylib.a** built on __Ubuntu 24.04__ contains undefined symbols that do not exist in **glibc 2.36**. If you were to link it on __Bookworm__ it would return a wall of undefined symbol errors from inside a static library you didn't compile and can't easily rebuild. 

__So what the hell do we do!?!__

 Patch the archive in place! The actions workflow creates a Debian 12 container matching the Pi's actual **libc** exactly, rather than hoping a newer distro's output is compatible: 

```docker run --rm --platform linux/arm64 -v "$PWD:/src" -w /src debian:12 bash -c '...'```

Inside this container it compiles a shim that provides the missing symbols to **glibc** by forwarding to the classic ones:

```
int __isoc23_sscanf(const char *s, const char *fmt, ...) {
    va_list ap; va_start(ap, fmt);
    int r = vsscanf(s, fmt, ap);
    va_end(ap); return r;
}
long __isoc23_strtol(const char *p, char **e, int b) { return strtol(p, e, b); }
/* ...and strtoul, strtoll, strtoull, strtoimax, strtoumax, fscanf, scanf, vsscanf */
```

Then it injects that shim object directly into the raylib static archive:

```
RAYLIB_A="$ODIN_ROOT/vendor/raylib/linux-arm64/libraylib.a"
ar r "$RAYLIB_A" /tmp/isoc23_compat.o
ranlib "$RAYLIB_A"
```

This method adds a member to the symbol index. The index is rebuilt so the linker can find the new definitions. The shim is now inside raylib as far as the linker is concerned and it resolves its own dangling references. No patching Odin's source code, no forking raylib, no rebuilding the toolchain. It handles itself. You're welcome. 

__You can verify each step yourself as well. This is always running so if a workflow ever fails you'll know why.__

```
test -f packaging/raspberrypi/isoc23_compat.c \
  || { echo "FATAL: isoc23_compat.c is not committed"; exit 1; }

nm -g --defined-only /tmp/isoc23_compat.o | grep isoc23 \
  || { echo "FATAL: shim compiled but defines no __isoc23_* symbols"; exit 1; }

ar t "$RAYLIB_A" | grep isoc23_compat.o \
  || { echo "FATAL: ar did not add the shim member"; exit 1; }
```

I also implemented a sandbox native swap of raylib. I did this because **Odin** ships raylib as a prebuilt binary in vendor/raylib/. Inside a Flatpak sandbox a blob compiled against some other distribution's libraries is a liability so the manifest rebuilds raylib from source and installs it over the vendored copy:

```
- name: raylib
  build-commands:
    - |
      cmake -S . -B build -GNinja \
        -DCMAKE_BUILD_TYPE=Release \
        -DBUILD_SHARED_LIBS=OFF \
        -DBUILD_EXAMPLES=OFF \
        -DPLATFORM=Desktop \
        -DCMAKE_POSITION_INDEPENDENT_CODE=ON
      ninja -C build
      install -Dm644 "$BUILT" "/app/odin/vendor/raylib/$REL"
```

```BUILD_SHARED_LIBS=OFF``` means static linking. The game carries its raylib rather than depending on a runtime library resolution inside the sandbox. 

```POSITION_INDEPENDENT_CODE=ON``` keeps it compatible which the freedesktop SDK's hardening flags require.

Then the game module runs ```odin build .``` with ```ODIN_ROOT=/app/odin``, and Odin's ```vendor:raylib``` bindings link against the raylib that was compiled earlier in the sandbox against this exact runtime. If you thought the __Raspberry Pi OS Bookworm__ and the sandbox native swap of raylib method was clever, check my next party trick out.

I wanted to get my game published to **Flathub** as well, this will help massively with getting support for those odd ball __Linux__ distros. Instead of worrying about getting a CICD pipeline/actions workflow I can simply direct __Linux__ users to **Flathub** to download. The only issue is that **Flathub** doesn't allow for prebuilt releases, **Odin** doesn't have an SDK, and **Odin** doesn't support **Flathub**. These are some massive hurdles to get over. **Flathub** has SDK support for **Rust**, **Go**, **Node**, **.NET**, **LLVM** and **OpenJDK**, so we unintentionally created our own SDK for **Odin**. Following a similar process to the __Raspberry Pi OS Bookworm__ trick, we build the **Odin** compiler from source within the **Flathub** sandbox environment and then use that compiler to build the game. 

```
- name: odin
  buildsystem: simple
  build-options:
    append-path: /usr/lib/sdk/llvm20/bin
    prepend-ld-library-path: /usr/lib/sdk/llvm20/lib
    env:
      REAL_LLVM_CONFIG: /usr/lib/sdk/llvm20/bin/llvm-config
  build-commands:
    - bash ./odin-llvm-target-guards.sh
    - install -Dm755 odin /app/odin/odin
    - cp -r base core vendor shared /app/odin/
    - /app/odin/odin version
  sources:
    - type: git
      url: https://github.com/odin-lang/Odin.git
      tag: dev-2026-08
      commit: 8412dc37aa91def0c2fa90f89eade29056b4e608
```

Everything is pinned to a tag and commit SHA256, so the build is completely byte-reproducible even if the tag is moved upstream. This is something I'm very proud of, building the compiler itself as a workflow dependency and targeting cross-architecture, **twice**, for every release. **Odin's** compiler is written in **C++** that links **libLLVM**. **Flathub** has __org.freedesktop.Sdk.Extension.llvm20__. The main problem is that extension isn't built with every **LLVM** target enabled and the targets it does have depends on the host architecture. It doesn't target x86 and RISCV which I want to include support for. Odin's **build_odin.sh** assumes a full **LLVM** in two independent ways.

___


__Issue 1:__ It runs llvm-config --libs core native passes arm aarch64 x86 webassembly riscv. 

llvm-config errors out on an unknown component name.

__Issue 2:__ The compiler source calls LLVMInitializeX86TargetInfo(), LLVMInitializeRISCVTarget(), etc. unconditionally. 

Even if you fix the first problem you would still get undefined symbol errors at link time. 

___

__Fix 1:__
The script writes a wrapper shell script, puts that script first on **PATH**, and points **LLVM_CONFIG** at it. The wrapper asks the real **llvm-config** **--targets-built** then filters unavailable component names out of the argument list before forwarding:

```
BUILT=" $("$REAL" --targets-built | tr '[:upper:]' '[:lower:]') "
args=()
for a in "$@"; do
    case "$a" in
        x86|aarch64|arm|webassembly|riscv|nvptx|amdgpu|mips|powerpc|...)
            if echo "$BUILT" | grep -qw "$a"; then
                args+=("$a")
            else
                echo "llvm-config shim: dropping unavailable component '$a'" >&2
            fi
            ;;
        *) args+=("$a") ;;
    esac
done
exec "$REAL" ${args[@]+"${args[@]}"}
```

The script then self tests the shim with the exact invocation **build_odin.sh** will use so if the shim is broken you'll know right away instead of 30 minutes into a compiler build.

```
"${SHIM_DIR}/llvm-config" --libs core native passes arm aarch64 x86 webassembly riscv \
  || { echo "FATAL: shim failed --libs"; exit 1; }
```

__Fix 2:__

For the undefined symbols the script generates a header at build time and injects it into every translation unit via **-include**:

```
export CXXFLAGS="${CXXFLAGS:-} -include ${HEADER}"
export CPPFLAGS="${CPPFLAGS:-} -include ${HEADER}"
```

The script also refuses to proceed if the host target is missing since that would produce a compiler incapable of compiling anything:

```
echo "$TARGETS_LOWER" | grep -qw "$HOST_TARGET" \
  || { echo "FATAL: LLVM lacks the host target ($HOST_TARGET) — unusable"; exit 1; }
```

The header is generated by a shell heredoc. In an early version I created, it used an unquoted heredoc delimiter. After many hours of debugging I realized the issue was that shell quoting bug. It was three layers deep surfacing as a **C++** semantic error about an undeclared variable named **'s'**. The fix was pretty simple thankfully. Two characters added to quote the delimiter which disables all shell processing. I also added a  gate immediately after:

```
echo "=== Checking generated LLVM target header ==="
clang++ -x c++ -std=c++17 -fsyntax-only "$HEADER"
```

So if you learn anything from this blog post, make sure to remember to syntax-check the output before you feed it to a build that takes an hour and a half...lol.

+++
