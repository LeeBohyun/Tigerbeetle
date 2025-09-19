# How to run tigerbeetle

- https://github.com/tigerbeetle/tigerbeetle

### Install zig
``bash
# Example for x86_64; check latest at https://ziglang.org/download/
cd ~
wget https://ziglang.org/download/0.13.0/zig-linux-x86_64-0.13.0.tar.xz
tar xf zig-linux-x86_64-0.13.0.tar.xz
echo 'export PATH="$HOME/zig-linux-x86_64-0.13.0:$PATH"' >> ~/.bashrc
source ~/.bashrc
zig version

 zig build -Drelease=true
``
  
``bash
# download the compiler
bash zig/download.sh

 ./zig/zig build --release -Dconfig_verify=true 

  sudo blkdiscard /dev/nvme1n1

  ./tigerbeetle benchmark --cache-grid=32GiB --transfer-count=400000  --file=/dev/nvme1n1

``
