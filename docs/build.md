# Building

## Requirements

| Tool | Notes |
| --- | --- |
| CMake | 3.19 or newer |
| C++17 compiler | mingw-w64 (winlibs UCRT), MSVC, gcc or clang |
| Vulkan SDK | Only for the Vulkan backend. Provides `glslc`; the `SPIRV-Headers` CMake package is required too. |
| CUDA toolkit | Only for the CUDA backend. Provides `nvcc`, which needs MSVC rather than mingw as its host compiler on Windows. |
| Ninja | Optional but much faster than the default generator |
| Python 3.9+ | Only for converting weights to GGUF |

The configure step probes which of these is installed, so a plain `cmake -B build`
works either way. The Vulkan backend works on any driver exposing Vulkan 1.2, which
covers NVIDIA, AMD and Intel on both Windows and Linux.

## Clone

The ggml backend is a submodule, so clone recursively:

```
git clone --recursive https://github.com/<owner>/Breeze-TTS-2.cpp
cd Breeze-TTS-2.cpp
```

If you already cloned without `--recursive`:

```
git submodule update --init --recursive
```

## Configure and build

Any GPU, whichever one the machine can actually build for. No backend flag is needed:
Vulkan and CUDA are each turned on when their toolchain is found, and the CPU is always
there. If both are found, both go into `libggml` and ggml picks one at runtime.

```
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
```

CUDA only, even on a machine that also has the Vulkan SDK:

```
cmake -B build-cuda -DCMAKE_BUILD_TYPE=Release -DBREEZE_VULKAN=OFF -DBREEZE_CUDA=ON
cmake --build build-cuda -j
```

Vulkan only:

```
cmake -B build-vulkan -DCMAKE_BUILD_TYPE=Release -DBREEZE_VULKAN=ON -DBREEZE_CUDA=OFF
cmake --build build-vulkan -j
```

CPU only:

```
cmake -B build-cpu -DCMAKE_BUILD_TYPE=Release -DBREEZE_VULKAN=OFF -DBREEZE_CUDA=OFF
cmake --build build-cpu -j
```

[CMakePresets.json](../CMakePresets.json) offers the same three trees as `cpu`,
`vulkan` and `cuda`, which also name the build directory and set `Release`:

```
cmake --preset cuda
cmake --build --preset cuda -j
```

The configure step prints what it settled on, which is worth a look:

```
-- breeze: Vulkan SDK not found (needs glslc + SPIRV-Headers), building without the Vulkan backend
-- breeze: CUDA toolkit 12.1.105 found, enabling the CUDA backend
-- breeze: building CUDA for the local GPU: sm_86
-- breeze: backends = CPU;CUDA
```

## CMake options

| Option | Default | Effect |
| --- | --- | --- |
| `BREEZE_VULKAN` | `auto` | Build ggml with the Vulkan backend. `auto` is probed from the installed toolchain, and is on only when `glslc` and the `SPIRV-Headers` CMake package are both found; passing an explicit value always wins. |
| `BREEZE_CUDA` | `auto` | Build ggml with the CUDA backend. `auto` is probed from the installed toolchain, and is on when the CUDA toolkit is found, but never under mingw, where nvcc cannot use mingw as its host compiler; passing an explicit value always wins, and an explicit `ON` under mingw stops the configure step with that reason instead of failing later in nvcc. |
| `BREEZE_CUDA_NCCL` | `OFF` | Link `ggml-cuda` against NCCL. Only useful for multi GPU, and ggml turns it on by itself when `libnccl` happens to be installed. |
| `BREEZE_BUILD_CLI` | `ON` | Build `breeze-cli`. |
| `BREEZE_BUILD_SERVER` | `ON` | Build `breeze-server`. |
| `BREEZE_BUILD_SHARED` | `ON` | Build the shared C library. |
| `BREEZE_MP3` | `ON` | Let the server encode MP3 for OpenAI style requests, using the `third_party/shine` submodule. Off, or with the submodule missing, those requests get WAV instead. |

