# CLI Usage

## 安装位置

macOS 官方安装脚本的落点:

| 路径 | 说明 |
|---|---|
| `/Applications/CuaDriver.app` | 应用本体, 承载 macOS TCC 授权身份 `com.trycua.driver` |
| `~/.local/bin/cua-driver` | 指向 app 内部二进制的**符号链接**, 是稳定入口 |

**不要**把 app 装到临时目录或非 `/Applications` 位置. TCC 授权绑定签名身份与 bundle 位置, 移动后授权失效. 用官方脚本安装:

```shell
/bin/bash -c "$(curl -fsSL https://cua.ai/driver/install.sh)"
```

符号链接是升级后依然稳定的入口, 引用时优先用它而不是 app bundle 内部路径.

## 调用范式

```shell
cua-driver list-tools                    # 全部工具名 + 一句话说明
cua-driver describe <tool>               # 单个工具的完整参数 schema
cua-driver call <tool>                   # 调用工具
cua-driver call <tool> key=value ...     # 带参数调用
cua-driver call <tool> --json '{"k":1}'  # 复杂参数用 JSON
```

输出默认为 JSON, 可直接交给 `python3 -m json.tool` 或 `jq` 处理.

## 工具分组 (共 58 个)

### 只读观测 (优先使用)

| 工具 | 用途 |
|---|---|
| `list_apps` | 列出运行中与已安装的应用, 含 pid / bundle_id / state |
| `list_windows` | 列出 WindowServer 已知的 layer-0 顶层窗口 |
| `get_accessibility_tree` | 全桌面轻量快照: 应用 + 可见窗口 + bounds + z-order |
| `get_window_state` | 单个应用的 AX 树, 返回结构化 `elements` 与 Markdown 两种形态 |
| `get_desktop_state` | 全屏截图 (真实屏幕像素) |
| `zoom` | 窗口局部裁剪截图, 自动加 20% 边距 |
| `get_screen_size` | 主显示器逻辑尺寸与缩放因子 |
| `get_cursor_position` | 当前鼠标位置 |
| `check_permissions` | 报告无障碍与屏幕录制授权状态 |

### 输入操作 (有副作用)

| 工具 | 用途 |
|---|---|
| `click` / `double_click` / `right_click` | 按 pid 或坐标点击 |
| `drag` | 从 (from_x,from_y) 拖到 (to_x,to_y), 使用窗口局部截图坐标 |
| `move_cursor` | 移动鼠标 |
| `scroll` | 滚动目标 pid |
| `type_text` | 通过 `AXSetAttribute(kAXSelectedText)` 插入文本 |
| `press_key` / `hotkey` | 单键 / 组合键 |
| `set_value` | 直接设值到 UI 元素 (比模拟输入更可靠) |
| `invoke_menu` | 按菜单路径逐级定位并调用 |

### 应用与窗口管理

| 工具 | 用途 |
|---|---|
| `launch_app` | 后台启动 macOS 应用 (**不会**把目标带到前台) |
| `bring_to_front` | 持续激活应用并置于前台 |
| `kill_app` | 强制终止进程 |
| `set_window_frame` | 设置窗口 frame 并独立回读校验 |

### 浏览器 (需显式绑定)

`browser_prepare` / `browser_navigate` / `browser_click` / `browser_type` / `browser_pointer` / `browser_dialog` / `browser_download` / `browser_set_input_files` / `get_browser_state`.
操作需先 `browser_prepare` 准备受控的 DevTools 端点, 且作用于**精确绑定**的标签页.

### 剪贴板

`clipboard_read` / `clipboard_write`. 剪贴板属隐私敏感内容, 读取前应确认必要性.

### 校验与回放

| 工具 | 用途 |
|---|---|
| `verify_state` | 对单个窗口做确定性谓词校验 |
| `start_recording` / `stop_recording` / `get_recording_state` | 轨迹录制 |
| `replay_trajectory` | 按词序重放已录制的每轮工具调用 |
| `health_report` | 单次端到端诊断 |

### 生命周期与会话

`start_session` / `end_session` / `list_sessions` / `get_session`. 会话用于组织一次完整工作流的清理钩子.

### 配置与维护

`get_config` / `set_config` / `check_for_update` / `install_ffmpeg` / `install_extension`.

## 推荐工作流

1. `list_windows` 或 `get_accessibility_tree` — 确认目标窗口存在并拿到 `pid` / `window_id`.
2. `get_window_state` — 取 AX 元素树, 优先用结构化 `elements` 定位元素.
3. 优先用 `element_index + window_id` 定位而非盲点坐标; 坐标仅作兜底.
4. 执行动作.
5. `get_window_state` 重新观测 — **已发出的点击不保证达成预期结果**, 必须重新验证.

## 重要语义

- **`element_token` 会失效**: 对同一窗口重新快照后, 之前的元素令牌作废, 需要重新获取.
- **截图坐标系**: `drag` 使用 `get_window_state` 返回的窗口局部截图像素坐标; `get_desktop_state` 使用屏幕像素.
- **取消不回滚**: 调用被取消时已经送入应用的输入无法撤销, 重试前必须重新读取当前状态.
- **`target` 与 `pid`/`window_id` 二选一**, 不要混用.
