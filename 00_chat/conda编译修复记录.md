# ORFS 在 conda `openroad` 环境下的编译修复记录

> 时间:2026-08-07　|　机器:Rocky Linux 9.8 / 128 核 / 1.5 TB 内存　|　conda 环境:`openroad`
> 目标:用 `build_openroad.sh --local` 在 conda 环境里编译 OpenROAD-flow-scripts

## 最终结果

| 工具 | 状态 | 版本 |
|------|------|------|
| **openroad** | ✅ 编译成功 + 可运行 | `26Q3-771-g7cfb2105c9` |
| **yosys** | ✅ 编译成功 + 可运行 | `Yosys 0.67+` |
| kepler-formal | ⚠️ 未完成(可选形式验证工具,见末尾) | — |

二进制位置:`tools/install/OpenROAD/bin/openroad`、`tools/install/yosys/bin/yosys`

---

## 一、根本诊断:conda 环境为什么是空的

- `conda list -n openroad` 显示 **0 个包**。
- 原因:`/data/trd/kshi/code/conda/openroad.yml` 的 `dependencies:` 段是**空的**。
- `conda-meta/history` 显示曾执行 `conda env create -f openroad.yml`,因依赖段为空,conda **覆盖了原来可用的环境**,创建了一个空环境。
- 之前那次失败的 `build_openroad.log`(13:46)卡在 `ortools::fzn → bin/fzn-ortools`(老版 libortools 的打包 bug)。

> 结论:需要从头重建 conda 环境,并解决 conda-forge 库与 OpenROAD 构建假设之间的一系列不匹配。

---

## 二、完整的 conda 包清单(写入 `openroad.yml`)

项目根目录新建了 **`openroad.yml`**,内容(已更新为最终可用版):

```yaml
name: openroad
channels:
  - conda-forge
dependencies:
  - python=3.11
  - cmake>=3.31            # 实际装到 4.4.2
  - c-compiler             # gcc_linux-64 + sysroot
  - cxx-compiler           # gxx_linux-64(gcc 14),关键:解决 ABI
  - ninja
  - ccache
  - swig>=4.3
  - bison>=3.8
  - flex>=2.6
  - pkg-config             # 关键:让 or-tools 经 pkg-config 找到 Clp/Cbc
  - libboost-devel>=1.85
  - fmt
  - spdlog>=1.15
  - gtest>=1.14
  - eigen=3.4
  - yaml-cpp>=0.8
  - zlib
  - libortools>=9.10       # 9.15;自动带 coin-or-clp/cbc/cgl/osi/utils、absl、re2、protobuf
  - capnproto              # kepler-formal/naja 需要
```

**需从源码编译进 conda 环境的库(conda-forge 无可用/兼容版本):**
- CUDD 3.0.0(OpenSTA 需要)
- lemon(OpenROAD fork,C++20 已修;gpl/cts/dpl 需要)

**用系统的:** tcl(`/usr/lib64/libtcl.so`,8.6)

---

## 三、逐个问题与修法(共 9 个)

### 1. OR-Tools 找不到 `Clp`
- 现象:`ortools could not be found because dependency Clp could not be found`
- 原因:`libortools` 的 cmake 配置经 `find_dependency(Clp)` → `FindClp.cmake` 用 pkg-config 找 clp;但 conda 激活后 **`PKG_CONFIG_PATH` 为空**,系统 pkg-config 不知道 conda 里的 `.pc`。
- 修法:装 conda-forge **`pkg-config`**。它的激活钩子(实际是 conda pkg-config 自带 conda 前缀搜索路径)让 clp/cbc 可达。

### 2. OR-Tools 包名纠正
- conda-forge 上 C++ OR-Tools 不叫 `ortools`/`or-tools`,而是 **`libortools`**。版本 9.4/9.5/9.6/9.15。
- libortools 9.15 已**不再**引用旧版那个缺失的 `bin/fzn-ortools`(改用存在的 `fzn-cp-sat`/`solve`),所以原 fzn bug 在 9.15 已修复,无需占位文件。

### 3. Boost 静态/共享不匹配
- 现象:`boost_serialization_FOUND = FALSE ... No suitable build variant (shared, Boost_USE_STATIC_LIBS=ON)`
- 原因:`src/CMakeLists.txt` 默认 `Boost_USE_STATIC_LIBS ON`(非系统 boost 时);conda libboost 只有共享库。
- 修法:编译传 **`-DUSE_SYSTEM_BOOST=ON`**(使 `Boost_USE_STATIC_LIBS OFF`,用 conda 共享 boost)。

