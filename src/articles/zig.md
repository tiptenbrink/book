# Zig

See [Zig wiki](https://github.com/ziglang/zig/wiki/Building-Zig-From-Source) for up to date info.

## LLVM

Make sure you update your sources to include the newest version, e.g. for Ubuntu Jammy you can put in the file `/etc/apt/sources.list.d/llvm.list`:

```
deb [arch=amd64] http://apt.llvm.org/jammy/ llvm-toolchain-jammy-21 main
deb-src [arch=amd64] http://apt.llvm.org/jammy/ llvm-toolchain-jammy-21 main
```

Then install LLVM and related tools with:

```
sudo apt install llvm-21 lvm-21-dev clang-21 lld-21 lldb-21 libclang libclang-21-dev liblld-21-dev
```

Then download the Zig repository and go to the commit you want and do:

```
mkdir build
cd build
cmake -DCMAKE_PREFIX_PATH=/usr/lib/llvm-21 -DCMAKE_BUILD_TYPE=Release ..
make install
```

This can take 5-10 minutes. There then exists `./build/stage3` (relative to Zig repo root), which contains a `bin` and `lib` folder.
