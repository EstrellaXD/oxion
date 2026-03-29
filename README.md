# Oxion

A pure Rust reader for Thermo Scientific RAW mass spectrometry files. No .NET runtime or Mono required.

Parses the Thermo RAW binary format directly, achieving **100-150x faster scan decoding** and **8-10x faster XIC extraction** compared to the official .NET RawFileReader library.

## Requirements

- [Rust](https://www.rust-lang.org/tools/install) 1.70+ (2021 edition)
- For Python bindings: Python 3.8+ and [maturin](https://github.com/PyO3/maturin)

## Building

```bash
cargo build --release
```

This builds all workspace crates including the core library and the CLI tool.

## CLI Usage

The `oxion-cli` crate provides a command-line interface for common operations.

### File information

```bash
cargo run --release -p oxion-cli -- info sample.raw
```

Output includes instrument model, serial number, scan count, RT range, and mass range.

### TIC export

```bash
# Print to stdout
cargo run --release -p oxion-cli -- tic sample.raw

# Save to file
cargo run --release -p oxion-cli -- tic sample.raw -o tic.csv
```

### XIC extraction

```bash
# Single target, 5 ppm tolerance
cargo run --release -p oxion-cli -- xic sample.raw --mz 524.2644 --ppm 5.0

# MS1 scans only (faster for DDA data)
cargo run --release -p oxion-cli -- xic sample.raw --mz 524.2644 --ms1-only

# Multiple targets in a single pass
cargo run --release -p oxion-cli -- xic sample.raw --mz 524.2644 --mz 445.12 --ms1-only
```

### Batch XIC across multiple files

Extracts chromatograms from multiple RAW files, aligns to a common RT grid, and outputs a CSV matrix.

```bash
cargo run --release -p oxion-cli -- batch-xic \
    -f file1.raw -f file2.raw \
    --mz 524.2644 --mz 445.12 \
    --rt-resolution 0.01 -o output.csv
```

### Single scan export

```bash
# Export scan as JSON
cargo run --release -p oxion-cli -- scan sample.raw -n 1

# Show trailer extra data for a scan
cargo run --release -p oxion-cli -- trailer sample.raw -n 1
```

### RAW to mzML conversion

```bash
# Single file
cargo run --release -p oxion-cli -- convert sample.raw

# Specify output path
cargo run --release -p oxion-cli -- convert sample.raw -o output.mzML

# Folder conversion (all .raw files)
cargo run --release -p oxion-cli -- convert ./raw_files/ -o ./mzml_output/

# Custom precision and compression
cargo run --release -p oxion-cli -- convert sample.raw \
    --mz-bits 32 --intensity-bits 32 --compression zlib
```

### Benchmarking

```bash
# Full scan decode benchmark
cargo run --release -p oxion-cli -- benchmark sample.raw --parallel --mmap

# XIC benchmark
cargo run --release -p oxion-cli -- benchmark sample.raw --mmap --xic
```

## Library Usage (Rust)

Add `oxion` to your `Cargo.toml`:

```toml
[dependencies]
oxion = { path = "crates/oxion" }
```

```rust
use oxion::RawFile;

// Open a RAW file (use open_mmap for better performance on large files)
let raw = RawFile::open("sample.raw")?;

// File metadata
let meta = raw.metadata();
println!("Instrument: {}", meta.instrument_model);
println!("Scans: {}", raw.n_scans());
println!("RT range: {:.2}-{:.2} min", raw.start_time(), raw.end_time());

// Read a single scan
let scan = raw.scan(1)?;
println!("MS level: {:?}, RT: {:.4} min", scan.ms_level, scan.rt);
println!("Centroid peaks: {}", scan.centroid_mz.len());

// TIC from scan index (no decode needed, sub-millisecond)
let tic = raw.tic();

// XIC extraction
let xic = raw.xic(524.2644, 5.0)?;          // all scans
let xic = raw.xic_ms1(524.2644, 5.0)?;      // MS1 only (faster for DDA)

// Batch XIC: multiple targets in a single scan pass
let targets = vec![(524.2644, 5.0), (445.12, 5.0), (302.05, 5.0)];
let xics = raw.xic_batch_ms1(&targets)?;

// Parallel scan decode
let scans = raw.scans_parallel(1..raw.n_scans() + 1)?;

// Trailer extra data
let trailer = raw.trailer_extra(1)?;
```

## Python Bindings

Python bindings are provided via PyO3 in `crates/oxion-py/`.

### Installation

```bash
cd crates/oxion-py
pip install maturin
maturin develop --release
```

### Usage

```python
import oxion

raw = oxion.RawFile("sample.raw")
scan = raw.scan(1)
print(f"MS level: {scan.ms_level}, points: {len(scan.mz)}")
```

## Performance

Benchmarked on Apple Silicon against Thermo .NET RawFileReader via Mono/pythonnet:

| Operation | Rust (parallel+mmap) | .NET | Speedup |
|-----------|---------------------|------|---------|
| Full scan decode (227K scans) | 45 ms | 7,067 ms | **157x** |
| XIC extraction (MS1) | ~0.6 s | ~5.3 s | **~8x** |
| Batch XIC (10 targets) | ~0.66 s | N/A | Near-zero marginal cost per target |
| TIC (index-based) | < 1 ms | N/A | No scan decode needed |

See [BENCHMARK.md](BENCHMARK.md) for detailed results.

## Project Structure

```
crates/
  cfb-reader/              OLE2 compound file container reader
  oxion/                   Core library (parsing, scan decode, XIC)
  oxion-cli/               Command-line interface
  oxion-py/                PyO3 Python bindings
  oxion-gui/               Tauri v2 + Vue 3 GUI for RAW-to-mzML conversion
docs/                      Format specifications and reference
tools/
  ground-truth/            Python ground truth exporter
  ground-truth-exporter/   C# ground truth exporter
  hex-analyzer/            Binary format analysis tool
vendor/libs/               Thermo .NET DLLs (reference only)
test-data/                 Test RAW files (not in git)
```

## Documentation

- [FORMAT_SPEC.md](docs/FORMAT_SPEC.md) -- Binary format specification
- [SCAN_DATA_ENCODING.md](docs/SCAN_DATA_ENCODING.md) -- Scan data encoding details
- [OLE2_STRUCTURE.md](docs/OLE2_STRUCTURE.md) -- OLE2 container layout
- [VERSION_DIFFERENCES.md](docs/VERSION_DIFFERENCES.md) -- Version-specific differences
- [BENCHMARK.md](BENCHMARK.md) -- Performance benchmarks

## License

The Thermo RawFileReader DLLs in `vendor/libs/` are subject to the [Thermo Fisher license](LICENSE.md). The Rust source code in this repository is private and proprietary.

RawFileReader reading tool. Copyright (c) 2016 by Thermo Fisher Scientific, Inc. All rights reserved.
