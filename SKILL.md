---
name: dsh-computer-use-cua-skill
description: 通过 Cua Driver CLI 直接操作本地桌面 (读取窗口/截图/点击/输入), 以及排查 DSH computer-use 预设挂载失败.
---

# DSH Computer Use via Cua Driver

本技能内容以 Cua Driver `0.30.1` 与 dsh `0.1.7-rc.2` 为基准编写.

## 核心结论

**优先用 CLI, 不要依赖 agent preset.**

`cua-driver call` 提供与 MCP 相同的工具能力, 但不经过 preset/profile/Loader 任何一层, 任何会话都能直接使用. 实测已验证可行.

## 章节索引

- [CLI Usage](references/cli-usage.md): 安装位置, 工具清单, 调用范式与输出形态.
- [Permission Model](references/permission-model.md): macOS TCC 授权, 权限模式, 遥测与安全边界.
- [Preset Mount](references/preset-mount.md): DSH computer-use preset 的组成与失败排查 (含已确认的 PATH 陷阱).
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