### Vulkan or CUDA

Vulkan stays the default whenever its SDK is installed. It is a little ahead at Q4_K,
level at F16, and it needs no toolkit. Measured on an RTX 3060 12 GB with its tensor core
limit unlocked, per frame of audio (80 ms). The Q4_K and Q8_0 rows are averages over 4
seeds:

| Model | `cfg_scale` | Backend | Backbone | Depth | Vocoder | Total per frame | Real time factor |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Q4_K | 1 | Vulkan | 7.12 ms | 23.66 ms | 2.38 ms | 33.16 ms | 2.41x |
| Q4_K | 1 | CUDA | 10.23 ms | 24.20 ms | 3.37 ms | 37.80 ms | 2.12x |
| Q4_K | 3 | Vulkan | 10.91 ms | 32.65 ms | 2.42 ms | 45.98 ms | 1.74x |
| Q4_K | 3 | CUDA | 16.44 ms | 27.33 ms | 3.41 ms | 47.18 ms | 1.70x |
| Q8_0 | 1 | Vulkan | 9.06 ms | 28.83 ms | 2.39 ms | 40.28 ms | 1.99x |
| Q8_0 | 3 | Vulkan | 13.07 ms | 35.10 ms | 2.56 ms | 50.74 ms | 1.58x |
| F16 | 1 | Vulkan | 13.06 ms | 40.95 ms | 2.41 ms | 56.42 ms | 1.42x |
| F16 | 1 | CUDA | 12.86 ms | 42.12 ms | 3.03 ms | 58.01 ms | 1.38x |

CUDA used to be far behind, about 0.6x, for two reasons, and both are fixed:

- **CUDA graphs were never compiled in.** ggml leaves them off unless it is built
  by llama.cpp, so `BREEZE_CUDA` now turns them on. The depth decoder replays one
  fixed graph per frame, which took it from 55 ms to 23 ms. Set
  `GGML_CUDA_DISABLE_GRAPHS=1` to compare.
- **The CUDA `conv_transpose_1d` kernel is a per sample loop.** It cost about 1.6 s
  on a 100 frame decode. The vocoder now upsamples with a matmul and overlap add,
  which took it from 62 ms to 7 ms per frame on CUDA, and from 12 ms to 6 ms on
  Vulkan. Keeping its state between flushes then took those to about 3.5 ms and 2.3 ms.

The backbone is what still separates them, because it builds a new graph every
frame and CUDA cannot replay that. Only the depth decoder gets the full benefit
of graphs today.

Tensor core speed, including an unlocked card, only reaches the large batched
matmuls: backbone prefill and the vocoder. Depth and backbone decode multiply one
or two rows at a time on a path that skips the tensor cores, and they are limited
by reading the weights from memory. That is why a smaller depth type is the
fastest option on either backend: F16 spends about 42 ms per frame in depth, and
F16 with a Q4_K depth decoder spends about 25 ms.

Choosing a model, running several streams and tuning are in
[performance.md](performance.md).

### CUDA specifics

Architectures are probed, not guessed. When `BREEZE_CUDA` is on and
`CMAKE_CUDA_ARCHITECTURES` was not set, the configure step reads
`nvidia-smi --query-gpu=compute_cap` and turns `8.6` into `86`, so a first build compiles
one architecture instead of the eight ggml falls back to. Several cards are read as one
list, `86;89` for example. `nvidia-smi` missing, failing, or printing something else is
not an error: the variable is then left undefined and ggml uses its own list. Cross
compiling skips the probe altogether, because the build machine's cards say nothing about
the target: with `CMAKE_CROSSCOMPILING` set, `nvidia-smi` is not read at all and
`CMAKE_CUDA_ARCHITECTURES` is left for you to pass. With CMake
3.24 or newer and `GGML_NATIVE` on, ggml asks nvcc for `native` instead, which is the
same idea done by the compiler.

