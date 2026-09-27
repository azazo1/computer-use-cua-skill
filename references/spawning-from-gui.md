# Spawning the Driver From a GUI Host

当 Cua Driver 不是由你在终端里手动调用, 而是由某个 **GUI 应用**(agent harness, 编辑器, 桌面工具)以子进程方式拉起时, 有一个隐蔽的失败模式.

本文只讨论这个跨平台通用的坑, 不涉及任何特定 harness 的配置格式.

## 症状

- 在终端里 `cua-driver` 一切正常
- 但 GUI 应用里始终报 **"never started"** / 子进程无法启动 / ENOENT
- 明明 `which cua-driver` 在终端里能解析

## 根因: GUI 应用不继承 shell 的 PATH

### 事实

子进程通常继承**父进程自己的 `process.env`**. 而 GUI 应用的 PATH 来自系统默认, **不读任何 shell 配置文件**:

```shell
launchctl getenv PATH            # macOS 上通常为空 → 用系统默认
cat /etc/paths /etc/paths.d/*    # 系统级 PATH, 这里没有 ~/.local/bin
```

也就是说:

- `~/.bashrc` / `~/.zshrc` / `~/.config/fish/config.fish` 里的 `export PATH=...`
- fish 的 `fish_add_path`
- 任何 shell 启动时执行的东西

**统统对 GUI 应用无效**, 无论它们存在多久, 也无论 GUI 应用重启多少次.

### 为什么容易误判

两个假象会让人走弯路:

1. **"我终端里能用啊"** — 终端是 shell 的子进程, 当然能用. 这证明不了 GUI 能用.
2. **"我重启过 app 了"** — 重启不会让系统 PATH 长出新的目录. 这个方向是死路.

如果 GUI 应用内部**还提供了自己的 shell 工具**(很多 agent harness 会读取用户 shell 环境来丰富工具的 PATH), 假象会更强: **那些工具能看到可执行文件, 但宿主 spawn 子进程时看不到**. 两者用的是不同的环境来源.

## 处置

### 首选: 用绝对路径

不要把可执行文件的查找交给 PATH, 直接写绝对路径.

macOS 上用官方安装脚本产生的**符号链接**, 而不是 app bundle 内部路径:

```
~/.local/bin/cua-driver          # 推荐 (稳定入口, 升级不失效)
/Applications/CuaDriver.app/Contents/MacOS/cua-driver   # app 内部, 不建议写死
```

若宿主只接受配置文件, 写展开后的绝对路径 (YAML 等格式里 `~` 不会自动展开).

### 备选: 改系统层环境

只有当必须保留裸命令时才考虑, 且要明白代价:

| 方案 | 代价 |
|---|---|
| `launchctl setenv PATH ...` | 会话级, **重启失效**, 需配合 LaunchAgent 才能持久 |
| 从终端启动 GUI 应用 | 只对那一次启动有效 |
| 配 LaunchAgent 注入 | 引入系统级配置, 为一个应用改动过大 |

## 验证方法

复现 GUI 环境做测试, **不要用当前 shell 的 PATH** —— 那样会得到**假阳性**:

```python
import subprocess, json

CMD = "/Users/<you>/.local/bin/cua-driver"     # 绝对路径
GUI_PATH = "/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"

p = subprocess.Popen([CMD, "mcp"],
    stdin=subprocess.PIPE, stdout=subprocess.PIPE, text=True, bufsize=1,
    env={"PATH": GUI_PATH, "HOME": "/Users/<you>"})

p.stdin.write(json.dumps({"jsonrpc":"2.0","id":1,"method":"initialize","params":{
    "protocolVersion":"2024-11-05","capabilities":{},
    "clientInfo":{"name":"probe","version":"1.0"}}}) + "\n")
p.stdin.flush()
print(p.stdout.readline())    # 期望含 serverInfo: cua-driver ...
p.kill()
```

用精简 PATH 跑通, 才说明配置真的不依赖 PATH.

## 排查原则

**先取到真实报错文本, 再改配置.**

"never started" 这类笼统提示可能来自多个环节 (可执行文件找不到 / 握手失败 / 参数错误 / 权限不足). 在没拿到具体错误前反复调整配置, 只是在猜.
