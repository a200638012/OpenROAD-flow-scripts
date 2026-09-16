# Qt 开发环境 & 在 OpenROAD GUI 加聊天窗口

> 接续 `GUI启用与跑设计.md`(openroad 已编译为 `+GUI`)。本文记录:
> 1. conda 环境里的 Qt 开发工具现状
> 2. 在 OpenROAD GUI 里新增一个聊天窗口(Widget)的步骤

## 一、Qt 开发工具:已随 `qt-main` 装好,无需再装

`qt-main`(Qt5 5.15)已经带齐全套开发工具,都在 `$CONDA_PREFIX/bin/`:

| 工具 | 用途 |
|------|------|
| **designer** | 可视化设计 UI(拖控件生成 `.ui`)—— 画聊天窗口界面用这个 |
| **moc** | Qt 元对象代码生成(信号槽 Q_OBJECT 必需) |
| **uic** | 把 `.ui` 编译成 C++ 头 |
| **rcc** | 资源文件 `.qrc` 编译(图标等) |
| **linguist / lupdate / lrelease** | 国际化 |
| **qmake / assistant** | Qt 构建 / Qt 帮助文档 |

验证:
```bash
conda activate openroad
for t in designer moc uic rcc linguist qmake assistant; do command -v $t; done
```

OpenROAD GUI 的构建已开启 `AUTOMOC/AUTOUIC/AUTORCC`(`src/gui/CMakeLists.txt:61-64`),
所以**新增的 Qt 源码 / `.ui` / `.qrc` 会自动**走 moc/uic/rcc,不用手动调用。

### Qt Creator(IDE)conda-forge 没有
`conda search qtcreator` → `No match found`。要 IDE:
- 系统包:`sudo dnf install qt-creator`(Rocky 9)
- 或 Qt 官方安装器
- 否则:`designer` 画 UI + VS Code/编辑器写 C++

### 启动 designer / GUI 时(本机 NVIDIA GPU)
conda/Qt 环境下 `nvidia-drm`/GLX 加载失败 → 强制软件渲染,否则可能崩:
```bash
export DISPLAY=:2
export LIBGL_ALWAYS_SOFTWARE=1
export GALLIUM_DRIVER=llvmpipe
designer &     # 或 openroad -gui
```

---

## 二、OpenROAD GUI 源码结构(加 Widget 前先了解)

位置:`tools/OpenROAD/src/gui/`

- **每个面板 = 一个 Widget 类**,各自 `src/<name>Widget.cpp/.h`。范例:
  `clockWidget`、`drcWidget`、`browserWidget`、`chartsWidget`、`timingWidget`、`helpWidget`…
- **`.ui` 文件**放 `src/gui/ui/`,`AUTOUIC`(搜索路径 `ui`)自动处理。
- **注册到主窗口**:`src/mainWindow.cpp`
  - 成员:如 `clock_viewer_(new ClockWidget(this))`(第 ~104 行)
  - 停靠:`addDockWidget(Qt::LeftDockWidgetArea, controls_)`(第 ~120 行起)
- **加入 gui 库**:`src/gui/CMakeLists.txt` 的 `target_sources(gui PRIVATE …)`。

---

## 三、加聊天窗口的步骤(仿 clockWidget)

### 1. 新建 Widget 类
`src/gui/src/chatWidget.h` + `src/gui/src/chatWidget.cpp`,继承 `QDockWidget`。
典型控件:`QListView`(消息流)+ `QLineEdit`(输入)+ `QPushButton`(发送)。
关键点:
- 头文件类声明里加 `Q_OBJECT`(否则 moc/信号槽不工作)。
- 把命名空间包进 OpenROAD gui 的命名空间(参考 `clockWidget.h`)。

### 2. (可选)用 designer 画 UI
```bash
designer   # 新建 Widget,存为 src/gui/ui/chatWidget.ui
```
存到 `ui/` 目录后,`AUTOUIC` 会自动生成 `ui_chatWidget.h`,`#include "ui_chatWidget.h"` 即可用。