Override it when the build machine and the run machine differ, or for a binary you hand
out:

```
cmake -B build -DBREEZE_VULKAN=OFF -DBREEZE_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES="75;86"
```

The quotes matter: a bare `;` ends the command in most shells.

An `XX-virtual` entry such as `90-virtual` compiles to PTX instead of device code and is
JITed on first use, which covers cards newer than the toolkit. Note that setting the
variable at all takes ggml's own default list out of the picture, and that handing it
`native` only works when the build runs CMake 3.24 or newer, which is the other reason
the probe in `CMakeLists.txt` is there.

CUDA graphs are compiled in: `BREEZE_CUDA` turns `GGML_CUDA_GRAPHS` on. The depth decoder
replays one graph per frame, which is worth the 55 ms to 23 ms per frame quoted above.
`GGML_CUDA_DISABLE_GRAPHS=1` in the environment turns them off at runtime for a
comparison.

The binaries need `libcudart` and `libcublas` at runtime. Both ship with the toolkit, so a
machine with a driver but no toolkit will not run a CUDA build, while the Vulkan build
only needs the driver. NCCL is a separate matter: ggml links it in as soon as `libnccl` is
installed, single GPU inference has no use for it, so `BREEZE_CUDA_NCCL` defaults to
`OFF`.

A CUDA build contains no Vulkan code at all, which is easy to confirm:

```
ldd build-cuda/breeze-cli | grep -i vulkan   # no output
```

#### Shared or one file

On Linux the default build puts the ggml backends in shared libraries and `breeze-cli`
finds them through its RUNPATH, so `./build-cuda/breeze-cli` runs straight out of the build
tree. That convenience costs disk space: `libggml-cuda.so` alone is close to 200 MB, and
every binary in the tree links it.

For a single self contained file, the same shape the release archives have, ask for static
ggml backends:

```
cmake -B build-cuda-static -DCMAKE_BUILD_TYPE=Release -DBREEZE_VULKAN=OFF -DBREEZE_CUDA=ON -DBUILD_SHARED_LIBS=OFF
cmake --build build-cuda-static -j
```

`cmake --preset cuda-static` sets that command line up, and `cmake --build --preset
cuda-static` is the build. Measured on this machine: 3.4 s to configure, 228 s to compile on
14 threads, no errors, and a `breeze-cli` of 190 MiB. `ldd` then lists no `libggml*` at all
and no Vulkan either, only the system libraries plus the CUDA runtime (`libcudart`,
`libcublas`, `libcublasLt`, `libcuda`). One thing to know: `BUILD_SHARED_LIBS=OFF` only
reaches ggml's backend libraries. `libbreeze.so` is declared `SHARED` explicitly, so it is
still built.

The architecture is still what the build machine probed, so the result runs on those cards
only. Pass the list yourself when you hand the binary to someone else:

```
cmake --preset cuda-static -DCMAKE_CUDA_ARCHITECTURES="75;86"
```

## Outputs

| Target | Description |
| --- | --- |
| `breeze-cli` | One shot synthesis to WAV |
| `breeze-server` | HTTP server with optional web UI |
| `breeze-quantize` | Converts an F16 GGUF to Q8_0, Q6_K, Q4_K, Q3_K or Q2_K |
| `libbreeze` | Shared C library, see [c-api.md](c-api.md) |
| `libbreeze_core` | Static library if you would rather link directly |

The web UI is compiled into `breeze-server` at build time by
`cmake/embed_webui.cmake`, which turns the files in [webui](../webui) into a
generated header. Edit the sources and rebuild to pick up changes.

## Backend selection at runtime

Both apps ask ggml for any GPU backend it was built with and fall back to CPU if
there is none. Pass `--cpu` to skip the GPU entirely. The startup line names the
backend that was picked:

```
backend: Vulkan0, sample rate: 24000
```

A CUDA build prints `backend: CUDA0` instead, and a build with no GPU backend prints
`backend: CPU`. Nothing else differs, the same binary serves every backend it was
compiled with and still falls back to the CPU when the GPU cannot be used.

## mingw specifics

mingw builds link the compiler runtime statically, so the binaries depend only
on system DLLs plus `vulkan-1.dll`. You can copy them to a machine with no
mingw installed and they will run.

OpenMP is disabled for mingw because there is no static `libgomp`, and linking
it dynamically would leak a `libgomp-1.dll` dependency into the shared library.
The ggml CPU backend falls back to its own thread pool, so it still runs
multithreaded.

## Troubleshooting

**`third_party/ggml is missing`** means the submodule was not initialised. Run
`git submodule update --init --recursive`.

**`third_party/shine is missing so mp3 requests will get wav`** is the same thing
for the MP3 encoder, which was added after ggml. A clone from before then needs
`git submodule update --init --recursive` once to pick it up.

**Vulkan shader generation crashes with `0xC0000139`** on Windows when another
program has put an incompatible `libstdc++-6.dll` earlier on `PATH`. Tesseract
OCR is a common culprit. Put your mingw `bin` directory first:

```powershell
$env:PATH = "C:\path\to\mingw64\bin;" + $env:PATH
```

**No Vulkan device found** usually means an outdated GPU driver, or running
inside a container or remote session without GPU passthrough. It is not fatal;
the model loads on CPU instead, just slowly.

**`BREEZE_VULKAN=ON needs the Vulkan SDK (glslc) and the SPIRV-Headers CMake package`**
means you asked for the Vulkan backend on a machine that cannot build it. Leaving the
flag out probes for the SDK and builds without Vulkan when it is missing, which is what
most CUDA machines want; an older version defaulted the option to `ON`, so a bare
`cmake -B build` that used to fail here now just works. Install the SDK (the LunarG
installer on Windows, `vulkan-sdk` or the LunarG tarball on Linux) if you do want Vulkan,
or pass `-DBREEZE_VULKAN=OFF`.

**`BREEZE_CUDA=ON needs the CUDA toolkit`** is the same thing for `nvcc`. Set
`CUDAToolkit_ROOT` when the toolkit is installed somewhere the find module does not look.

**`nvcc fatal: Unsupported gpu architecture 'compute_XXX'`** comes from
`CMAKE_CUDA_ARCHITECTURES` naming a card newer than the toolkit, typically in a build tree
reused from another machine. Pass architectures the toolkit knows, for instance
`-DCMAKE_CUDA_ARCHITECTURES=86`, or delete the build directory and let it be probed
again. If instead the program starts and reports **`no kernel image is available for
execution on the device`**, the binary was built for a different card than it is running
on: rebuild for the compute capability of that card
(`nvidia-smi --query-gpu=compute_cap --format=csv,noheader`) or add an `XX-virtual` entry
so PTX JIT can fill the gap.

**Out of video memory** during generation: use a smaller quantization. Q4_K
needs roughly 3 GB of VRAM against about 7 GB for F16. See
[models.md](models.md).

If that is still too much, for instance next to a local LLM on a 6 GB card,
`--te-cpu` (or `BREEZE_TE_CPU=1` for the C API and the Python bindings) keeps
the T5Gemma2 text encoder in system memory and runs it on the CPU. It runs
once per piece of text, so the cost is a delay before each piece rather than a
slower stream. On an RTX 2060 with Q4_K and a Q2_K depth decoder, peak video
memory went from 2725 MB to 1811 MB and the first audio from about 0.2 s to
0.75 s (the encoder took about 0.5 s on 6 threads). Do not combine it with
`--parallel`, every encode pauses the other streams. The numbers on a 3060 and
the details are in [performance.md](performance.md#short-on-video-memory).
