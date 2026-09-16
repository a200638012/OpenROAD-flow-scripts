# 任务:为 OpenROAD GUI 增加 AI 能力(AI for OpenROAD)

> **状态:** 设计 / 待开发
> **目标仓库:** `tools/OpenROAD/`(引擎 + GUI 子模块)
> **相关笔记:** `00_chat/Qt开发与聊天窗口.md`、`00_chat/GUI启用与跑设计.md`
> **前置条件:** openroad 已编译为 `+GUI`(Qt5)版本 ✅

---

## 0. 任务概述

在 OpenROAD 的图形界面(GUI)中集成 AI 能力,让用户能用自然语言与 EDA 工具交互:
- 一边查看版图/时序报告,一边在**聊天窗口**里提问、下指令;
- 由 **Agent** 调用大模型(LLM),并允许 LLM **执行 OpenROAD 命令 / 查询设计状态**,
  把"看图 + 改参数 + 跑流程"变成对话式体验。

最终效果示例:

```
用户:当前最差路径的 slack 是多少?把它高亮在版图上。
Agent:[调用 report_worst_slack] 当前最差路径 slack = -0.42ns,所属寄存器 reg_xxx。
      [调用 highlight] 已在版图上用红色高亮该路径。(见画布)
```

---

## 1. 背景与现状(已完成的前置条件)

| 项目 | 状态 | 说明 |
|------|------|------|
| openroad 编译为 `+GUI` | ✅ | Qt5 5.15(`qt-main`),见 `GUI启用与跑设计.md` |
| GUI 源码结构已摸清 | ✅ | `tools/OpenROAD/src/gui/`,每面板 = 一个 `QDockWidget` |
| 增量构建流程已跑通 | ✅ | `cmake --build tools/OpenROAD/build --target openroad` |
| 现有 LLM / Agent 代码 | ❌ 无 | 本任务为**全新功能(greenfield)** |

### 可直接复用的现成"桥"
- **`ScriptWidget::setupTcl(Tcl_Interp*, …)`**(`src/gui/src/scriptWidget.h`):
  GUI 里已持有 Tcl 解释器 —— Agent 执行 OpenROAD 命令的入口就靠它。
- **`gui::Painter` / `Renderer` / `SelectionSet`**(`include/gui/gui.h`):
  Agent 想在版图上高亮/标注对象时,走这套绘制接口。
- **Widget 注册模式**(`src/gui/src/mainWindow.cpp:104,120-129`):
  `addDockWidget(...)` + `tabifyDockWidget(...)`。

---

## 2. 总体架构

```
┌──────────────────────────── OpenROAD GUI (Qt5) ────────────────────────────┐
│                                                                            │
│   ┌─── chatWidget(新)─────────────┐      ┌── Layout / 报表面板 ──┐          │
│   │  消息流(QListView)            │      │  (画布、时序、DRC…)    │          │
│   │  输入框(QLineEdit)+发送       │◀────▶│  高亮 / 选中反馈        │          │
│   │  设置(模型/Key/只读模式开关) │      └────────────────────────┘          │
│   └───────────────┬───────────────┘                                          │
│                   │ 用户文本                                                 │
│            ┌──────▼───────┐                                                  │
│            │  Agent 循环   │  ReAct / function-calling:规划→工具→观察→回答    │
│            └──────┬───────┘                                                  │
│        工具调用     │                                                          │
│   ┌────────────────┼───────────────────────────────────┐                     │
│   │  run_tcl       get_design_summary   highlight      │  ← 内置 tools        │
│   │   (走 setupTcl 解释器)  (查询)        (走 Painter)   │                     │
│   └────────────────┬───────────────────────────────────┘                     │
│                    │ 统一 OpenAI 兼容 HTTP 客户端(QNetworkAccessManager)     │
└────────────────────┼────────────────────────────────────────────────────────┘
                     │
        ┌────────────┴───────────────────────────┐
        ▼                                         ▼
 ┌─────────────────┐                    ┌──────────────────────┐
 │ 云端 LLM:glm   │                    │ 本地 LLM:llama.cpp  │
 │ (智谱,OpenAI   │                    │ (llama-server,      │
 │  兼容 API)      │                    │  OpenAI 兼容 API)    │
 └─────────────────┘                    └──────────────────────┘
```

**核心设计点:** glm 与 llama.cpp 的 `llama-server` 都暴露**OpenAI 兼容**的
`/v1/chat/completions`(含 `tools` function-calling)。因此只需**一套** HTTP 客户端 +
`LLMProvider` 抽象,通过切换 `base_url` + `api_key` 即可在云端/本地间切换。

