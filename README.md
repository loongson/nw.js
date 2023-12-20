# nw.js for loongarch64（交叉编译）

* nw.js社区项目地址 - https://nwjs.io/
# 当前分支
* **nw82**
-------
# nw.js 源码编译（交叉编译）
* **编译系统：debian 12**
* **编译平台：x86_64**
* **目标平台：loongarch64**
* **编译器：[binutils-gdb](https://sourceware.org/git/binutils-gdb.git)和[llvm-project](https://github.com/llvm/llvm-project.git)**
* **sysroot：[loongson](https://github.com/loongson)仓库的[build-tools](https://github.com/loongson/build-tools)中的CLFS（clfs-loongarch64-system-x.x-sysroot.squashfs）**
------
## 1.制作nw.js所需交叉编译工具链
### 1.1.编译binutils-gdb
```
git clone https://sourceware.org/git/binutils-gdb.git
cd binutils-gdb
mkdir build
cd build
../configure --prefix=/opt/loongarch64/toolchain --target=x86_64-linux-gnu --disable-nls --enable-shared --enable-64-bit-bfd --disable-werror
make && make install

rm ./* -rf
../configure --prefix=/opt/loongarch64/toolchain --target=loongarch64-linux-gnu --with-sysroot --disable-nls --enable-shared --enable-64-bit-bfd --disable-werror
make && make install
```
### 1.2.将交叉编译链接器加入环境变量
```
export PATH=$PATH:/opt/loongarch64/toolchain/bin
```
### 1.3.制作sysroot
```
mkdir -p /opt/loongarch64/sysroot
mount clfs-loongarch64-system-x.x-sysroot.squashfs /mnt
cp -raf /mnt /opt/loongarch64/sysroot/
umount /mnt
```
* **sysroot保留以下目录结构：**
```
.
|-- lib -> usr/lib
|-- lib64 -> usr/lib64
`-- usr
    |-- include
    |-- lib
    |-- lib64
    `-- share
```
### 1.4.编译llvm-project(18分支）
```
git clone https://github.com/llvm/llvm-project.git
cd llvm-project
cmake -S llvm -B build -G "Ninja" -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_PROJECTS="clang" -DCMAKE_INSTALL_PREFIX=/opt/loongarch64/llvm-18 -DLLVM_DEFAULT_TARGET_TRIPLE="x86_64-linux-gnu;loongarch64-linux-gnu"
cd build
ninja && ninja install
```
* **继续在llvm-project目录下编译compiler-rt:**
```
mkdir build-compiler-rt
cd build-compiler-rt
cmake ../compiler-rt/ -G Ninja -DCMAKE_AR=/opt/loongarch64/llvm-18/bin/llvm-ar -DCMAKE_ASM_COMPILER_TARGET=loongarch64-unknown-linux-gnu -DCMAKE_ASM_FLAGS="-mcmodel=medium -mabi=lp64d --target=loongarch64-linux-gnu --sysroot=/opt/loongarch64/sysroot" -DCMAKE_C_COMPILER=/opt/loongarch64/llvm-18/bin/clang -DCMAKE_C_COMPILER_TARGET=loongarch64-unknown-linux-gnu -DCMAKE_C_FLAGS="-mcmodel=medium -mabi=lp64d --target=loongarch64-linux-gnu --sysroot=/opt/loongarch64/sysroot" -DLLVM_ENABLE_PER_TARGET_RUNTIME_DIR=ON -DCMAKE_NM=/opt/loongarch64/llvm-18/bin/llvm-nm -DCMAKE_RANLIB=/opt/loongarch64/llvm-18/bin/llvm-ranlib  -DCOMPILER_RT_BUILD_BUILTINS=ON  -DCOMPILER_RT_BUILD_LIBFUZZER=OFF -DCOMPILER_RT_BUILD_MEMPROF=OFF -DCOMPILER_RT_BUILD_PROFILE=ON -DCOMPILER_RT_BUILD_SANITIZERS=OFF -DCOMPILER_RT_BUILD_XRAY=OFF -DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON -DLLVM_CMAKE_DIR=/opt/loongarch64/llvm-18 -DCMAKE_INSTALL_PREFIX=/opt/loongarch64/llvm-18/lib/clang/18
ninja && ninja install
rm ./* -rf
```
* **编译x64的compiler-rt:**
```
cmake ../compiler-rt/ -G Ninja -DCMAKE_AR=/opt/loongarch64/llvm-18/bin/llvm-ar -DCMAKE_ASM_COMPILER_TARGET=x86_64-unknown-linux-gnu -DCMAKE_ASM_FLAGS="" -DCMAKE_C_COMPILER=/opt/loongarch64/llvm-18/bin/clang -DCMAKE_C_COMPILER_TARGET=x86_64-unknown-linux-gnu -DCMAKE_C_FLAGS="" -DLLVM_ENABLE_PER_TARGET_RUNTIME_DIR=ON -DCMAKE_NM=/opt/loongarch64/llvm-18/bin/llvm-nm -DCMAKE_RANLIB=/opt/loongarch64/llvm-18/bin/llvm-ranlib  -DCOMPILER_RT_BUILD_BUILTINS=ON  -DCOMPILER_RT_BUILD_LIBFUZZER=OFF -DCOMPILER_RT_BUILD_MEMPROF=OFF -DCOMPILER_RT_BUILD_PROFILE=ON -DCOMPILER_RT_BUILD_SANITIZERS=OFF -DCOMPILER_RT_BUILD_XRAY=OFF -DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON -DLLVM_CMAKE_DIR=/opt/loongarch64/llvm-18 -DCMAKE_INSTALL_PREFIX=/opt/loongarch64/llvm-18/lib/clang/18
ninja && ninja install
```
## 2.拉取nw.js源码及依赖
### 2.1.获取google的代码拉取工具depot_tools并添加到环境变量
```
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH=$PATH:/path/to/depot_tools
cd depot_tools
./update_depot_tools
```
### 2.2.拉取代码
```
mkdir -p $HOME/nwjs
cd $HOME/nwjs
gclient config --name=src https://github.com/nwjs/chromium.src.git@origin/nw82
git clone -b nw82 https://github.com/nwjs/v8.git src/v8
git clone -b nw82 https://github.com/nwjs/node.git src/third_party/node-nw
git clone -b nw82 https://github.com/loongson/nw.js.git src/content/nw
gclient sync --with_branch_heads
```
### 2.3 将sysroot拷贝到build/linux目录下
```
cd $HOME/nwjs/src/build/linux
mkdir debian_bullseye_loong64-sysroot
cp -raf /opt/loongarch64/sysroot/* debian_bullseye_loong64-sysroot/
```
## 3.编译nw.js
### 3.1.生成编译工程文件
```
cd $HOME/nwjs/src
./buildtools/linux64/gn gen out/nw --args='clang_use_chrome_plugins=false treat_warnings_as_errors=false dcheck_always_on=false use_gold=false use_lld=false clang_base_path="/opt/loongarch64/llvm-18" is_debug=false is_component_build=false is_component_ffmpeg=true target_cpu="loong64" use_sysroot=false'
 PYTHONPATH=${PWD}/third_party/node-nw/tools/v8_gypfiles GYP_CHROMIUM_NO_ACTION=0 GYP_CROSSCOMPILE=1 ./build/gyp_chromium -I third_party/node-nw/common.gypi -D building_nw=1 -D clang=1 -D target_arch=loong64 -D clang_base_dir="/opt/loongarch64/llvm-18" third_party/node-nw/node.gyp --no-duplicate-basename-check
```
### 3.2.编译
* 编译nwjs
```
ninja -C out/nw nwjs
```
* 编译node
```
ninja -C out/Release node
ninja -C out/nw copy_node
```
* 编译组件
```
ninja -C out/nw credits.html
ninja -C out/nw nwjc
ninja -C out/nw payload
ninja -C out/nw chromedriver
```
### 3.3.打包nw.js
```
cd $HOME/nwjs/src
python content/nw/tools/package_binaries.py -p out/nw -a loong64 -n symbol
```
* **在out/nw/dist目录可以找到打包生成的文件，sdk版本打包需加参数`-m sdk`**
## License

`NW.js`'s code in this repo uses the MIT license, see our `LICENSE` file. To redistribute the binary, see [How to package and distribute your apps](https://github.com/nwjs/nw.js/wiki/How-to-package-and-distribute-your-apps)
