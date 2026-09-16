# 启用 OpenROAD GUI + 跑设计 + 开 GUI

> 接续 `conda编译修复记录.md`(openroad + yosys 已编译成功)。本文记录:
> 1. 把 openroad 重新编译成**带 GUI** 的版本(Qt5)
> 2. 跑一个设计(gcd 全流程)
> 3. 用 `make gui_final` 打开最终版图 GUI

## 最终结论(先看这里)

- ✅ **gcd 全流程跑通**:综合→布局→CTS→布线→提取,产物 `6_final.odb/.def/.spef/.v` 都在 `flow/results/nangate45/gcd/base/`。
- ✅ **GUI 已启用并打开**:`openroad` 现为 `+GUI` 版本,`make gui_final` 已用软件渲染稳定打开(见用户 DISPLAY)。
- ⚠️ 唯一跳过:最后的 GDS/DRC/LVS 步骤需要 KLayout(本机未装);不影响布局结果与 GUI。

---

## 一、启用 GUI(给 openroad 加上 Qt5)

默认编译出的 openroad 是**无 GUI** 版(`Features: -GUI`)。开 GUI 需要 Qt5。三个坑:

### 坑1:装 Qt5
```bash
conda activate openroad
conda install -n openroad -c conda-forge -y qt-main   # Qt5 5.15,自带 Qt5Charts
```

### 坑2:清掉 CMakeCache,否则不会重新检测 Qt5
`find_package(Qt5)` 的结果被**缓存**。装了 qt-main 后,不清缓存重配,GUI 仍是 OFF。
```bash
rm -f tools/OpenROAD/build/CMakeCache.txt   # 强制重新检测;已编译的 .o 保留,增量很快
cmake -B tools/OpenROAD/build -S tools/OpenROAD \
  -DUSE_SYSTEM_BOOST=ON -DCUDD_DIR=$CONDA_PREFIX -DBUILD_PYTHON=OFF \
  -DCMAKE_INSTALL_PREFIX=$PWD/tools/install/OpenROAD -DCMAKE_POLICY_VERSION_MINIMUM=3.5
# 确认日志里出现 "-- GUI is enabled" / "GUI Support : ON"
```

### 坑3:GUI 模块要 OpenGL 头 `GL/gl.h`
gui 编译报 `QtGui/qopengl.h:141: fatal error: GL/gl.h: No such file or directory`。
- 系统**有** `/usr/include/GL/gl.h`(mesa-libGL-devel),但 conda 编译器用 sysroot 看不到系统 `/usr/include`。
- 修法:把系统 GL 头软链进 conda include(只引入 GL,不污染其它):
```bash
ln -sf /usr/include/GL $CONDA_PREFIX/include/GL
```

### 重新编译 openroad 目标(带 GUI)
注:`build_openroad.sh` 走 `--target install`,会顺带构建一个无关工具 `thirdparty/yaml-cpp/util/yaml-cpp-sandbox`(链接失败:YAML:: 未定义),卡住整个构建。但 **openroad 本体不依赖 sandbox**,直接只构建 openroad 目标即可绕过:
```bash
source /users/G01/kshi/kshi/tool/anaconda3/etc/profile.d/conda.sh && conda activate openroad
export PKG_CONFIG_PATH="$CONDA_PREFIX/lib/pkgconfig:${PKG_CONFIG_PATH:-}"
cmake --build tools/OpenROAD/build --target openroad -j 100   # 含 gui 模块 + Qt5 重链
cp -f tools/OpenROAD/build/bin/openroad tools/install/OpenROAD/bin/openroad   # 装到位
```
验证:
```bash
tools/install/OpenROAD/bin/openroad -gui -no_init -exit 2>&1 | grep -i gui
# 不再出现 "[ERROR] This code was compiled with the GUI disabled" 即成功
```

---

## 二、跑一个设计(gcd,nangate45)

```bash
conda activate openroad
cd flow
source ../env.sh                 # 把 tools/install/*/bin 加进 PATH
export LD_LIBRARY_PATH="$CONDA_PREFIX/lib:${LD_LIBRARY_PATH:-}"   # 运行时 conda 库
make DESIGN_CONFIG=designs/nangate45/gcd/config.mk
```
- gcd 是小设计,全流程约 2–4 分钟。
- 结果在 `flow/results/nangate45/gcd/base/`:`6_final.odb`(最终数据库)、`6_final.def`、`6_final.spef`(寄生)、`6_final.v`。
- **KLayout 未装** → 最后的 `check-klayout` 报错 `KLayout not found`,GDS/DRC/LVS 步骤跳过;但布线/提取已完成,布局结果完整,GUI 可用。
  - 要补 GDS:装 KLayout(RPM 或预编译二进制),或设 `KLAYOUT_CMD`。

