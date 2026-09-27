# Permission Model

## macOS TCC 授权

Cua Driver 需要两项系统授权, 绑定在应用签名身份 `com.trycua.driver` 上:

1. **辅助功能 (Accessibility)** — 读取无障碍树, 模拟输入
2. **屏幕录制 (Screen Recording)** — 截图

### 正确授权方式

```shell
open -n -g -a CuaDriver --args serve     # 先让 app 起来, 使授权归属到 app
~/.local/bin/cua-driver permissions grant
```

**顺序很重要**: 必须先通过 `open` 启动 app, macOS 才会把权限请求归属到 `CuaDriver.app` 而不是发起调用的终端. 若直接跑 `permissions grant`, 授权可能记在终端程序名下, 导致 driver 用不上.

### 弹窗陷阱

macOS 授权对话框上的按钮是 **"Open System Settings"** 而不是 "Allow". 点击它只是把 CuaDriver 加进设置列表, **开关仍是关闭状态**, 必须手动打开:

- 系统设置 → 隐私与安全性 → 辅助功能 → 打开 CuaDriver
- 系统设置 → 隐私与安全性 → 屏幕与系统音频录制 → 打开 CuaDriver

系统可能提示需要退出并重开 CuaDriver — 应当同意, 授权只在 app 完全重启后生效.

macOS 不一定一次弹出两个授权框, 用 `permissions status` 确认实际到位情况, 缺哪个再跑一次.

### 状态查询的语义

```shell
~/.local/bin/cua-driver permissions status
```

该项**只读且不触发实时探测**, 通过运行中的 daemon 回答. **没有 daemon 时报告 `unknown` 而不是报告终端的授权**, 不会谎报 `granted`.

需要真实探测时用:

```shell
~/.local/bin/cua-driver check_permissions
```

它会返回 JSON, 其中 `source` 字段说明该结果归属哪个进程身份.

## 权限模式

模式在运行时启动时固定, 运行中无法更改 (改需重启 daemon).

| 模式 | 适用场景 |
|---|---|
| `standard` (默认) | 常规本地使用. 常规操作无提示, 残留边界 (如附着已登录的 Chromium 配置) 仍需显式授权 |
| `bounded` | 无人值守场景. **必须**提供能力清单, 清单外的范围默认拒绝 |
| `unrestricted` | 一次性或完全可信环境. 必须显式加 `--dangerously-bypass-approvals` |

```shell
cua-driver serve                                              # standard
cua-driver serve --permission-mode bounded \
  --capability-manifest ./cua-capabilities.yaml \
  --approve-capability-manifest
cua-driver serve --dangerously-bypass-approvals               # unrestricted
```

单独传 `--permission-mode unrestricted` 会 **fail closed** (失败即关闭), 不会默认放开.

授权在**原生运行时内部**强制执行: 位置在传输参数清洗之后、平台动作执行之前. 所有接入方式走同一个执行边界.

## daemon

`cua-driver serve` 提供常驻运行时. 查询状态:

```shell
cua-driver status
```

关注 `permission mode` 一行, 确认为预期模式 (默认应为 `standard (built_in_default)`).

macOS 上 autostart 目前**不支持**, 重启后 daemon 不会自动回来, 需要重新 `open -n -g -a CuaDriver --args serve`.

## 遥测

新安装**默认开启**. 官方声明收集匿名安装 ID 与无内容的使用元数据, 不收集提示词/工具参数/屏幕内容/文件路径.

关闭 (持久, 升级后依然保持关闭):

```shell
~/.local/bin/cua-driver telemetry disable
```

用 `cua-driver doctor` 确认 `telemetry: disabled via persisted`.

## 安全边界

**能力本身的性质**: 授权后 agent 可截图并模拟键鼠, 能够操作所有已登录应用. 这不是理论风险.

实际约束:

- **共享桌面** — 多个会话与独立进程共享同一桌面. 注册机制**不串行化**操作, 调用方需自行协调.
- **取消不回滚** — 被取消的调用可能已把输入送入应用.
- **不预留窗口** — 提供方不为某个会话保留窗口或完整工作流.
- 无人值守场景务必用 `bounded` + 手写能力清单, 不要图省事上 `unrestricted`.