### 3. 把源文件加入 gui 库
编辑 `src/gui/CMakeLists.txt`,在 `target_sources(gui PRIVATE …)` 列表里加:
```cmake
  src/chatWidget.cpp
```
(`.ui`、`.h` 不用手动加,AUTOUIC/AUTOMOC 自动处理;`.h` 通常也列上便于 IDE。)

### 4. 注册到主窗口
编辑 `src/gui/src/mainWindow.cpp`:
- 头部 `#include "chatWidget.h"`
- 类成员加:`ChatWidget* chat_widget_;`
- 构造函数:`chat_widget_(new ChatWidget(this))`
- 停靠:`addDockWidget(Qt::RightDockWidgetArea, chat_widget_);`(区域自选)
- (可选)在「View/Windows」菜单加一个切换项(参考其它 widget 的 menuAction)。

### 5. 增量重编 openroad(只动 gui,几分钟)
```bash
source /users/G01/kshi/kshi/tool/anaconda3/etc/profile.d/conda.sh && conda activate openroad
cd /data/trd/kshi/code/OpenROAD-flow-scripts
export PKG_CONFIG_PATH="$CONDA_PREFIX/lib/pkgconfig:${PKG_CONFIG_PATH:-}"
cmake --build tools/OpenROAD/build --target openroad -j 100
cp -f tools/OpenROAD/build/bin/openroad tools/install/OpenROAD/bin/openroad
```
> ⚠️ 不要走 `build_openroad.sh`(它会卡在无关的 `yaml-cpp-sandbox` 链接失败);直接构建 `--target openroad` 绕过。

### 6. 验证
```bash
export DISPLAY=:2 LD_LIBRARY_PATH="$CONDA_PREFIX/lib:${LD_LIBRARY_PATH:-}" LIBGL_ALWAYS_SOFTWARE=1
cd flow && source ../env.sh
make DESIGN_CONFIG=designs/nangate45/gcd/config.mk gui_final   # 应看到新聊天窗口面板
```

---

## 四、常见坑

| 现象 | 原因 / 修法 |
|------|------|
| 新 Widget 信号槽没反应 / moc 报错 | 类里漏了 `Q_OBJECT` 宏;AUTOMOC 会自动跑 moc,只要 `Q_OBJECT` 在 |
| `ui_chatWidget.h: No such file` | `.ui` 没放在 `src/gui/ui/`,或 `AUTOUIC` 没生效(确认 CMakeLists 里 `set(CMAKE_AUTOUIC ON)` + 搜索路径 `ui`) |
| 改了 CMakeLists 不生效 | 删 `tools/OpenROAD/build/CMakeCache.txt` 重配(或 cmake 会自动检测 CMakeLists 变更重配) |
| GUI 编译报 `GL/gl.h` 缺失 | `ln -sf /usr/include/GL $CONDA_PREFIX/include/GL`(见 `GUI启用与跑设计.md`) |
| designer / GUI 启动闪退 | NVIDIA GLX 失败 → `export LIBGL_ALWAYS_SOFTWARE=1` |
| 新窗口在 GUI 里找不到 | 没在 `mainWindow.cpp` 里 `addDockWidget`,或被默认隐藏(菜单 View 里勾选) |

---

## 五、环境变量速查(开发 + 运行 GUI)

```bash
source /users/G01/kshi/kshi/tool/anaconda3/etc/profile.d/conda.sh && conda activate openroad
cd /data/trd/kshi/code/OpenROAD-flow-scripts
export LD_LIBRARY_PATH="$CONDA_PREFIX/lib:${LD_LIBRARY_PATH:-}"
export PKG_CONFIG_PATH="$CONDA_PREFIX/lib/pkgconfig:${PKG_CONFIG_PATH:-}"   # 编译用
export DISPLAY=:2                                                            # designer / GUI 用
export LIBGL_ALWAYS_SOFTWARE=1                                               # designer / GUI 用(NVIDIA 机)
```