---

## 3. 子任务一:聊天窗口(Chat Window)

> 目标:先做出一个**不接 LLM 也能用**的聊天面板(本地 echo / 历史记录),
> 作为 Agent 的 UI 载体。完全仿照 `clockWidget` 的做法。

### 3.1 UI 设计
- 继承 `QDockWidget`,停靠区建议 `Qt::RightDockWidgetArea`(与 timing/drc 等并排)。
- 控件:
  - `QListView` / `QTextEdit`(只读):消息流(区分 用户 / 助手 / 工具调用 / 错误,可用颜色或角色标签)。
  - `QLineEdit` + `QPushButton`(发送):输入;支持 `Enter` 发送、`Shift+Enter` 换行。
  - 顶部工具条:模型选择(云/本地)、API Key 输入(密码框)、**只读模式**开关、清空历史。
- 状态指示:连接中 / 等待 LLM / 工具执行中 / 就绪。

### 3.2 新建 Widget 类(仿 `src/gui/src/clockWidget.h`)
新增文件:
- `src/gui/src/chatWidget.h` / `src/gui/src/chatWidget.cpp`
- (可选)`src/gui/ui/chatWidget.ui` —— 用 `designer` 画(放 `ui/` 目录,`AUTOUIC` 自动生成 `ui_chatWidget.h`)。

要点:
- 类声明里务必加 **`Q_OBJECT`**(否则 moc / 信号槽不生效)。
- 包进 `namespace gui { ... }`(参考 `clockWidget.h`)。
- 需要拿到 `utl::Logger*`(日志)与 Tcl 解释器引用(供后续 Agent 用),参考
  `mainWindow.cpp` 中 `clock_viewer_->setLogger(...)` / `setSTA(...)` 的注入方式。

### 3.3 消息数据模型
建议基于 `QAbstractListModel` 实现 `ChatMessageModel`:
```
struct ChatMessage { Role role;  // User / Assistant / Tool / System
                     QString text;
                     QVariant tool_call_meta; };  // 关联的工具调用与结果
```
- 持久化:`QSettings` 存历史(参考 `ScriptWidget::readSettings/writeSettings`)。

### 3.4 注册到主窗口(`src/gui/src/mainWindow.cpp`)
```cpp
#include "chatWidget.h"
// 成员:
ChatWidget* chat_widget_;
// 构造初始化列表(仿第 104-105 行):
chat_widget_(new ChatWidget(this)),
// 停靠(仿第 120-129 行):
addDockWidget(Qt::RightDockWidgetArea, chat_widget_);
tabifyDockWidget(timing_widget_, chat_widget_);   // 与现有右侧面板成标签组
// View/Windows 菜单加切换项(仿第 887-888 行):
windows_menu_->addAction(chat_widget_->toggleViewAction());
```
并在 `setLogger` / chipLoaded 等处把 Logger、Tcl interp 注入 chatWidget。

### 3.5 CMake 集成(`src/gui/CMakeLists.txt`)
- 第 61-63 行 `AUTOMOC/AUTORCC/AUTOUIC` 已开启 → `.h`/`.ui`/`.qrc` 自动处理。
- 在第 77 行 `target_sources(gui PRIVATE …)` 列表里加 `src/chatWidget.cpp`(`.h` 也建议列上,便于 IDE)。

### 3.6 构建与验证
```bash
source /users/G01/kshi/kshi/tool/anaconda3/etc/profile.d/conda.sh && conda activate openroad
cd /data/trd/kshi/code/OpenROAD-flow-scripts
export PKG_CONFIG_PATH="$CONDA_PREFIX/lib/pkgconfig:${PKG_CONFIG_PATH:-}"
cmake --build tools/OpenROAD/build --target openroad -j 100
cp -f tools/OpenROAD/build/bin/openroad tools/install/OpenROAD/bin/openroad

# 验证(软件渲染):
export DISPLAY=:2 LIBGL_ALWAYS_SOFTWARE=1 LD_LIBRARY_PATH="$CONDA_PREFIX/lib:${LD_LIBRARY_PATH:-}"
cd flow && source ../env.sh
make DESIGN_CONFIG=designs/nangate45/gcd/config.mk gui_final   # 应能看到新 Chat 面板
```
> ⚠️ **不要**走 `build_openroad.sh`(会卡在无关的 `yaml-cpp-sandbox` 链接失败);直接 `--target openroad` 绕过。详见 `Qt开发与聊天窗口.md` 第四节"常见坑"。

---

## 4. 子任务二:Agent

