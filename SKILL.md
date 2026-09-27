---
name: computer-use-cua-skill
description: 通过 Cua Driver 操作本地桌面 (读取窗口/截图/点击/输入), 含 macOS 授权模型与 GUI 宿主 spawn 子进程的 PATH 陷阱.
---

# Computer Use via Cua Driver

本技能内容以 Cua Driver `0.30.1` 为基准编写.

## 核心结论

**优先用 CLI, 不要让宿主去 spawn MCP 子进程.**

`cua-driver call` 提供与 MCP 相同的全部工具能力, 但不经过任何宿主的配置层. 任何环境都能直接使用, 排查成本最低.

## 章节索引

- [CLI Usage](references/cli-usage.md): 安装位置, 58 个工具的分组清单, 调用范式与输出形态.
- [Permission Model](references/permission-model.md): macOS TCC 授权, 权限模式, 遥测与安全边界.
- [Spawning From GUI](references/spawning-from-gui.md): GUI 应用拉不起 driver 的根因与处置 (跨平台通用).
- [Verify](references/verify.md): 从零验证环境是否可用.

## 操作前必读

操作真实桌面具有副作用, 且**取消调用无法撤销已送出的输入**. 使用前确认:

1. 优先选择只读工具 (`list_apps`, `list_windows`, `get_window_state`, 截图), 确有必要再动输入类工具.
2. 目标应用可能包含用户隐私内容 (聊天记录, 邮件, 文档). 截图与无障碍树会把这些内容读进上下文.
3. 多个会话/进程共享同一桌面, 调用之间桌面状态可能被他人改变.
4. 一次只跑一个 computer-use 工作流, 不要并发操作同一桌面.

## 快速开始

```shell
cua-driver call list_apps          # 列出应用 (只读)
cua-driver list-tools              # 列出全部可用工具
cua-driver describe <tool>         # 查看单个工具的参数 schema
```

工具名与参数见 [CLI Usage](references/cli-usage.md).

## 推荐工作流

1. `list_windows` 或 `get_accessibility_tree` — 确认目标窗口存在, 拿到 `pid` / `window_id`.
2. `get_window_state` — 取 AX 元素树, 优先用结构化 `elements` 定位.
3. 优先用 `element_index + window_id` 定位, 而非盲点坐标.
4. 执行动作.
5. 重新观测 — **已发出的点击不保证达成预期结果**, 必须重新验证.
