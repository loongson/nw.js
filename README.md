# nw.js for loongarch64（交叉编译）

* nw.js社区项目地址 - https://nwjs.io/
# 目前支持分支
* **nw62**
-------
# nw.js 源码编译（交叉编译）
* **编译平台：x86_64**
* **目标平台：loongarch64**
* **编译器：[loongson](https://github.com/loongson)仓库的[binutils-gdb](https://github.com/loongson/binutils-gdb)和[llvm-project](https://github.com/loongson/llvm-project)**
* **sysroot：[loongson](https://github.com/loongson)仓库的[build-tools](https://github.com/loongson/build-tools)中的CLFS（loongarch64-clfs-system-x.x.tar.bz2）**
------
## 1.制作nw.js所需交叉编译工具链
### 1.1.编译llvm-project
```
git clone https://github.com/loongson/llvm-project.git
cd llvm-project
cmake -S llvm -B build -G "Ninja" -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_PROJECTS="clang" -DCMAKE_INSTALL_PREFIX=/opt/loongarch64/toolchain -DLLVM_DEFAULT_TARGET_TRIPLE="x86_64-linux-gnu;loongarch64-linux-gnu"
cd build
ninja && ninja install
```
### 1.2.编译binutils-gdb
```
git clone https://github.com/loongson/binutils-gdb.git
cd binutils-gdb
mkdir build
cd build
../configure --prefix=/opt/loongarch64/toolchain --target=x86_64-linux-gnu --disable-nls --enable-shared --enable-64-bit-bfd --disable-werror
make && make install

rm ./* -rf
../configure --prefix=/opt/loongarch64/toolchain --target=loongarch64-linux-gnu --disable-nls --enable-shared --enable-64-bit-bfd --disable-werror
make && make install
```
### 1.3.将交叉编译工具链加入环境变量
```
export PATH=$PATH:/opt/loongarch64/toolchain/bin
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
gclient config --name=src https://github.com/nwjs/chromium.src.git@origin/nw62
git clone -b nw62 https://github.com/nwjs/v8.git src/v8
git clone -b nw62 https://github.com/nwjs/node.git src/third_party/node-nw
git clone -b nw62 https://github.com/loongson/nw.js.git src/content/nw
gclient sync --with_branch_heads
```
### 2.3 制作sysroot
```
cd $HOME/nwjs/src/build/linux
mkdir debian_sid_loong64-sysroot
tar xvf loongarch64-clfs-system-x.x.tar.bz2 -C debian_sid_loong64-sysroot
```
* **提示:loongarch64-clfs-system-x.x.tar.bz2下载链接在前面**
## 3.编译nw.js
### 3.1.生成编译工程文件
```
cd $HOME/nwjs/src
./buildtools/linux64/gn gen out/nw --args='clang_use_chrome_plugins=false enable_swiftshader=false angle_enable_swiftshader=false enable_swiftshader_vulkan=false treat_warnings_as_errors=false dcheck_always_on=false use_gold=false use_lld=false clang_base_path="/opt/loongarch64/toolchain" is_debug=false is_component_build=false is_component_ffmpeg=true target_cpu="loong64" use_sysroot=false'
GYP_CHROMIUM_NO_ACTION=0 ./build/gyp_chromium -I third_party/node-nw/common.gypi -D building_nw=1 -D clang=1 -D clang_base_dir="/opt/loongarch64/toolchain" third_party/node-nw/node.gyp
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
ninja -C out/nw minidump_stackwalk
```
### 3.3.打包nw.js
```
cd $HOME/nwjs/src
python content/nw/tools/package_binaries.py -p out/nw -a loong64 -n symbol
```
* **在out/nw/dist目录可以找到打包生成的文件，sdk版本打包需加参数`-m sdk`**
## License

`NW.js`'s code in this repo uses the MIT license, see our `LICENSE` file.

