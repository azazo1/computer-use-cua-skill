# DSH Computer-Use Preset 与失败排查

## 组成

DSH 的 computer-use 能力由三部分组成, **注册表服务 + 二选一的提供方**:

| 包 | 作用 | 是否需 `isolate` |
|---|---|---|
| `@deepseek-ai/dsh-computer-use` | 独占具名注册表, 无配置, 不添加模型可见工具 | 需与提供方同组 isolate |
| `@deepseek-ai/dsh-experimental-computer-use-cua-driver-mcp` | 通过 MCP 连接已安装的 `cua-driver` 可执行文件 | 同上 |
| `@deepseek-ai/dsh-experimental-computer-use-cua-driver-native` | 内嵌原生 npm SDK, 不依赖外部 CLI | 同上 |

**同一服务实例只允许注册一个提供方**, 第二个 (即便是同名实例) 会激活失败.

两个提供方都是官方标注的**实验性**包. npm 上其 `dist-tags` 长期停留在:

```
{"latest": "0.1.6-alpha.1", "alpha": "0.1.7-alpha.2", "next": "0.1.7-rc.2"}
```

连 `latest` 都是 alpha. 选用时必须和本地 dsh 运行时版本对齐, 否则 peer 依赖检查会直接拒绝安装.

## 声明写法

`computer-use` 组需要 `isolate`, 让服务实例与唯一消费者处在同一 realm:

```yaml
- id: computer-use
  name: cordis:group
  group: true
  isolate:
    computerUse: true
  config:
    - id: computer-use-service
      name: '@deepseek-ai/dsh-computer-use'
    - id: computer-use-cua-driver-mcp
      name: '@deepseek-ai/dsh-experimental-computer-use-cua-driver-mcp'
      config:
        command: cua-driver
        args: [mcp]
```

`config.command` 官方文档说明为 **"已安装的可执行程序路径或 PATH 命令"** — 两种都支持. 官方自己的 e2e 测试用的是**绝对路径**, 实践中绝对路径更可靠 (原因见下).

## 已确认的 PATH 陷阱

**这是最容易踩的坑, 且现象具有迷惑性.**

### 事实链

1. dsh 的 MCP transport 用 `scrubbedParentEnv()` 构造子进程环境:

   ```ts
   // packages/mcp/mcp-client/src/transport.ts
   function buildChildEnv(extra) {
     return { ...scrubbedParentEnv(), ...extra }
   }
   ```

   它读的是 **dsh 进程自己的 `process.env`**, 只清洗凭据形状变量与过期 `DSH_*`, **PATH 原样继承**.

2. macOS GUI 应用的 PATH 来自系统默认, **不读任何 shell rc 文件**:

   ```shell
   launchctl getenv PATH     # 通常为空
   cat /etc/paths /etc/paths.d/*   # 不含 ~/.local/bin
   ```

   因此 `~/.bashrc` 里的 `export PATH=...` 和 fish 的 `fish_add_path` **对 GUI 应用毫无作用**, 无论它们存在多久.

3. 若 `dsh-load-shell-env` 之类的插件把用户 shell 环境注入到了 **bash 工具**, 会造成**假象**: agent 的 shell 里 `which cua-driver` 成功, 但 dsh 自身 spawn 时仍找不到.

### 结论

```
dsh 进程 PATH (GUI 默认, 无 ~/.local/bin)
    ↓ scrubbedParentEnv 原样继承
MCP 子进程 PATH (同样没有)
    ↓
spawn("cua-driver") → ENOENT → 提供方激活失败
```

**勿轻信"重启 app 就好了"** — 重启不会让 GUI PATH 出现新目录. 这与 shell 配置何时添加无关.

### 处置建议

优先把 `command` 写成绝对路径 (`~/.local/bin/cua-driver` 的展开形式). 这是官方支持的一等用法, 不算打补丁.

若坚持用裸命令, 需要改的是**系统层** (LaunchAgent / 从终端启动 app / `launchctl setenv`), 而不是本项目配置. 注意 `launchctl setenv` 是会话级且重启失效的.

## 安装到 profile

用 `plugin_manager` 的 `install_bundle`, `target` 传 bundle 目录的绝对路径.

一个 bundle 的最小结构:

```
<name>/package.json      # 声明 dsh.bundle.patch
<name>/cordis.patch.yml  # 实际的 preset 声明
```

`package.json`:

```json
{
  "name": "dsh-computer-use-preset",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "files": ["cordis.patch.yml"],
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } },
  "dependencies": {
    "@deepseek-ai/dsh-computer-use": "<与运行时对齐>",
    "@deepseek-ai/dsh-experimental-computer-use-cua-driver-mcp": "<同上>"
  }
}
```

### 已知的 `ambiguous-install`

安装器通过"对比安装前后 `package.json` 依赖是否有变化"来判定装了什么. 对**同一个本地路径重复安装**时:

```
before == after → installed.length === 0 → ManagementFailure('ambiguous-install')
```

**处置**: 先 `remove_bundle` 再 `install_bundle`. 卸载只解除 `link:` 依赖, **不会删除源目录**.

## 验证步骤

```shell
# 1. 声明是否装载
#    plugin_manager list_plugins -> 找到 preset-<id> 行, 期望 enabled=true / fiberPhase="active"
# 2. config 是否被识别
#    cordis_inspect_query: platform=host, provider=Config, method=listConfigs
#      input={"entry": "include:preset-<id>"}  -> status 应为 "schema"
```

注意 `listConfigs` 只反映**声明是否装载**, **不能证明会话能成功 mount**. preset 是在**新建会话时**才实例化的.

## 排查断点

| 现象 | 可能原因 |
|---|---|
| 会话里选该 preset 报 "never started" | 提供方 spawn 失败; 优先查 PATH 与可执行文件是否存在 |
| 第二个提供方激活失败 | 注册表独占, 需卸载另一个 provider |
| 安装报 `ambiguous-install` | 同路径重复安装, 先 remove 再 install |
| `permissions status` 报 `unknown` | daemon 未运行 |

**排查时务必先取到真实报错文本再改配置.** 本技能的这些结论都是在拿到确切证据后才成立的.

## 替代路线

若 preset 挂载难以调通, **直接用 CLI 是更简单的选择**:

```shell
~/.local/bin/cua-driver call list_apps
```

它绕过 preset / profile / Loader 全部环节, 任何会话都能用, 能力与 MCP 完全一致 (同一份工具目录). 代价是多一层 bash 包装.
