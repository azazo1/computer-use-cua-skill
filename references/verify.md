# Verify

从零确认环境可用. 任一步失败按对应章节处理.

## 1. 二进制与安装布局

```shell
~/.local/bin/cua-driver --version
~/.local/bin/cua-driver doctor
```

`doctor` 关注:

- `binary:` 版本与架构
- `install dir:` 应指向 `/Applications/CuaDriver.app/Contents/MacOS/cua-driver`
- `home dir:` 配置目录
- `telemetry:` 期望 `disabled via persisted`

## 2. 授权 (macOS)

```shell
~/.local/bin/cua-driver check_permissions
```

期望 JSON:

```json
{
  "accessibility": true,
  "screen_recording": true,
  "source": { "bundle_id": "com.trycua.driver", ... }
}
```

**注意**: `permissions status` 是只读的归属查询, **不做实时探测**. 要真实探测用 `check_permissions`. 若 `status` 报 `unknown`, 通常是 daemon 没跑.

归属身份必须是 `com.trycua.driver`. 若归属成终端程序, 说明授权方式不对 — 参考 [Permission Model](permission-model.md) 重新授权.

## 3. daemon

```shell
~/.local/bin/cua-driver status
```

期望:

```
Cua Driver daemon is running
  permission mode: standard (built_in_default)
```

## 4. 桌面可读性 (关键)

```shell
~/.local/bin/cua-driver call list_apps
```

期望返回 JSON, `apps` 数组包含你认识的**已运行的 GUI 应用**并带真实 `pid`.

**若列表为空**: 不是权限问题就是当前桌面会话没有可操作的应用 — 先打开一个 GUI 应用再重试.

单凭 `--version` 成功**不能**证明桌面访问可用.

## 5. 工具目录

```shell
~/.local/bin/cua-driver list-tools
```

期望列出约 58 个工具.

## 6. 完整验证脚本

```shell
set -e
CUA="$HOME/.local/bin/cua-driver"

echo "== version ==";     "$CUA" --version
echo "== permissions =="; "$CUA" check_permissions
echo "== status ==";      "$CUA" status
echo "== desktop ==";     "$CUA" call list_apps | head -20
echo "== tools ==";       "$CUA" list-tools | wc -l
```