### 4. CUDD 缺失(OpenSTA 需要)
- 现象:`CUDD_LIB` NOTFOUND,`src/sta/CMakeLists.txt` 无条件引用 → 致命错误。
- 注:`CUDD` 对 OpenROAD 功能可选,但构建必需;conda-forge 无 cudd 包。
- 修法:按 `etc/DependencyInstaller.sh` 方式,从源码编译 CUDD 3.0.0 进 `$CONDA_PREFIX`:
  ```bash
  cd /tmp && git clone --depth=1 -b 3.0.0 https://github.com/The-OpenROAD-Project/cudd.git
  cd cudd && autoreconf -fi && ./configure --prefix="$CONDA_PREFIX" --enable-shared --enable-static
  make -j 16 && make install
  ```
  编译时传 `-DCUDD_DIR=$CONDA_PREFIX`。

### 5. C++ ABI 不匹配(关键)
- 现象:链接报 `undefined reference to ...@GLIBCXX_3.4.30`、`__cxa_call_terminate@CXXABI_1.3.15`(来自 conda `libspdlog.so`)。
- 原因:conda 的 C++ 库(spdlog/boost/fmt/absl 等)用更新的 gcc 编译,需要 `GLIBCXX_3.4.30+`;而系统 **gcc 11.5** 的 libstdc++ 只到 `3.4.29`,且系统 g++ 强行链接自己的旧 libstdc++。
- 修法:装 conda 编译器 **`c-compiler` + `cxx-compiler`**(gcc 14),编译/链接都用 conda 的 libstdc++(6.0.35,含所需符号),ABI 一致。
  - 激活后 `CC=x86_64-conda-linux-gnu-cc`,`CXX=x86_64-conda-linux-gnu-c++`。

### 6. slang 误用 conda fmt(关键)
- 现象:slang 编译报 `'format' is not a member of 'fmt'; did you mean 'std::format'?`
- 原因:`third-party/CMakeLists.txt:88` 直接 `add_subdirectory(slang-elab/third_party/slang)`(绕过 slang-elab 包装);slang 经 `FIND_PACKAGE_ARGS` 的 `find_package(fmt)` 找到了 **conda fmt 12.2.0**,而其 `<fmt/core.h>` 没有 `fmt::format`(slang 代码是针对自带 fmt 写的)。同时 conda `Boost::headers` 把 `$CONDA_PREFIX/include` 带进 slang 编译路径,盖过 slang 自带 fmt 头。
- 修法:改 **`tools/OpenROAD/third-party/CMakeLists.txt`**,在 slang 的 `add_subdirectory` 前后加(目录作用域,只影响 slang,不影响顶层 spdlog):
  ```cmake
  set(CMAKE_DISABLE_FIND_PACKAGE_fmt ON)
  set(CMAKE_DISABLE_FIND_PACKAGE_Boost ON)
  add_subdirectory(slang-elab/third_party/slang EXCLUDE_FROM_ALL)
  unset(CMAKE_DISABLE_FIND_PACKAGE_fmt)
  unset(CMAKE_DISABLE_FIND_PACKAGE_Boost)
  ```
  → slang 改用自带 fmt + vendored boost,彻底隔绝 conda 头。

### 7. odb Python SWIG 生成 Python2 代码
- 现象:`odbPYTHON_wrap.cxx` 报 `PyString_FromString` / `PyInt_FromLong` 未声明(Python2 API,Py3 已删)。
- 修法:传 **`-DBUILD_PYTHON=OFF`**(OpenROAD 是 Tcl 工具,不需要 Python 绑定;选项在 `src/CMakeLists.txt:139`)。

### 8. conda lemon 与 C++20 不兼容
- 现象:`lemon/bits/array_map.h` 报 `'std::allocator' has no member named 'construct/destroy'`。
- 原因:conda vanilla lemon 1.3.1 用了 `allocator.construct/destroy` 成员(C++17 弃用、**C++20 移除**)。
- 修法:移除 conda lemon,改用 **OpenROAD 自己的 lemon-graph fork**(已为 C++20 打补丁,用 `AllocatorTraits::construct/destroy`)编进 `$CONDA_PREFIX`:
  ```bash
  conda remove -n openroad -y lemon
  cd /tmp && git clone --depth=1 -b 1.3.1 https://github.com/The-OpenROAD-Project/lemon-graph.git
  cd lemon-graph
  # fork 的 CMakeLists 是 cmake 2.8 时代,需 patch 兼容 cmake 4:
  sed -i '1s/.*/cmake_minimum_required(VERSION 3.5)/; /CMAKE_POLICY(SET CMP0048 OLD)/d' CMakeLists.txt
  cmake -DCMAKE_INSTALL_PREFIX="$CONDA_PREFIX" -DCMAKE_POLICY_VERSION_MINIMUM=3.5 -DLEMON_ENABLE_GLPK=OFF -B build .
  cmake --build build --target install -j 16
  ```
  注:该 fork 产出的库名是 `libemon.a`(不是 liblemon),`LEMONConfig.cmake` 引用的就是它,`find_package(LEMON)` 能正常找到。