> 目标:把聊天窗口接入 LLM,并让 LLM 通过**工具调用(tool-calling)**操作 OpenROAD。

### 4.1 Agent 循环(ReAct / function-calling)
```
1. 用户输入 → 组装 messages(含系统提示 + 设计上下文摘要 + 历史)
2. POST {base_url}/v1/chat/completions   (带 tools 定义)
3. 收到响应:
   ├─ 含 tool_calls  → 在 GUI 执行对应工具 → 把结果作为 tool 消息回灌 → 回到第 2 步
   └─ 仅文本         → 作为最终回答渲染到聊天窗口,结束
```
- **流式输出**:`stream: true`,边收边显示,体验更好(可用 `QNetworkReply::readyRead`)。
- **可中断**:长循环提供"停止"按钮(`QNetworkReply::abort()`)。

### 4.2 设计上下文与内置工具(tools)
系统提示(system prompt)应包含:OpenROAD 是什么、当前设计的简要信息(设计名、节点、
是否已综合/布线)、可用工具清单。建议先实现这几个工具:

| 工具 | 作用 | 实现 |
|------|------|------|
| `run_tcl(command)` | 执行任意 Tcl(综合/布局/报告…) | 走 `ScriptWidget` 的 Tcl 解释器(`setupTcl`) |
| `get_design_summary()` | 返回当前设计概览(实例数/线网数/阶段) | 查 `odb::dbDatabase` |
| `get_selection()` | 返回当前 GUI 选中的对象 | 走 `SelectionSet` |
| `highlight(name)` / `zoom_to(name)` | 在版图上高亮/聚焦对象 | 走 `Renderer` / `Painter` + `gui` 选中 API |
| `report_timing()` / `report_worst_slack()` | 时序报告 | 即 Tcl,可作为 `run_tcl` 的封装,或单独工具便于 LLM 调用 |

> 这些工具本质是把 GUI 已有能力"翻译"成 LLM 可调用的 function schema(JSON Schema),
> 返回结构化结果供 LLM 阅读。

### 4.3 连接云端 LLM —— glm(智谱)
- API:`https://open.bigmodel.cn/api/paas/v4/chat/completions`(OpenAI 兼容)。
- 鉴权:`Authorization: Bearer <API_KEY>`(从设置面板读取,**不要硬编码**)。
- 支持 `tools`(function-calling),适合做 Agent。
- 注意:云上**不要**把敏感设计数据(网表/关键 IP)整包上传 —— 默认只传摘要与命令结果。

### 4.4 连接本地 LLM —— llama.cpp
- 用 `llama-server`(llama.cpp 自带)加载本地 GGUF 模型,暴露:
  `http://localhost:8080/v1/chat/completions`(OpenAI 兼容)。
- 适合内网/保密设计;支持 tool-calling 的模型(如部分 Qwen / GLM GGUF)即可驱动 Agent。
- 启动示例:`llama-server -m <model.gguf> --port 8080 -c 4096`(用户侧自行准备模型文件)。

### 4.5 统一 `LLMProvider` 抽象(C++)
```cpp
struct LLMRequest { QString base_url; QString api_key; QString model;
                    QJsonArray messages; QJsonArray tools; bool stream; };
class LLMProvider : public QObject {        // 走 QNetworkAccessManager,异步
  Q_OBJECT
public slots:
  void send(const LLMRequest& req);
signals:
  void textDelta(const QString& delta);     // 流式增量
  void toolCalls(const QJsonArray& calls);  // 工具调用
  void finished(); void error(const QString& msg);
};
```
- 因为 glm 和 llama.cpp 都是 OpenAI 兼容,**一套实现即可**,只换 `base_url`/`api_key`/`model`。
- JSON 用 Qt 自带 `QJsonDocument`,**不引入新依赖**。
- 配置持久化到 `QSettings`(model、base_url、是否流式、只读模式等)。

### 4.6 安全:命令确认门(重要)
LLM 生成 `run_tcl` 这类**可能改写设计**的命令时,默认**不自动执行**:
- **只读模式(默认 ON)**:只允许白名单命令(`report_*`、`get_*`、`gui::select*` 等)自动跑;
  其余(综合、place、route、写文件…)弹出**确认对话框**,显示将执行的 Tcl,用户点"允许"才执行。
- 提供开关让高级用户关掉确认门(自担风险)。

---

## 5. 关键技术决策

