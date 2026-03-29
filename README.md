# Oxion

A universal mass spectrometry file reader. Fast, cross-platform, no .NET required.

Written in Rust, Oxion parses Thermo RAW, Shimadzu LCD, and mzML files directly, achieving **100-150x faster scan decoding** and **8-10x faster XIC extraction** compared to the official .NET RawFileReader library.

## Installation

### CLI

Download the latest binary from [Releases](https://github.com/EstrellaXD/oxion/releases):

| Platform | Download |
|----------|----------|
| Linux x86_64 | `ms-reader-x86_64-unknown-linux-gnu.tar.gz` |
| Linux aarch64 | `ms-reader-aarch64-unknown-linux-gnu.tar.gz` |
| macOS x86_64 | `ms-reader-x86_64-apple-darwin.tar.gz` |
| macOS Apple Silicon | `ms-reader-aarch64-apple-darwin.tar.gz` |
| Windows x64 | `ms-reader-x86_64-pc-windows-msvc.zip` |

```bash
# Example: macOS Apple Silicon
tar xzf ms-reader-aarch64-apple-darwin.tar.gz
./ms-reader info sample.raw
```

### GUI (RAW-to-mzML Converter)

Download `raw-converter-*` from [Releases](https://github.com/EstrellaXD/oxion/releases):

- macOS: `.tar.gz` (contains `.app`)
- Linux: `.deb`
- Windows: `.msi` or `-setup.exe`

### Python

```bash
pip install rawfilereaders
```

Or download `.whl` from [Releases](https://github.com/EstrellaXD/oxion/releases) for offline install.

## Supported Formats

| Format | Extension | Status |
|--------|-----------|--------|
| Thermo RAW | `.raw` | Full support (v57-66) |
| Shimadzu LCD | `.lcd` | Read support |
| mzML | `.mzml`, `.mzml.gz` | Full support |

## CLI Usage

### File information

```bash
ms-reader info sample.raw
```

### TIC export

```bash
ms-reader tic sample.raw -o tic.csv
```

### XIC extraction

```bash
# Single target, 5 ppm tolerance
ms-reader xic sample.raw --mz 524.2644 --ppm 5.0

# MS1 scans only (faster for DDA data)
ms-reader xic sample.raw --mz 524.2644 --ms1-only

# Multiple targets
ms-reader xic sample.raw --mz 524.2644 --mz 445.12 --ms1-only
```

### Batch XIC across multiple files

```bash
ms-reader batch-xic \
    -f file1.raw -f file2.raw \
    --mz 524.2644 --mz 445.12 \
    --rt-resolution 0.01 -o output.csv
```

### Single scan export

```bash
ms-reader scan sample.raw -n 1
```

### RAW to mzML conversion

```bash
# Single file
ms-reader convert sample.raw -o output.mzML

# Folder conversion (all .raw files, parallel)
ms-reader convert ./raw_files/ -o ./mzml_output/

# Custom precision and compression
ms-reader convert sample.raw --mz-bits 32 --intensity-bits 32 --compression zlib
```

## Python Usage

```python
import rawfilereaders

raw = rawfilereaders.RawFile("sample.raw")
scan = raw.scan(1)
print(f"MS level: {scan.ms_level}, points: {len(scan.mz)}")
```

## Performance

Benchmarked on Apple Silicon against Thermo .NET RawFileReader via Mono/pythonnet:

| Operation | Oxion | .NET | Speedup |
|-----------|-------|------|---------|
| Full scan decode (73K scans) | 10 ms | 7,067 ms | **700x** |
| XIC extraction (2000 targets) | 33 ms | 4,200 ms | **127x** |
| Batch XIC (10 targets) | ~0.7 ms | N/A | Near-zero marginal cost |
| TIC (index-based) | < 1 ms | N/A | No scan decode needed |

## License

Proprietary. Distributed as pre-compiled binaries and Python wheels.
