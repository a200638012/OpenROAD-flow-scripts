# 本地构建提速 —— 对话结果汇总

环境实测:128 核 CPU,1.5 TB 内存(可用 ~1.4 TB),`ccache` 未安装,
`tools/OpenROAD/` 子模块当前未初始化(为空)。内存充足,无需降线程。

## 1. 用指定核心数编译(例如 100 核)

`-t` / `--threads` 会同时传给 OpenROAD(`Build.sh -threads=`)
和 Yosys/KLayout 的 `cmake --build -j`:

```bash
./build_openroad.sh --local --no_init -t 100
```

- 默认是 `nproc --all`(=128),`-t 100` 即把上限降到 100。
- 后台跑、不卡前台,加 `-n`(nice):`./build_openroad.sh --local --no_init -t 100 -n`
- 脚本开头会打印 `[INFO FLW-0028] Compiling with N threads.` 用于核对。

## 2. 本地构建提速手段(按收益排序)

**① ccache —— 重复构建最大杀器**(首次不变,之后改代码/pull/切分支重编快数倍)

```bash
sudo apt install -y ccache        # 或 yum/dnf install ccache
ccache -M 100G                    # 配置大缓存

./build_openroad.sh --local --no_init \
  --openroad-args="-DCMAKE_CXX_COMPILER_LAUNCHER=ccache -DCMAKE_C_COMPILER_LAUNCHER=ccache"
```

**② 增量构建,别加 `--clean` / `--clean-force`**
脚本默认 `cmake --build tools/OpenROAD/build` 只重编改动文件;`--clean` 会清空从头来。

**③ 不需要 OpenROAD 就跳过它**(占整个编译时间 ~90%)
```bash
./build_openroad.sh --local --no_init -s   # -s / --skip_openroad
```

**④ 用 Ninja 替代 Make**(增量调度更快,需先清一次 build 目录)
```bash
rm -rf tools/OpenROAD/build
./build_openroad.sh --local --no_init \
  --openroad-args="-G Ninja -DCMAKE_CXX_COMPILER_LAUNCHER=ccache"
```

**⑤ 最快:根本不编译**
只想跑 flow、不改 C++ 时,用预编译二进制(`docs/user/BuildWithPrebuilt.md`,
Precision Innovations 的 `.deb`)或 Docker 镜像,省掉全部编译时间。

## 3. `--no_init` 的含义

`--no_init` = 跳过脚本里的 `git submodule update --init --recursive`
(它把 `OPENROAD_FLOW_NO_GIT_INIT` 设为 1,从而跳过 `__common_setup()` 中的子模块更新)。

- 不加时:每次构建前都会联网拉取并把子模块 checkout 到 ORFS 记录的固定 commit。
- 加上后:不再联网、不动子模块,直接用磁盘现有源码编译。
- 主要用途:重复构建省时省网络、离线编译、**本地改 OpenROAD C++ 源码时防止脚本
  把分支切回固定 commit 而丢失改动**。

⚠️ **关键提醒**:当前 `tools/OpenROAD/` 为空(子模块未 checkout)。
→ **第一次构建绝不能加 `--no_init`**,否则无源码可编、直接失败。先初始化一次:

```bash
# 二选一:
git submodule update --init --recursive          # 手动初始化
# 或首次构建不带 --no_init:
./build_openroad.sh --local -t 100               # 会顺带拉下子模块
```

子模块就位后,后续重复编译再加 `--no_init`。

## 推荐命令(子模块已初始化后的日常重复编译)

```bash
./build_openroad.sh --local --no_init -t 100 -n \
  --openroad-args="-DCMAKE_CXX_COMPILER_LAUNCHER=ccache -DCMAKE_C_COMPILER_LAUNCHER=ccache"
```