---

## 三、打开 GUI(`make gui_final`)

ORFS 的开 GUI 机制在 `flow/Makefile` 里用宏生成(所以 `grep gui_final:` 搜不到):
- `$(eval $(call OPEN_GUI_SHORTCUT,final,6_final.odb))` 生成 `gui_final` 目标。
- `make gui_final` 实际执行:
  ```
  ODB_FILE=./results/nangate45/gcd/base/6_final.odb \
    <openroad> -gui -threads N  scripts/open.tcl
  ```
  即用项目自带的 `scripts/open.tcl` 加载 `6_final.odb` 并开 GUI。

### 命令(后台启动,窗口开在你的 `$DISPLAY`)
```bash
conda activate openroad
cd flow && source ../env.sh
export DISPLAY=:2                        # 你的 X 显示
export LD_LIBRARY_PATH="$CONDA_PREFIX/lib:${LD_LIBRARY_PATH:-}"
# 本机是 NVIDIA GPU,conda/Qt 环境下 nvidia-drm/GLX 加载失败 → 强制软件渲染,否则 GUI 可能崩
export LIBGL_ALWAYS_SOFTWARE=1
export GALLIUM_DRIVER=llvmpipe
nohup make DESIGN_CONFIG=designs/nangate45/gcd/config.mk gui_final > /tmp/gui_final.log 2>&1 &
```
- 日志里看到 `Features included (+) or not (-): +GPU +GUI -Python`、`read_db ... 6_final.odb`、`gui::select_chart`、`gui::update_timing_report` 即成功,设计已加载并在渲染。
- 各阶段都可开 GUI:`make gui_synth` / `gui_floorplan` / `gui_place` / `gui_cts` / `gui_route` / `gui_final`(分别加载对应阶段的 odb)。
- 非 GUI 批处理打开:`make open_final`(同 open.tcl 但不开图形窗口)。

### 关 GUI
```bash
pkill -f "openroad -gui"
```

---

## 四、踩坑小结(GUI 相关)

| 现象 | 原因 | 修法 |
|------|------|------|
| `compiled with the GUI disabled` | 没装 Qt5 / 没重新检测 | 装 `qt-main`,删 `CMakeCache.txt` 重配 |
| `GUI is not enabled`(重配后仍 OFF) | find_package(Qt5) 读旧缓存 | `rm CMakeCache.txt` |
| gui 编译 `GL/gl.h: No such file` | conda sysroot 无 OpenGL 头 | `ln -sf /usr/include/GL $CONDA_PREFIX/include/GL` |
| `make gui_final` 走 build_openroad.sh 卡在 `yaml-cpp-sandbox` 链接 | 无关工具链接失败 | 直接 `cmake --build --target openroad` 绕过 |
| GUI 进程起来又退出 / GLX 失败 | NVIDIA `nvidia-drm` 在 conda/Qt 下加载失败 | `export LIBGL_ALWAYS_SOFTWARE=1`(llvmpipe 软件渲染) |
| 找不到 `gui_final` 目标 | 它是 Makefile 宏 `$(eval $(call OPEN_GUI_SHORTCUT,...))` 动态生成的 | 直接 `make gui_final` 即可 |

---

## 五、环境变量速查(跑设计 / 开 GUI 前都要设)

```bash
source /users/G01/kshi/kshi/tool/anaconda3/etc/profile.d/conda.sh && conda activate openroad
cd /data/trd/kshi/code/OpenROAD-flow-scripts
source env.sh
export LD_LIBRARY_PATH="$CONDA_PREFIX/lib:${LD_LIBRARY_PATH:-}"
export PKG_CONFIG_PATH="$CONDA_PREFIX/lib/pkgconfig:${PKG_CONFIG_PATH:-}"   # 编译用
export DISPLAY=:2                                                            # 开 GUI 用
export LIBGL_ALWAYS_SOFTWARE=1                                               # 开 GUI 用(NVIDIA 机)
```
