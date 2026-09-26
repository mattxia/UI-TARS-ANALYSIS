# UI-TARS 项目分层技术架构分析

> 分析日期：2026-09-26 · 分析范围：仓库全量代码与文档

## 一、项目定位

本仓库是字节跳动 **UI-TARS** 官方仓库，本体形态为「文档 + 轻量 Python 后处理 SDK」。SDK 即 PyPI 包 `ui-tars` 0.1.4（见 `codes/pyproject.toml`，**零运行时依赖**）。GUI Agent 大模型本体不在仓库内，通过 OpenAI 兼容接口远程调用；仓库负责 Prompt 约定与模型输出的解析、坐标归一化和代码生成落地。

## 二、分层架构总览

```
┌─────────────────────────────────────────────────────────┐
│ L0 文档与指南层（仓库根目录，不参与运行时）                  │
│    README.md / README_deploy.md / README_coordinates.md  │
│    README_v1.md / UI_TARS_paper.pdf                      │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│ 上层应用（调用方，外部）：UI-TARS-desktop / OSWorld Agent   │
│         编排 Agent 循环，维护任务与多轮历史                 │
└───────────────┬─────────────────────────────────────────┘
                │ ① 读取模板，组装 messages（指令+base64截图+历史）
┌───────────────▼─────────────────────────────────────────┐
│ L1 Prompt 模板层：codes/ui_tars/prompt.py                │
│    COMPUTER_USE_DOUBAO / MOBILE_USE_DOUBAO              │
│    / GROUNDING_DOUBAO                                   │
└───────────────┬─────────────────────────────────────────┘
                │ ② OpenAI 兼容推理请求（多模态 messages）
┌───────────────▼─────────────────────────────────────────┐
│ L2 模型推理层（外部）：UI-TARS-1.5-7B @ HF TGI / vLLM     │
│         输入截图+指令，输出 "Thought: … Action: …" 文本    │
└───────────────┬─────────────────────────────────────────┘
                │ ③ 返回原始 Thought / Action 文本
┌───────────────▼─────────────────────────────────────────┐
│ L3 核心解析层（本仓库 SDK）：codes/ui_tars/action_parser.py│
│    对外: parse_action_to_structure_output()              │
│          parsing_response_to_pyautogui_code()            │
│          add_box_token()                                 │
│    内部: smart_resize / parse_action /                   │
│          convert_point_to_coordinates /                  │
│          escape_single_quotes                            │
└───────────────┬─────────────────────────────────────────┘
                │ ④⑤ 解析·坐标归一化 → 生成 pyautogui 代码
┌───────────────▼─────────────────────────────────────────┐
│ L4 动作执行层（外部）：pyautogui 脚本 @ OSWorld/桌面/模拟器 │
│         执行操作 → 新截图回流，进入下一轮（回到①）          │
└─────────────────────────────────────────────────────────┘
┌────────────────────────┐  ┌────────────────────────────┐
│ L5 测试与验证层          │  │ L6 工程支撑层               │
│ codes/tests/            │  │ pyproject.toml / makefile  │
│ action_parser_test.py   │  │ uv.lock / CI test.yml      │
│ inference_test.py       │  │ data/ 样例数据              │
└────────────────────────┘  └────────────────────────────┘
```

## 三、各层组件与职责

### L0 文档与指南层（仓库根目录，不参与运行时）

| 组件 | 职责 |
|---|---|
| `README.md` | 项目总览、基准成绩、安装与两步快速开始 |
| `README_deploy.md` | HuggingFace Inference Endpoints（TGI 3.2.1）部署参数 + `openai` SDK 推理示例（含历史消息 `add_box_token` 预处理） |
| `README_coordinates.md` / `codes/tests/inference_test.py` | 坐标换算原理教程与可视化验证 |
| `README_v1.md` | v1 模型的 vLLM 部署与 Prompt 范式 |
| `UI_TARS_paper.pdf` 等论文 | 算法与基准说明 |

### L1 Prompt 模板层 — `codes/ui_tars/prompt.py`

| 组件 | 职责 |
|---|---|
| `COMPUTER_USE_DOUBAO` | 桌面端模板：定义 `Thought/Action` 输出格式 + 桌面动作空间（click / drag / hotkey / type / scroll / wait / finished） |
| `MOBILE_USE_DOUBAO` | 移动端模板：增加 `long_press / open_app / press_home / press_back` |
| `GROUNDING_DOUBAO` | 纯定位模板：只输出 `Action` 不输出思考，用于 grounding 评测 |

三者均以 `{instruction}`、`{language}` 占位符交给调用方填充，本质是约束模型输出格式使其可被 L3 解析。

