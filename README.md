# Oxion

A universal mass spectrometry file reader. Fast, cross-platform, no .NET required.

Oxion reads Thermo RAW, Shimadzu LCD, and mzML files directly from their binary formats, achieving **up to 700x faster scan decoding** than the official .NET RawFileReader library. It provides a CLI tool, a desktop GUI converter, and Python bindings with NumPy integration.

## Table of Contents

- [Installation](#installation)
- [Supported Formats](#supported-formats)
- [CLI Reference](#cli-reference)
- [Python API](#python-api)
- [GUI Converter](#gui-converter)
- [Performance](#performance)
- [License](#license)

---

## Installation

### CLI

Download the latest binary for your platform from [Releases](https://github.com/EstrellaXD/oxion/releases):

| Platform | File | Install |
|----------|------|---------|
| **Linux x86_64** | `oxion-x86_64-unknown-linux-gnu.tar.gz` | `tar xzf oxion-*.tar.gz` |
| **Linux aarch64** | `oxion-aarch64-unknown-linux-gnu.tar.gz` | `tar xzf oxion-*.tar.gz` |
| **macOS Intel** | `oxion-x86_64-apple-darwin.tar.gz` | `tar xzf oxion-*.tar.gz` |
| **macOS Apple Silicon** | `oxion-aarch64-apple-darwin.tar.gz` | `tar xzf oxion-*.tar.gz` |
| **Windows x64** | `oxion-x86_64-pc-windows-msvc.zip` | Extract zip |

After extracting, optionally move `oxion` to a directory on your `PATH`.

### Python

```bash
pip install oxion
```

Requires Python 3.11+. Pre-built wheels are available for Linux (x86_64, aarch64), macOS (Intel, Apple Silicon), and Windows (x64). Wheels can also be downloaded from [Releases](https://github.com/EstrellaXD/oxion/releases) for offline install:

```bash
pip install oxion-*.whl
```

### GUI

Download the desktop converter from [Releases](https://github.com/EstrellaXD/oxion/releases):

| Platform | File |
|----------|------|
| macOS (Intel + Apple Silicon) | `oxion-gui-*.tar.gz` |
| Linux | `oxion-gui-*.deb` |
| Windows | `oxion-gui-*.msi` or `*-setup.exe` |

---

## Supported Formats

| Format | Extension | Read | Convert to mzML |
|--------|-----------|------|-----------------|
| Thermo RAW | `.raw` | Full (v57-66, all Orbitrap/LTQ/Astral instruments) | Yes |
| Shimadzu LCD | `.lcd` | Scans, chromatograms, MRM events | No |
| mzML | `.mzml`, `.mzml.gz` | Full (indexed + non-indexed, gzip) | N/A |

### Thermo RAW Details

- Supports format versions 57 through 66 (covering instruments from LTQ through Orbitrap Astral)
- Reads both 32-bit and 64-bit address modes
- Decodes centroid, profile, FT, and LT packet types
- Extracts trailer extra metadata (86+ fields including AGC, injection time, charge state)
- No .NET, Mono, or Wine required

---

## CLI Reference

```
oxion <COMMAND> [OPTIONS]
```

### `info` - File Information

Display instrument model, scan count, RT range, mass range, and trailer field names.

```bash
oxion info sample.raw
oxion info data.mzML
```

### `scan` - Single Scan Export

Export a single scan as JSON with m/z and intensity arrays.

```bash
oxion scan sample.raw -n 1          # First scan
oxion scan sample.raw -n 5000       # Scan number 5000
```

### `tic` - Total Ion Chromatogram

Export the TIC as a two-column CSV (rt, intensity). Sub-millisecond extraction from the scan index (no scan decoding needed).

```bash
oxion tic sample.raw                # Print to stdout
oxion tic sample.raw -o tic.csv     # Save to file
```

### `xic` - Extracted Ion Chromatogram

Extract one or more XIC traces from a file.

```bash
# Single target at 5 ppm (default)
oxion xic sample.raw --mz 524.2644

# Multiple targets in one pass (shared scan iteration)
oxion xic sample.raw --mz 524.2644 --mz 445.12 --mz 302.05

# MS1 only — skips MS2 scans, much faster for DDA data
oxion xic sample.raw --mz 524.2644 --ms1-only

# Custom tolerance
oxion xic sample.raw --mz 524.2644 --ppm 10.0

# Save to file
oxion xic sample.raw --mz 524.2644 -o xic.csv
```

### `batch-xic` - Multi-File Batch XIC

Extract XIC traces across multiple files, align to a common RT grid, and output a CSV matrix.

```bash
oxion batch-xic \
    -f file1.raw -f file2.raw -f file3.raw \
    --mz 524.2644 --mz 445.12 \
    --ppm 5.0 \
    --rt-resolution 0.01 \
    -o batch_output.csv

# With RT range filter
oxion batch-xic \
    -f *.raw \
    --mz 524.2644 \
    --rt-range "2.0,15.0" \
    -o filtered.csv
```

### `ms2-spectra` - DDA Fragment Extraction

Find all MS2 scans matching a precursor m/z and export their fragment spectra.

```bash
oxion ms2-spectra sample.raw --mz 524.2644 --ppm 10.0 -o fragments.csv
```

### `convert` - RAW to mzML Conversion

Convert Thermo RAW files to indexed mzML format.

```bash
# Single file (output: sample.mzML in same directory)
oxion convert sample.raw

# Specify output path
oxion convert sample.raw -o output.mzML

# Folder conversion (parallel, all .raw files)
oxion convert ./raw_files/ -o ./mzml_output/

# Options
oxion convert sample.raw \
    --mz-bits 32 \
    --intensity-bits 32 \
    --compression zlib \
    --ms1-only \
    --min-intensity 100 \
    --no-index
```

### `trailer` - Trailer Extra Data

Display raw trailer metadata for a specific scan (Thermo RAW only).

```bash
oxion trailer sample.raw -n 1
```

### `benchmark` - Performance Test

Benchmark scan decoding speed.

```bash
oxion benchmark sample.raw --mmap              # Sequential decode
oxion benchmark sample.raw --mmap --parallel    # Parallel decode
oxion benchmark sample.raw --mmap --xic         # XIC extraction benchmark
```

---

## Python API

### Opening Files

```python
import oxion

# Auto-detect format from extension
raw = oxion.open("sample.raw")           # Thermo RAW
mzml = oxion.open("data.mzML")           # mzML
lcd = oxion.open("sample.lcd")            # Shimadzu LCD

# RAW file with memory-mapped I/O (faster for large files)
raw = oxion.open("sample.raw", mmap=True)

# Or use the format-specific class directly
raw = oxion.RawFile("sample.raw", mmap=True)
```

### File Metadata

```python
raw = oxion.RawFile("sample.raw")

raw.n_scans              # Total number of scans
raw.first_scan           # First scan number
raw.last_scan            # Last scan number
raw.start_time           # Start RT in minutes
raw.end_time             # End RT in minutes
raw.instrument_model     # Instrument model string
raw.sample_name          # Sample name from acquisition
raw.version              # RAW format version (57-66)
```

### Reading Scans

```python
# Get scan data as NumPy arrays
mz, intensity = raw.scan(1)

# Get scan metadata (no array decoding)
info = raw.scan_info(1)
info.scan_number         # 1
info.rt                  # Retention time in minutes
info.ms_level            # 1, 2, 3, ...
info.polarity            # "positive" or "negative"
info.tic                 # Total ion current
info.base_peak_mz        # Base peak m/z
info.base_peak_intensity # Base peak intensity
info.filter_string       # e.g. "FTMS + p NSI Full ms [100.00-1000.00]"
info.precursor_mz        # Precursor m/z (MS2+ only, None for MS1)
info.precursor_charge    # Charge state (MS2+ only, None for MS1)

# Read all MS1 scans in parallel
all_ms1 = raw.all_ms1_scans(progress=True)  # list of (mz, intensity) tuples
```

### Chromatograms

```python
# TIC (sub-millisecond, from scan index)
rt, intensity = raw.tic()

# XIC (single target)
rt, intensity = raw.xic(524.2644, ppm=5.0)

# XIC restricted to MS1 scans (faster for DDA)
rt, intensity = raw.xic_ms1(524.2644, ppm=5.0)

# Batch XIC (multiple targets, single scan pass)
targets = [(524.2644, 5.0), (445.12, 5.0), (302.05, 10.0)]
results = raw.xic_batch_ms1(targets, progress=True)
for rt, intensity in results:
    print(f"  {len(rt)} points")
```

### DDA / DIA Analysis

```python
# Acquisition type detection
raw.acquisition_type()     # "dda", "dia", "ms1_only", or "mixed"

# MS level queries
raw.ms_level_of_scan(100)  # 1 or 2
raw.is_ms2_scan(100)       # True/False
raw.scan_numbers_by_level(2)  # [4, 5, 6, 8, ...]

# Precursor information
precursors = raw.precursor_list()     # Unique precursor m/z as NumPy array
parent = raw.parent_ms1_scan(5000)    # Parent MS1 scan number for scan 5000

# Find MS2 scans for a precursor
ms2_scans = raw.ms2_scans_for_precursor(524.2644, tolerance_ppm=10.0)
for s in ms2_scans:
    print(f"  Scan {s.scan_number}, RT={s.rt:.2f}, CE={s.collision_energy}")

# All MS2 scan metadata (fast, no decoding)
all_ms2 = raw.all_ms2_scan_info()
```

### DIA Windows

```python
# Get unique isolation windows
windows = raw.isolation_windows()
for w in windows:
    print(w)  # IsolationWindow(center_mz=500.0, width=25.0, ce=30.0, activation=HCD)

# Get MS2 scans for a window
scans = raw.scans_for_window(windows[0])

# XIC within a DIA window
rt, intensity = raw.xic_ms2_window(524.2644, ppm=5.0, window=windows[0])
```

### Trailer Metadata

```python
# Available trailer field names
fields = raw.trailer_fields()  # ["Charge State", "Ion Injection Time (ms)", ...]

# Trailer data for a specific scan
data = raw.trailer_extra(1)    # dict: {"Charge State": "2", "Ion Injection Time (ms)": "35.00", ...}
```

### Progress Bars

Most long-running operations support `tqdm` progress bars:

```python
# Install tqdm for progress bar support
# pip install tqdm

rt, intensity = raw.xic(524.2644, progress=True)
results = raw.xic_batch_ms1(targets, progress=True)
scans = raw.all_ms1_scans(progress=True)
```

---

## GUI Converter

The desktop application provides drag-and-drop RAW-to-mzML conversion with a visual interface.

- Drag one or more `.raw` files onto the window
- Configure output settings (precision, compression, MS1-only filter)
- Click convert and monitor progress
- Output mzML files are written alongside the source files or to a chosen directory

---

## Performance

Benchmarked on Apple Silicon (M-series) with a 517 MB DDA file (73,306 scans):

| Operation | Oxion | .NET RawFileReader | Speedup |
|-----------|-------|--------------------|---------|
| Full scan decode (sequential) | 10 ms | 7,067 ms | **700x** |
| Full scan decode (parallel) | 6.6 ms | N/A | **1070x** vs .NET |
| XIC extraction (2000 targets) | 33 ms | 4,200 ms | **127x** |
| Batch XIC (10 targets) | 0.7 ms | N/A | Near-zero marginal cost |
| TIC from index | < 1 ms | N/A | No scan decode needed |
| RAW-to-mzML conversion | 2.5 s | ~30 s | **12x** |

### Why is Oxion fast?

- **Direct binary parsing** — reads the RAW format directly without .NET runtime overhead
- **Memory-mapped I/O** — leverages OS virtual memory for zero-copy file access
- **Zero-allocation hot paths** — XIC extraction reads raw bytes in-place with no heap allocations
- **Buffer reuse** — scan decoding reuses pre-allocated buffers, eliminating ~220k allocations per file
- **Parallel processing** — multi-core scan decoding and folder conversion via work-stealing
- **Optimized encoding** — mzML conversion reuses compression and base64 buffers across scans
- **LTO + codegen-units=1** — whole-program link-time optimization for maximum inlining

---

## License

Proprietary. Distributed as pre-compiled binaries and Python wheels.

For questions or bug reports, please open an [issue](https://github.com/EstrellaXD/oxion/issues).
