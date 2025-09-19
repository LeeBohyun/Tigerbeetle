# How to Run TigerBeetle

TigerBeetle is a distributed financial accounting database designed for mission-critical workloads.  
This guide shows how to build and run benchmarks locally.  

Repository: [tigerbeetle/tigerbeetle](https://github.com/tigerbeetle/tigerbeetle)

---

## Prerequisites
- Linux (x86_64)
- [Zig compiler](https://ziglang.org/download/) (v0.13.0 or newer)
- Root access (for benchmarking with raw devices)

---

## 1. Install Zig

Check the latest release at [ziglang.org/download](https://ziglang.org/download/).  
Example for v0.13.0:

```bash
cd ~
wget https://ziglang.org/download/0.13.0/zig-linux-x86_64-0.13.0.tar.xz
tar xf zig-linux-x86_64-0.13.0.tar.xz
echo 'export PATH="$HOME/zig-linux-x86_64-0.13.0:$PATH"' >> ~/.bashrc
source ~/.bashrc
zig version
```

## 2. Clone and Build TigerBeetle
```bash
git clone https://github.com/tigerbeetle/tigerbeetle.git
cd tigerbeetle

# Build with system Zig
zig build -Drelease=true
```

or use a bundled Zig version:
```bash
# Download the compiler
bash zig/download.sh

# Build with verification enabled
./zig/zig build --release -Dconfig_verify=true
```
## 3. Run Benchmark
```bash
# Wipe the target device
sudo blkdiscard /dev/nvme1n1

# Run benchmark
./tigerbeetle benchmark \
  --cache-grid=32GiB \
  --transfer-count=400000 \
  --file=/dev/nvme1n1
```