### L2 模型推理层（外部）

UI-TARS-1.5-7B 权重部署于 HuggingFace TGI / vLLM，暴露 OpenAI 兼容 `chat.completions` 接口；输入为多模态 messages（文本指令 + base64 截图 + 多轮历史），输出即 `Thought: … Action: …` 文本。仓库不包含此层代码，仅由 `README_deploy.md` 描述调用范式。

### L3 核心解析层 — `codes/ui_tars/action_parser.py`（仓库核心）

**对外 API：**

| 组件 | 职责 |
|---|---|
| `parse_action_to_structure_output()`（L146 起） | 主入口：文本清洗（`<point>`→坐标、`start_point=`→`start_box=`）→ 提取 Thought/Reflection → 按 `)\n\n` 拆分多动作 → 逐个 AST 解析 → 坐标除以 resize 后尺寸归一化到 0~1 → 输出 `[{reflection, thought, action_type, action_inputs, text}]` |
| `parsing_response_to_pyautogui_code()`（L279 起） | 结构化动作 → 可执行 pyautogui 代码字符串：映射 hotkey / press / release / type（默认剪贴板粘贴）/ drag / scroll / click / doubleClick / right-click / hover / finished→DONE，并做按键名转换（arrowleft→left 等） |
| `add_box_token()`（L502 起） | 给多轮历史中 assistant 消息的坐标加 `<\|box_start\|>/<\|box_end\|>` 标记，供下一轮推理 |

**内部支撑：**

| 组件 | 职责 |
|---|---|
| `smart_resize()` / `linear_resize()` + 常量（`IMAGE_FACTOR=28`、`MIN_PIXELS`、`MAX_PIXELS`） | Qwen-VL 图像尺寸约束计算：对齐 28 倍数、控制像素预算 |
| `parse_action()` | `ast.parse` 把 `click(start_box='(x,y)')` 解析为 `{function, args}` |
| `convert_point_to_coordinates()` | `<point>x y</point>` → `(x,y)` |
| `escape_single_quotes()` 等工具 | 单引号转义、`round_by_factor` 等数值工具 |

### L4 动作执行层（外部）

L3 生成的 pyautogui 代码在 OSWorld 基准或用户桌面 / Android 模拟器中执行，执行后截图回流形成 Agent 循环。

### L5 测试与验证层 — `codes/tests/`

| 组件 | 职责 |
|---|---|
| `action_parser_test.py` | unittest，覆盖 L3 三个对外 API（`parse_action` / `parse_action_to_structure_output` / `parsing_response_to_pyautogui_code`） |
| `inference_test.py` | 可执行教学脚本——本地复制了一份 `smart_resize`（仅从 action_parser 导入常量），用 PIL + matplotlib 在原图标注换算后的点击点 |

### L6 工程支撑层

| 组件 | 职责 |
|---|---|
| `codes/pyproject.toml` | hatchling 构建，打包仅含 `ui_tars/**`，零运行时依赖，dev 依赖 matplotlib/pillow |
| `codes/makefile` + `uv.lock` + `.python-version` | `make test` = unittest discover |
| `.github/workflows/test.yml` | CI 在 PR / push 时 `uv sync` → `make test` |
| `data/` | `test_messages.json`（多轮对话 + base64 截图样例）、`training_example.json`、坐标演示图 |

## 四、组件间调用关系

### 运行时主链路（图中①~⑤循环）

1. 上层应用读取 L1 模板，组装 messages（指令 + base64 截图 + 多轮历史）；
2. 调用 L2 模型端点（OpenAI 兼容接口）；
3. 模型返回原始 `Thought/Action` 文本；
4. `parse_action_to_structure_output()` 解析文本并归一化坐标；
5. `parsing_response_to_pyautogui_code()` 生成 pyautogui 代码 → L4 执行并截图 → 历史经 `add_box_token()` 处理后进入下一轮。

### 模块内部调用

- `parse_action_to_structure_output()` → `smart_resize`（qwen25vl 分支计算 resize 尺寸）、`convert_point_to_coordinates`、`parse_action`、`escape_single_quotes`（type 内容转义）
- `parsing_response_to_pyautogui_code()` → `escape_single_quotes`
- `action_parser_test` → L3 三个对外 API
- `inference_test` → 仅导入 L3 常量（`smart_resize` 为本地副本，非导入）
- CI → `makefile` → unittest → L3

## 五、架构要点

仓库刻意把「模型推理」与「动作执行」都留在仓库外，`ui_tars` 包只做纯函数式的「文本 → 结构 → 代码」转换，因此零依赖、易嵌入 UI-TARS-desktop / OSWorld / Midscene 等宿主环境。