| 议题 | 选择 | 理由 |
|------|------|------|
| 网络/HTTP | `QNetworkAccessManager` | Qt5 自带,异步,无新依赖 |
| JSON | `QJsonDocument` | Qt5 自带,无新依赖 |
| LLM 协议 | OpenAI 兼容 `/v1/chat/completions` | glm 与 llama.cpp 都兼容,一套客户端通吃 |
| 模型切换 | 设置面板存 `base_url`+`api_key`+`model` | 云/本地无缝切换 |
| 聊天 UI 载体 | 新建 `chatWidget`(仿 `clockWidget`) | 复用既有 Widget 注册模式 |
| 工具执行入口 | 复用 GUI 已有 Tcl 解释器(`setupTcl`) | 不重复造桥 |
| 版图交互 | `gui::Painter/Renderer/SelectionSet` | 官方绘制/选中 API |
| 命令安全 | 只读白名单 + 确认门 | 防止 LLM 误改/破坏设计 |

---

## 6. 里程碑(分阶段交付)

| 阶段 | 交付物 | 验证 |
|------|--------|------|
| **M1 聊天面板骨架** | `chatWidget` 能显示、能本地 echo、有历史持久化 | GUI 里能看到面板、能输入/显示消息 |
| **M2 LLM 接入(纯对话)** | 接通 glm 或 llama.cpp,支持流式输出、多轮上下文 | 能和模型正常对话(尚不能操作设计) |
| **M3 Agent 工具调用** | 实现 `run_tcl` 等工具 + Agent 循环 + 只读白名单 | 用户能"用自然语言查时序/选中对象" |
| **M4 版图联动** | `highlight` / `zoom_to` / 选中反馈打通 | Agent 能在画布上高亮对象并回显 |
| **M5 安全与打磨** | 写命令确认门、设置面板、错误处理、可中断 | 破坏性命令必须经用户确认 |

---

## 7. 验收标准

- [ ] `make gui_final` 能在右侧看到 Chat 面板,可拖动/停靠/隐藏(View 菜单可切换)。
- [ ] 能切换云端(glm)与本地(llama.cpp)两种后端,均在设置里填 URL/Key/Model。
- [ ] 多轮对话上下文正确;支持流式输出与"停止"。
- [ ] Agent 至少能通过 `run_tcl` 完成一次"自然语言 → 查询 → 回答"闭环(如问最差路径 slack)。
- [ ] 默认只读模式:改写类命令必须弹确认框,显示完整 Tcl 后由用户放行。
- [ ] 不引入新的第三方库(仅用 Qt5 自带能力)。
- [ ] 增量构建通过:`cmake --build tools/OpenROAD/build --target openroad`。

---

## 8. 风险与待定问题

- **Tcl 执行线程**:GUI 主线程跑 Tcl 会阻塞界面;长命令需考虑放到工作线程或加"执行中"遮罩。
- **模型能力差异**:不是所有本地 GGUF 模型都支持 tool-calling;需在文档里标注推荐模型。
- **上下文长度**:设计对象可能成千上万,不能整包喂给 LLM;只传摘要 + 按需 `get_*` 查询。
- **保密性**:云端模式下默认不传网表/IP;是否提供"完全本地"强开关,需确认产品诉求。
- **OpenROAD 升级**:本仓库是 submodule,改的是 `tools/OpenROAD`;需记录改动以便 upstream 同步。
- **待定**:
  1. 是否需要 RAG(检索 OpenROAD 文档/Tcl 手册)增强回答准确性?
  2. 是否要持久化对话历史到磁盘(而非仅 `QSettings`)?
  3. Agent 是否支持"批处理脚本生成"(让 LLM 生成一段 Tcl 让用户审阅后整体执行)?

---

## 9. 参考资料

- 笔记:`00_chat/Qt开发与聊天窗口.md`(加 Widget 全步骤、常见坑、环境变量)
- 笔记:`00_chat/GUI启用与跑设计.md`(启用 GUI、跑 gcd、开 GUI)
- Widget 范式:`tools/OpenROAD/src/gui/src/clockWidget.h`
- 主窗口注册:`tools/OpenROAD/src/gui/src/mainWindow.cpp:104,120-129,887-888`
- Tcl 桥:`tools/OpenROAD/src/gui/src/scriptWidget.h`(`setupTcl`)
- 绘制/选中 API:`tools/OpenROAD/src/gui/include/gui/gui.h`(`Painter`/`Renderer`/`SelectionSet`)
- 构建:CMake `tools/OpenROAD/src/gui/CMakeLists.txt:61-77`
- glm(智谱)API:https://open.bigmodel.cn/dev/api
- llama.cpp server(OpenAI 兼容):https://github.com/ggml-org/llama.cpp
```
