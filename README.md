# Kangaroo — Pollard's Kangaroo ECDLP Solver

**Version 2.2** — by Jean Luc PONS  
Source: <https://github.com/JeanLucPons/Kangaroo>

Kangaroo is a high-performance C++ implementation of **Pollard's Kangaroo algorithm** for solving the **Elliptic Curve Discrete Logarithm Problem (ECDLP)** on the **secp256k1** curve (the curve used by Bitcoin). Given a public key `P` and a known range `[rangeStart, rangeEnd]` for the private key `k`, it finds `k` such that:

```
k × G = P
```

where `G` is the secp256k1 generator point. This is the mathematical foundation of recovering a Bitcoin private key when the key is known to lie within a constrained range.

Kangaroo supports CPU multi-threading, NVIDIA GPU (CUDA) acceleration, and distributed computation across multiple machines via a server/client network mode.

---

## Table of Contents

1. [How the Algorithm Works](#how-the-algorithm-works)
2. [Requirements](#requirements)
3. [Building](#building)
4. [Input File Format](#input-file-format)
5. [Command-Line Options](#command-line-options)
6. [Usage Examples](#usage-examples)
7. [Work Files: Saving, Resuming, and Merging](#work-files-saving-resuming-and-merging)
8. [Distributed Mode](#distributed-mode)
9. [The Bitcoin Puzzle Context](#the-bitcoin-puzzle-context)
10. [Performance Notes](#performance-notes)
11. [License](#license)

---

## How the Algorithm Works

Pollard's Kangaroo (also called the "lambda algorithm") is a time–memory trade-off method for solving discrete logarithm problems in a known interval. It is far more efficient than brute force when the search range is constrained.

### Tame and Wild Kangaroos

The algorithm uses two groups of pseudo-random walkers in the key space:

- **Tame kangaroos** start from known positions derived from the range bounds. Their paths are fully traceable back to a known private key offset.
- **Wild kangaroos** start from the target public key `P`. Their paths are not known ahead of time.

Both groups take pseudo-random jumps of varying sizes, defined by a set of **NB\_JUMP = 32** pre-computed jump vectors. Each jump moves the kangaroo forward by a step whose size is determined by the current point's coordinates.

![Kangaroo paths illustration](DOC/paths.jpg)

### Distinguished Points (DP)

To detect a collision between a tame and a wild kangaroo without comparing every step, the algorithm uses **distinguished points**: positions whose coordinate satisfies a leading-zeros criterion (controlled by the `-d dpBit` parameter). When a kangaroo lands on a distinguished point, it stores its current location and the accumulated distance traveled in a shared hash table.

When a tame kangaroo and a wild kangaroo both visit the same distinguished point, their paths have collided. Using the known distances, the private key can be immediately recovered:

```
k = tame_distance - wild_distance  (mod curve order)
```

### Architecture Overview

![Architecture overview](DOC/architecture.jpg)

CPU threads and GPU kernels each run a batch of kangaroos in parallel. As distinguished points are found, they are inserted into a central hash table. The collision check happens at insert time.

### Success Probability

The expected number of operations to find a collision scales as approximately **2 × √(rangeWidth)**, making it practical for ranges up to ~160 bits with sufficient hardware. The success probability curve as a function of operations performed:

![Success probability](DOC/successprob.jpg)

### Example: 110-bit Key Search

The `DOC/keys110.jpg` image illustrates a real run solving 110-bit puzzle keys, showing throughput and convergence over time:

![110-bit key example run](DOC/keys110.jpg)

---

## Requirements

### All Builds
| Requirement | Details |
|---|---|
| **OS** | Linux 64-bit or Windows 64-bit |
| **Compiler** | g++ with SSSE3 support (`-mssse3`), GCC 4.8+ recommended |
| **Architecture** | x86-64 (64-bit binary, `-m64`) |
| **POSIX threads** | `libpthread` (standard on Linux) |

### GPU Build (Optional)
| Requirement | Details |
|---|---|
| **CUDA Toolkit** | 8.0 or later (path configurable in `Makefile`) |
| **NVCC** | Bundled with CUDA Toolkit |
| **g++-4.8** | Used by NVCC as the host compiler (`CXXCUDA` in `Makefile`) |
| **NVIDIA GPU** | Any CUDA-capable GPU; compute capability ≥ 3.0 recommended |

> **Windows users:** Visual Studio project files for CUDA 8, 10, and 10.2 are provided in `VC_CUDA8/`, `VC_CUDA10/`, and `VC_CUDA102/` respectively.

---

## Building

### CPU-Only Build

```bash
make
```

This produces the `kangaroo` binary in the current directory.

### GPU-Enabled Build

Provide the `gpu=1` flag and the compute capability (`ccap`) of your GPU:

```bash
# For a GPU with compute capability 7.5 (e.g., RTX 2080)
make gpu=1 ccap=75

# For a GPU with compute capability 8.6 (e.g., RTX 3080)
make gpu=1 ccap=86

# For a GPU with compute capability 6.1 (e.g., GTX 1080)
make gpu=1 ccap=61
```

> **Finding your compute capability:** Run `./kangaroo -l` after a GPU build (see [`-l` option](#command-line-options)), or consult the [NVIDIA GPU Compute Capability table](https://developer.nvidia.com/cuda-gpus).

### Adjusting the CUDA Path

If your CUDA installation is not at `/usr/local/cuda-8.0`, edit the `Makefile`:

```makefile
CUDA = /usr/local/cuda-11.0   # change to your CUDA path
CXXCUDA = /usr/bin/g++-7      # change to a g++ version compatible with your NVCC
```

### Clean Build

```bash
make clean
make        # or: make gpu=1 ccap=<ccap>
```

---

## Input File Format

The input configuration file has a simple line-based text format:

```
<rangeStart>
<rangeEnd>
<publicKey1>
[<publicKey2>]
[...]
```

| Line | Description |
|---|---|
| Line 1 | `rangeStart` — hexadecimal lower bound (inclusive) of the private key search range |
| Line 2 | `rangeEnd` — hexadecimal upper bound (inclusive) of the private key search range |
| Line 3+ | One secp256k1 public key per line, in compressed (33-byte) or uncompressed (65-byte) hex format |

Multiple public keys can be listed; Kangaroo will attempt to solve all of them within the same range.

### Example: `in.txt` (56-bit range, single key)

```
0
FFFFFFFFFFFFFF
02E9F43F810784FF1E91D8BC7C4FF06BFEE935DA71D7350734C3472FE305FEF82A
```

- **Range:** `0x0` to `0xFFFFFFFFFFFFFF` — a 56-bit search space (~7.2 × 10¹⁶ possible keys)
- **Public key:** compressed, 33 bytes (starts with `02` or `03`)

### Example: 64-bit range, single key

```
5B3F38AF935A3640D158E871CE6E9666DB862636383386EE0000000000000000
5B3F38AF935A3640D158E871CE6E9666DB862636383386EEFFFFFFFFFFFFFFFF
03BB113592002132E6EF387C3AEBC04667670D4CD40B2103C7D0EE4969E9FF56E4
```

- **Range:** the last 64 bits vary; the upper 192 bits are fixed — effectively a 64-bit search

---

## Command-Line Options

```
./kangaroo [OPTIONS] inFile
```

| Option | Argument | Default | Description |
|---|---|---|---|
| `-v` | — | — | Print version and exit |
| `-t` | `nbThread` | # of CPU cores | Number of CPU threads to use |
| `-d` | `dpBit` | auto | Number of leading zero bits required for a distinguished point |
| `-gpu` | — | disabled | Enable GPU acceleration |
| `-gpuId` | `id1[,id2,...]` | `0` | Comma-separated list of GPU device IDs to use |
| `-g` | `g1x,g1y[,g2x,g2y,...]` | auto | GPU kernel grid size per device (`x` blocks, `y` threads) |
| `-l` | — | — | List all CUDA-capable devices and exit |
| `-check` | — | — | Verify GPU kernel output against CPU reference and exit |
| `-o` | `fileName` | stdout | Write the found private key to `fileName` |
| `-m` | `maxStep` | unlimited | Give up after `maxStep × expected_ops` operations |
| `-w` | `workfile` | — | Periodically save search progress to `workfile` |
| `-i` | `workfile` | — | Resume search from a previously saved `workfile` |
| `-wi` | `workInterval` | `60` | Interval in **seconds** between automatic work-file saves |
| `-ws` | — | off | Also save kangaroo states (positions) in the work file |
| `-wss` | — | off | Save kangaroo states to the server (client mode) |
| `-wsplit` | — | — | Split the server's work file and reset its hash table |
| `-wm` | `file1 file2 dest` | — | Merge two work files into `dest` |
| `-wmdir` | `dir dest` | — | Merge all work files in directory `dir` into `dest` |
| `-wt` | `timeout` | `3000` | Timeout in **milliseconds** for work-file save operations |
| `-winfo` | `file` | — | Print metadata/info about a work file and exit |
| `-wpartcreate` | `name` | — | Create an empty partitioned work file (directory `name`) |
| `-wcheck` | `workfile` | — | Verify integrity of a work file and exit |
| `-s` | — | — | Start in **server** mode |
| `-c` | `server_ip` | — | Start in **client** mode, connecting to `server_ip` |
| `-sp` | `port` | `17403` | TCP port for server/client communication |
| `-nt` | `timeout` | `3000` | Network timeout in **milliseconds** |
| `inFile` | (positional) | required* | Input configuration file (`*` not required in client mode) |

> **DP auto-tuning:** When `-d` is omitted, Kangaroo automatically selects the distinguished-point bit size based on the range width and the number of available threads/GPUs to balance table size against collision probability.

---

## Usage Examples

### 1. Minimal CPU Run

Solve the key described in `in.txt` using all available CPU cores:

```bash
./kangaroo in.txt
```

### 2. Multi-Threaded CPU Run

Explicitly set the number of CPU threads (e.g., 8 threads):

```bash
./kangaroo -t 8 in.txt
```

Reduce CPU usage on a shared machine (e.g., 2 threads):

```bash
./kangaroo -t 2 in.txt
```

### 3. GPU-Accelerated Run

Enable the GPU (device 0) alongside CPU threads:

```bash
./kangaroo -gpu in.txt
```

Use a specific GPU by its device ID (e.g., GPU 1):

```bash
./kangaroo -gpu -gpuId 1 in.txt
```

Use multiple GPUs (devices 0 and 1) simultaneously:

```bash
./kangaroo -gpu -gpuId 0,1 in.txt
```

Manually specify the kernel grid size for GPU 0 (e.g., 64 blocks × 512 threads):

```bash
./kangaroo -gpu -gpuId 0 -g 64,512 in.txt
```

### 4. Saving and Resuming a Search

Start a long search, automatically saving progress to `work.dat` every 300 seconds (5 minutes), also preserving kangaroo states:

```bash
./kangaroo -w work.dat -wi 300 -ws in.txt
```

If the search is interrupted (Ctrl+C, power loss, etc.), resume from the saved file:

```bash
./kangaroo -i work.dat in.txt
```

Resume and keep saving (so it can be interrupted again):

```bash
./kangaroo -i work.dat -w work.dat -wi 300 -ws in.txt
```

### 5. Merging Work Files

If two independent searches have been run on the same key (e.g., on two separate machines), merge their work files to combine the discovered distinguished points and potentially trigger a collision:

```bash
./kangaroo -wm work_machine1.dat work_machine2.dat merged.dat
```

Merge an entire directory of work files produced by many clients:

```bash
./kangaroo -wmdir /path/to/workfiles/ merged.dat
```

Inspect a work file's metadata before merging:

```bash
./kangaroo -winfo work_machine1.dat
```

Check a work file for corruption:

```bash
./kangaroo -wcheck work_machine1.dat
```

### 6. Distributed Server + Client Run

**On the server machine** (runs the hash table and detects collisions):

```bash
./kangaroo -s in.txt
```

Use a custom port (default is 17403):

```bash
./kangaroo -s -sp 9000 in.txt
```

**On each client machine** (connects to the server and runs the walk):

```bash
./kangaroo -c 192.168.1.100
```

With a custom port:

```bash
./kangaroo -c 192.168.1.100 -sp 9000
```

Clients with GPU acceleration:

```bash
./kangaroo -gpu -c 192.168.1.100
```

Clients that also send their kangaroo states to the server for persistence:

```bash
./kangaroo -gpu -c 192.168.1.100 -wss
```

### 7. Saving Results to a File

Write the discovered private key (in hex) to `result.txt` instead of only printing to stdout:

```bash
./kangaroo -o result.txt in.txt
```

### 8. Listing GPUs and Verifying the Build

List all detected CUDA-capable GPU devices:

```bash
./kangaroo -l
```

Run a correctness check — compare GPU kernel output against the CPU reference implementation:

```bash
./kangaroo -check in.txt
```

---

## Work Files: Saving, Resuming, and Merging

Work files allow a search to be paused and resumed, or to be distributed across many machines and then combined.

### Work File Types

| Header Tag | Contents | Created by |
|---|---|---|
| `HEADW` | Full work file (distinguished points + metadata) | `-w` flag |
| `HEADK` | Kangaroo-only (kangaroo positions + distances) | `-ws` flag |
| `HEADKS` | Compressed kangaroo-only | `-wss` (server) |

### Typical Workflow

```
# Machine A — start search, save every 5 minutes
./kangaroo -w work_A.dat -wi 300 -ws in.txt

# Machine B — independently start the same search
./kangaroo -w work_B.dat -wi 300 -ws in.txt

# Later: merge the two files (can be done even while machines are running)
./kangaroo -wm work_A.dat work_B.dat combined.dat

# Load combined file and continue
./kangaroo -i combined.dat -w combined.dat -wi 300 -ws in.txt
```

### Partitioned Work Files

For very large searches, partitioned work files split the distinguished-point table into 256 (`MERGE_PART`) buckets stored as separate files in a directory. Create an empty partitioned work file with:

```bash
./kangaroo -wpartcreate my_partitioned_work/
```

---

## Distributed Mode

The server/client model allows many machines (or cloud instances) to cooperate on a single key search:

```
┌─────────────────────────────────┐
│            SERVER               │
│  ./kangaroo -s -sp 17403 in.txt │
│  - Holds the hash table         │
│  - Detects collisions           │
│  - Announces result             │
└──────────┬──────────────────────┘
           │  TCP port 17403
    ┌──────┴───────┐
    │              │
┌───▼───┐      ┌───▼───┐
│Client1│      │Client2│  ...
│-gpu   │      │-t 8   │
└───────┘      └───────┘
```

- Clients receive the configuration (range + public key) from the server automatically — no need to pass `inFile` on the client.
- Distinguished points found by clients are sent to the server every `SEND_PERIOD = 2` seconds.
- Idle clients are disconnected after `CLIENT_TIMEOUT = 3600` seconds.
- The server port defaults to **17403** and is configurable with `-sp`.
- Network timeout defaults to **3000 ms** and is configurable with `-nt`.

---

## The Bitcoin Puzzle Context

The file `puzzle32.txt` contains configuration entries for **Bitcoin Puzzle Transaction** keys #105 through #160. This is a well-known community challenge created in 2015 where the creator sent Bitcoin to addresses whose private keys are integers with a known bit-length. The higher the puzzle number, the larger the bit range.

Each entry in `puzzle32.txt` follows this structure:

```
# <puzzle_number> ( <bitcoin_address> ) [Solved date: Priv=<hex>]
<rangeStart_hex>
<rangeEnd_hex>
<compressed_public_key_hex>
```

**Examples from `puzzle32.txt`:**

| Puzzle | Bits | Status | Address |
|---|---|---|---|
| #105 | 105 | ✅ Solved (2019-09-23) | `1CMjscKB3QW7SDyQ4c3C3DEUHiHRhiZVib` |
| #110 | 110 | ✅ Solved (2020-05-30) | `12JzYkkN76xkwvcPT6AWKZtGX6w2LAgsJg` |
| #115 | 115 | ✅ Solved (2020-06-16) | `1NLbHuJebVwUZ1XqDjsAyfTRUPwDQbemfv` |
| #120 | 120 | ⏳ Open | `17s2b9ksz5y7abUm92cHwG8jEPCzK3dLnT` |
| #125 | 125 | ⏳ Open | `1PXAyUB8ZoH3WD8n5zoAthYjN15yN5CVq5` |
| #130 | 130 | ⏳ Open | `1Fo65aKq8s8iquMt6weF1rku1moWVEd5Ua` |
| #135–#160 | 135–160 | ⏳ Open | — |

To search for puzzle #120 using the GPU:

```bash
# Extract puzzle #120 entry from puzzle32.txt into a standalone file
# (lines 13–15 of puzzle32.txt, adjusting for comment lines)
./kangaroo -gpu puzzle32.txt
```

> **Note:** Puzzle32.txt contains multiple key entries. Kangaroo will process all keys in the file sequentially. For a single puzzle, create a dedicated input file with only that puzzle's three lines.

> **Ethical use:** These puzzles represent Bitcoin held in wallets specifically for the purpose of being solved. Applying this tool to arbitrary Bitcoin addresses without the owner's consent is illegal and unethical.

---

## Performance Notes

### Theoretical Complexity

The expected number of elliptic-curve point operations to find the private key is approximately:

```
Operations ≈ 2 × √(rangeWidth)
```

| Range Width | Bit Size | Expected Operations |
|---|---|---|
| 2⁵⁶ | 56-bit | ~1.7 × 10⁸ (~170 million) |
| 2⁶⁴ | 64-bit | ~4.3 × 10⁹ (~4.3 billion) |
| 2⁸⁰ | 80-bit | ~2.4 × 10¹² (~2.4 trillion) |
| 2¹⁰⁰ | 100-bit | ~2.5 × 10¹⁵ |
| 2¹²⁰ | 120-bit | ~2.6 × 10¹⁸ |

### Hardware Throughput (Approximate)

Actual throughput depends heavily on hardware, but as a rough guide:

| Hardware | Approximate Throughput |
|---|---|
| Single CPU core (modern) | ~3–6 Mkeys/s |
| 8 CPU threads | ~25–50 Mkeys/s |
| NVIDIA RTX 2080 (GPU) | ~1–2 Gkeys/s |
| NVIDIA RTX 3090 (GPU) | ~2–4 Gkeys/s |

Combining multiple GPUs and CPU threads scales linearly — all walkers share the same hash table, so the expected time decreases proportionally with the total throughput.

### Tuning Tips

- **`-d dpBit`:** Increasing `dpBit` reduces hash table memory usage but increases variance in search time. The auto-tuned value is usually optimal; adjust only when memory is constrained.
- **`-g gx,gy`:** Default grid size is `2 × (SM count)` blocks by `2 × (cores per SM)` threads. On some GPUs, tuning this manually can improve occupancy.
- **`-ws`:** Saving kangaroo states makes work files larger but allows more efficient resumption — the saved kangaroos continue walking exactly where they left off rather than starting fresh.
- **Multiple keys:** Listing multiple public keys in the input file does not multiply the search time — all keys are checked against each distinguished point found during a single walk.

---

## License

Kangaroo is free software, distributed under the **GNU General Public License v3.0**.

```
Copyright (c) 2020 Jean Luc PONS

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, version 3.

This program is distributed in the hope that it will be useful, but
WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
See the GNU General Public License for more details.
```

Full license text: [`LICENSE.txt`](LICENSE.txt)  
Original repository: <https://github.com/JeanLucPons/Kangaroo>