### 9. kepler-formal/naja 缺 CapnProto
- 现象:`Could not find a package configuration file provided by "CapnProto"`
- 修法:装 conda-forge **`capnproto`**。

---

## 四、最终可用的编译命令

```bash
conda activate openroad
export PKG_CONFIG_PATH="$CONDA_PREFIX/lib/pkgconfig:${PKG_CONFIG_PATH:-}"
./build_openroad.sh --local --no_init -t 100 \
  --openroad-args "-DUSE_SYSTEM_BOOST=ON -DCUDD_DIR=$CONDA_PREFIX -DBUILD_PYTHON=OFF"
```

关键参数说明:
- `--local`:本地编译(非 Docker)。
- `--no_init`:跳过子模块联网初始化(子模块已就位时用,省时省网)。
- `-t 100`:用 100 核(机器 128 核,留余量)。
- `--openroad-args "..."`(注意:**空格分隔**,值在下一个参数;`=` 形式会被当未知参数):
  - `-DUSE_SYSTEM_BOOST=ON`:用 conda 共享 boost。
  - `-DCUDD_DIR=$CONDA_PREFIX`:指定 CUDD 位置。
  - `-DBUILD_PYTHON=OFF`:跳过 odb Python 绑定。

> 运行 openroad/yosys 时需 conda 库在搜索路径:`export LD_LIBRARY_PATH="$CONDA_PREFIX/lib:${LD_LIBRARY_PATH:-}"`(或 `source env.sh`)。

---

## 五、改动/新增的文件清单(都在本项目目录内)

1. **`openroad.yml`**(项目根,新建)—— 完整 conda 包清单(见第二节)。
2. **`tools/OpenROAD/third-party/CMakeLists.txt`**(修改)—— 第 80~93 行加 `CMAKE_DISABLE_FIND_PACKAGE_fmt/Boost` 块(见问题 6)。
3. **conda 环境内从源码编译的库**:
   - CUDD 3.0.0:`$CONDA_PREFIX/lib/libcudd.so*`、`libcudd.a`、`include/cudd.h`
   - lemon-graph fork:`$CONDA_PREFIX/lib/libemon.a`、`include/lemon/`、`share/lemon/cmake/LEMONConfig.cmake`
4. **conda 环境新增包**:pkg-config、c-compiler、cxx-compiler、libortools(+coin-or-*)、capnproto、tbb 等。

> ⚠️ 注意:`tools/OpenROAD/third-party/CMakeLists.txt` 的改动在下次 `git submodule update` 时可能被覆盖,需要时重新应用。

---

## 六、验证

```bash
source /users/G01/kshi/kshi/tool/anaconda3/etc/profile.d/conda.sh && conda activate openroad
export LD_LIBRARY_PATH="$CONDA_PREFIX/lib:${LD_LIBRARY_PATH:-}"
tools/install/OpenROAD/bin/openroad -version   # → 26Q3-771-g7cfb2105c9
tools/install/yosys/bin/yosys -V               # → Yosys 0.67+
```

---

## 七、未完成:kepler-formal(可选)

kepler-formal 是形式验证工具,**不属于 RTL→GDSII 主流程**。`build_openroad.sh` 会无条件构建它(yosys 之后),它卡在自身两个问题上:

1. **TBB API 不匹配**:kepler-formal `#include <tbb/concurrent_unordered_map.h>`(旧 TBB 接口),conda tbb 2023 是 oneTBB,无 `tbb/` 头(且 conda tbb 包似乎只有运行库,头文件在别处)。
2. **Glucose / yaml-cpp 链接错误**:`undefined reference to Glucose::*`、`YAML::*`(疑似 kepler-formal 的 `-flto` 导致)。

这两个都需改 **kepler-formal 自身源码**(独立子模块 `tools/kepler-formal`),尚未处理。

---

## 八、心得:为什么 conda 路这么多坑

OpenROAD 官方依赖安装(`etc/DependencyInstaller.sh`)是**从源码编译** boost/spdlog/absl/lemon/or-tools/cudd 等到一个 prefix,与其 C++20 构建假设一致。而 conda-forge 的库:
- 为各自标准构建,不保证兼容 OpenROAD 的 C++20(如 lemon 的 allocator、slang 的 fmt);
- 版本/打包各有 bug(libortools fzn、tbb 头路径);
- 全局 include 路径会盖过 OpenROAD 自带的第三方头(slang fmt)。

所以 conda 方案需要逐个隔离/替换,如本记录所示。若追求省心,官方推荐 **Docker 或预编译二进制**(`docs/user/BuildWithDocker.md`、`BuildWithPrebuilt.md`)。
