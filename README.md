# FlowMedia-Plugins

FlowMedia 官方插件市场。开发者可按本文创建 Python 插件、打包并提交 Pull Request；仓库维护者审核合并后，用户可在 FlowMedia 的“插件管理 → 插件市场”中刷新、安装和配置。

本文说明 **SDK 1 与 SDK 2、Python 3.11、清单格式 1** 的插件规范，包含可运行示例与独立打包脚本。开发者无需取得或提交 FlowMedia 主程序源码。

**兼容性：当前 `main` 已包含 SDK 2 插件。仅支持 SDK 1 的旧客户端可能无法读取整个插件市场，而不只是无法安装某个插件。使用此市场前，请确认主程序明确支持 SDK 2；仅看到“插件管理”入口不能证明兼容。** 插件版本、SDK 版本、清单格式版本是不同概念，不能通过改写插件清单绕过主程序能力要求。

## 已发布插件与安装入口

| 插件 | 最新版本 | 所需 SDK | 功能与说明 |
| --- | --- | --- | --- |
| [网络代理监控](docs/network_proxy_monitor.md) | [1.0.0](plugins/network_proxy_monitor/1.0.0/) | 2 | Clash / v2rayA 状态、节点、分组与真实流量；按服务能力提供探测、故障切换和优选。 |

安装已发布插件：

1. 确认主程序支持插件所需 SDK，并满足[运行环境要求](#deployment)。
2. 在“插件管理 → 配置”填写仓库 `treehello0210/FlowMedia-Plugins`、分支 `main`，保存后进入“插件市场”刷新。公开仓库的浏览与安装无需 GitHub Token。
3. 选择插件并安装。也可[下载网络代理监控 ZIP](https://github.com/treehello0210/FlowMedia-Plugins/raw/refs/heads/main/plugins/network_proxy_monitor/1.0.0/plugin.zip)，通过页面顶部“上传”导入，再从“本地”安装。
4. 安装后默认停用。按插件说明保存配置；支持连接测试的插件先测试，再启用。网络代理监控需填写管理 API 地址，不能使用 HTTP/SOCKS 代理转发端口。

v2rayA 2.2.7.5 的探测可能影响核心运行，因此本插件仅提供监控；流量与控制能力差异详见[插件说明](docs/network_proxy_monitor.md)。界面只展示接口实际提供的数据。

本仓库公开，插件 ZIP 内的 Python 源码可被下载查看。提交范围应限于插件代码、元数据和说明；无需包含 FlowMedia 主程序实现，禁止上传用户配置、凭据或订阅链接。

## 开发文档导航

[能力与兼容性](#capabilities) · [SDK 1 入门](#quick-start) · [清单与权限](#manifest) · [配置表单](#configuration) · [SDK、日志与调度](#sdk) · [打包](#packaging) · [市场目录](#catalog) · [提交插件](#contributing) · [部署与持久化](#deployment) · [检查与排错](#troubleshooting) · [SDK 2 网络代理接口](#proxy-sdk)

<a id="capabilities"></a>

## 1. 先确认插件能力是否适合需求

两个 SDK 共用以下基础能力：

- Python 标准库及随 ZIP 分发的纯 Python 本地模块。
- 由 JSON 清单生成的配置表单。
- 按插件 ID 隔离的 JSON 持久化数据。
- 手动执行、启用事件、Cron 定时任务。
- 有限的数值执行结果与系统既有日志消息。

运行器每次执行创建独立容器；插件无法访问网络、主程序数据库、网盘账户、下载器、媒体目录、Docker socket 或 FlowMedia 的内部 Python 模块。`requests`、`httpx` 等第三方包不属于可用依赖，也不会执行 `pip install`。MoviePilot 等其他项目的插件不能直接复制使用。

| 能力 | SDK 1 | SDK 2 |
| --- | --- | --- |
| 插件进程直接访问网络、主程序内部模块或业务数据 | 不支持 | 不支持 |
| 配置、持久化数据、手动执行、启用事件、Cron、固定模板日志 | 支持 | 支持 |
| 主程序代为采集 Clash / v2rayA 状态 | 不支持 | 声明 `proxy.monitor` 后支持 |
| 主程序代为执行已支持的节点探测、选择与配置策略 | 不支持 | 同时声明 `proxy.monitor`、`proxy.control` 后支持 |
| 第三方依赖、插件自带前端脚本、任意 HTTP 请求 | 不支持 | 不支持 |

SDK 1 的 `permissions` 必须为 `[]`。SDK 2 仅允许[第 3 节](#manifest)列出的两种权限组合；`dependencies` 在两个 SDK 中都必须为 `[]`。SDK 2 不是通用远程 API 扩展机制，其他服务或业务集成仍需主程序增加相应能力，不能自行发明权限名称或更改 SDK 版本获得访问权。

<a id="quick-start"></a>

## 2. 十分钟创建一个可用插件

以下 **SDK 1** 插件每次运行把已保存的计数加上配置中的“数量”。它不依赖网络或外部文件，可由兼容 SDK 1 的主程序运行；SDK 2 网络代理插件的接口与示例见[第 11 节](#proxy-sdk)。

创建开发目录：

```text
workspace/
├── my_counter/
│   ├── manifest.json
│   └── plugin.py
├── build_plugin.py       # 第 6 节的独立打包脚本
└── FlowMedia-Plugins/    # 本仓库的本地副本或打包输出目录
```

`my_counter/manifest.json`：

```json
{
  "schema_version": 1,
  "id": "my_counter",
  "name": "计数",
  "version": "1.0.0",
  "sdk_version": "1",
  "python_version": "3.11",
  "entry": "plugin.py",
  "class_name": "Plugin",
  "description": "",
  "permissions": [],
  "dependencies": [],
  "config_schema": {
    "type": "object",
    "properties": {
      "step": {
        "type": "integer",
        "title": "数量",
        "default": 1,
        "minimum": 1,
        "maximum": 100
      }
    },
    "required": ["step"],
    "additionalProperties": false
  },
  "schedules": [
    {"id": "hourly", "cron": "0 * * * *", "method": "run"}
  ],
  "events": [
    {"event": "plugin.manual", "method": "run"}
  ]
}
```

`my_counter/plugin.py`：

```python
from flowmedia_plugin import FlowMediaPlugin


class Plugin(FlowMediaPlugin):
    def run(self, event=None):
        count = self.get_data("count", 0) + self.config.get("step", 1)
        self.save_data("count", count)
        self.notify("success")
        return {"count": count}
```

将第 6 节脚本保存为 `workspace/build_plugin.py`，在 `workspace` 运行：

```bash
python build_plugin.py my_counter FlowMedia-Plugins YOUR-GITHUB-USERNAME
```

将 `YOUR-GITHUB-USERNAME` 替换为自己的 GitHub 用户名。命令生成：

```text
FlowMedia-Plugins/plugins/my_counter/1.0.0/
├── manifest.json
└── plugin.zip
```

随后在 FlowMedia 中：

1. 打开“插件管理”，点击页面顶部“上传”，选择生成的 `plugin.zip`。此步只加入本地插件库，不执行插件。
2. 切换到“本地”，点击“计数”卡片，再点击“安装”。安装后默认“停用”。
3. 在“已安装”的插件卡片中设置“数量”，点击“保存”，重新打开卡片并点击“启用”。
4. 点击“开始执行”。任务实际成功后，系统日志出现现有消息“操作成功”；插件状态恢复“启用”，执行时间更新。
5. 保持启用时，示例每小时执行一次。测试后可点击“停用”。

“开始执行”的“操作完成”提示表示执行请求已提交；插件在后台运行，仍需查看卡片状态、执行时间和系统日志确认结果。通用插件卡片不会展示 `return {"count": count}` 的计数值，也没有浏览任意插件持久化数据的页面。SDK 2 网络代理监控面板展示的是主程序提供的服务快照，不是任意 `run()` 返回值。

示例 ID 仅用于学习。发布自己的插件时，应选择仓库中尚未使用的唯一 ID，并保持后续版本的 ID 不变。

<a id="manifest"></a>

## 3. ZIP 内的 manifest.json

清单是 UTF-8 JSON 对象，不支持注释、重复键或 `NaN`/`Infinity`。不接受下表之外的顶层字段。

| 字段 | 必填 / 默认值 | 规则 |
| --- | --- | --- |
| `schema_version` | 必填 | 整数 `1`，不能写成字符串或布尔值。 |
| `id` | 必填 | 3–64 个字符，匹配 `[a-z][a-z0-9_]{2,63}`。以小写字母开头，只含小写字母、数字和下划线。 |
| `name` | 必填 | 非空名称，最多 100 个字符，用于卡片标题。 |
| `version` | 必填 | 三段数字版本，例如 `1.0.0`。每段为 `0` 或无前导零的 1–9 位整数；不支持 `v1.0.0`、`01.0.0`、预发布及构建后缀。 |
| `sdk_version` | 必填 | 字符串 `"1"` 或 `"2"`；主程序必须支持所选版本。 |
| `python_version` | 必填 | 字符串 `"3.11"`。 |
| `entry` | 必填 | 固定为 `"plugin.py"`，且文件必须位于 ZIP 根目录。 |
| `class_name` | `"Plugin"` | 入口类名。名称匹配 `[A-Za-z][A-Za-z0-9_]{0,63}`；该类必须在 `plugin.py` 顶层定义并继承 `FlowMediaPlugin`。 |
| `description` | `""` | 最多 1000 个字符，可留空。 |
| `permissions` | 必填 | SDK 1 固定为 `[]`；SDK 2 限下表中的两种完整组合。 |
| `dependencies` | `[]` | 固定为 `[]`，不支持声明第三方依赖。 |
| `config_schema` | 空对象表单 | 第 4 节定义的平面 JSON Schema 子集。建议始终显式写出完整结构。 |
| `schedules` | `[]` | 最多 16 个定时任务，详见第 5 节。 |
| `events` | `[]` | 最多 32 个事件项，但当前只开放两个互不重复的事件名，详见第 5 节。 |

权限组合必须与 SDK 版本一致，不接受额外权限、重复项或其他组合：

| `sdk_version` | `permissions` 的完整值 | 用途 |
| --- | --- | --- |
| `"1"` | `[]` | 基础离线插件。 |
| `"2"` | `["proxy.monitor"]` | 使用主程序提供的网络代理监控快照与界面。 |
| `"2"` | `["proxy.monitor", "proxy.control"]` | 在监控基础上提供受控探测、手动选择及已配置策略。 |

SDK 2 同样使用 `schema_version: 1`；不要将两个版本字段混用，也不要给 SDK 2 填写空权限数组。

名称、描述、字段标题等文本不允许除换行和制表符以外的 ASCII 控制字符。清单不提供 `author`、图标、项目主页、远程下载地址或自定义表单代码等额外键；发布者 `author` 位于仓库外层描述符中。

**版本不可变：** 同一 `id + version` 对应的 ZIP 字节不能被替换。即使只改了注释、ZIP 时间戳或压缩方式，也应增加版本号再提交。重新上传完全相同的 ZIP 可复用已有版本。

<a id="configuration"></a>

## 4. 配置表单

通用配置表单来自 `config_schema`，不会调用插件代码生成界面。SDK 1 没有 `get_form()`、`get_state()`、动态选项请求或主程序业务 API；两个 SDK 都不允许插件自带 HTML/JavaScript。

SDK 2 网络代理能力在同一份平面 Schema 上，由主程序提供固定界面：根据 `provider` 切换显示的配置项，从真实快照生成分组选项，并为 Clash 提供域名规则编辑器。规则仍序列化到 `domain_rules` 字符串字段，不代表 Schema 开放了嵌套对象或数组。字段约定见[第 11 节](#proxy-sdk)。

Schema 顶层只允许以下四个键：

```json
{
  "type": "object",
  "properties": {},
  "required": [],
  "additionalProperties": false
}
```

`type` 必须为 `object`；`properties` 默认 `{}`，最多 64 个字段；`required` 默认 `[]`，每项必须引用已定义字段且不能重复；`additionalProperties` 默认 `false`，不能设为 `true`。不支持嵌套对象、数组、`$ref`、`oneOf`、正则 `pattern` 或表达式。

字段名匹配 `[A-Za-z][A-Za-z0-9_]{0,63}`。避免使用前端保留名称 `constructor`、`prototype`。字段定义只接受：

| 属性 | 用途与限制 |
| --- | --- |
| `type` | 必填，限 `string`、`boolean`、`integer`、`number`。分别展示文本框、开关、整数输入框、数值输入框。 |
| `title` | 参数名称，最多 500 个字符；省略时显示字段名。 |
| `description` | 参数下方说明，最多 500 个字符。 |
| `default` | 初始值；必须满足该字段的类型、范围、长度和枚举约束。 |
| `enum` | 1–100 个符合字段类型与约束的值；通用表单展示选择框，选项文案为值本身。SDK 2 专用界面可由主程序映射显示标签。 |
| `minimum` / `maximum` | 仅限 `integer` / `number`，必须为有限数值，最小值不得大于最大值。 |
| `minLength` / `maxLength` | 仅限 `string`，必须是 0–4096 的整数；默认长度范围为 0–4096。 |
| `format` | 仅字符串可设置，唯一允许值为 `password`，展示密码框和小眼睛；保存后重新打开仍回填已有值，默认遮蔽。 |

数值不接受 `NaN`、无穷大，布尔值不能代替整数。配置保存和启用时会再次校验，整体配置的 UTF-8 JSON 不得超过 64 KiB。缺少的字段先应用默认值，再检查必填项；建议给可直接运行的必填字段设置有效默认值。

密码字段不授予网络或凭据访问权限。SDK 1 会像其他字段一样将其传入插件配置；SDK 2 主程序会在创建插件执行载荷前剔除所有 `format: "password"` 字段，凭据仅供主程序适配器使用。因此 SDK 2 插件不能依赖 `self.config` 读取这些字段。

不要把真实密码、Token 或 Cookie 写入默认值、源码、示例或公开 ZIP；插件配置与持久化 JSON 不应被视为凭据加密存储。插件市场配置中的“GitHub Token”另按用户和仓库加密管理，不会传入任一 SDK 的插件。

<a id="sdk"></a>

## 5. SDK、事件与定时任务

运行器在容器内提供模块 `flowmedia_plugin`。无需安装 SDK 包，也不要在 ZIP 中自带同名模块。入口类必须可无参数实例化；执行方法是普通同步方法，不能使用 `async def`。

### 5.1 生命周期与方法签名

| 方法 | 行为 |
| --- | --- |
| `init_plugin(self, config=None)` | 每次执行时调用。默认实现将主程序提供的配置深拷贝到 `self.config`；SDK 2 收到的是已剔除密码格式字段的配置。重写时先调用 `super().init_plugin(config)`。 |
| `run(self, event=None)` | 默认执行入口；接收本次事件字典。可声明其他同样接收 `event=None` 的同步处理方法，再在清单中引用。 |
| `stop_service(self)` | 本次执行的收尾回调；没有参数，默认不做操作。异常时运行器也会尝试调用，回调抛出异常会使本次执行失败。强制停用、超时、进程退出等情况下不能保证它执行。 |
| `get_service_snapshot(self)` | SDK 2：返回本次执行服务快照的深拷贝，未提供快照时返回 `{}`；调用本身不发起请求。详细字段见第 11 节。 |
| `get_data(self, key=None, default=None)` | 读取本插件上次成功保存的数据；指定键时返回值或默认值，`key=None` 返回整个数据字典。返回深拷贝，直接修改返回值不会自动保存。 |
| `save_data(self, key, value)` | 更新本次执行的数据快照。键必须是非空字符串，最多 128 个字符；值须可编码为标准 JSON，全部数据合计最多 64 KiB。只有执行、结果校验与清理成功后才由主程序落盘。 |
| `notify(self, template="success", level="info")` | 添加一条固定模板日志，单次最多 16 条，详见下一节。它不发送 Telegram、邮件或其他外部通知。 |

每次执行都会新建插件对象：准备配置和已保存数据 → `init_plugin(config)` → 执行指定方法 → `stop_service()` → 结果校验与容器清理 → 主程序保存数据和日志。实例属性和模块全局变量不跨次保留；需要长期状态时使用 `get_data` / `save_data`。`/tmp` 也是一次性空间。

SDK 2 由主程序在创建 worker 前采集服务快照；声明控制能力时，主程序还会按配置执行策略。快照在 `init_plugin()` 前注入，也随事件的 `service_snapshot` 字段提供。它表示采集时的状态，不能当作随后控制动作的成功回执；策略和服务控制不由插件 Python 代码直接执行。

`run()` 及其他处理方法可返回 `None`、布尔值、有限整数/浮点数，以及由这些值组成的列表或字典。**不支持字符串结果**。字典键须匹配 `[A-Za-z][A-Za-z0-9_]{0,63}`，每层列表/字典最多 100 项，递归深度检查上限为 8，结果 JSON 最多 8 KiB。持久化 `save_data` 的值可以包含 JSON 字符串，它与执行结果的限制不同。

配置、事件和数据组成的输入 JSON 合计上限为 90 KiB；SDK 2 的服务快照也计入事件大小。结果、数据、消息等组成的总输出上限为 80 KiB，因此不要同时把各子项写满到各自上限。

### 5.2 日志

| 调用 | 系统既有文案 |
| --- | --- |
| `self.notify("success")` | `操作成功` |
| `self.notify("failure", "warning")` | `请求失败` |
| `self.notify("invalid", "error")` | `参数错误` |

`template` 只允许 `success` / `failure` / `invalid`；`level` 只允许 `info` / `warning` / `error`。原始 stdout/stderr 被丢弃，`print()`、自定义 `logging` 文本和异常堆栈不会进入主程序日志。仅任务成功完成后，合法模板消息才会写入系统日志；运行失败时主程序使用既有“操作失败”。

### 5.3 事件

`events` 每项必须恰好包含 `event` 和 `method`：

```json
[
  {"event": "plugin.started", "method": "on_started"},
  {"event": "plugin.manual", "method": "run"}
]
```

- `plugin.started`：手动启用成功后，以及主程序重启并恢复已启用插件时触发。处理方法收到 `{"type": "plugin.started"}`。未声明时不会自动执行 `run()`。
- `plugin.manual`：点击“开始执行”时触发。处理方法收到 `{"type": "plugin.manual"}`。未声明时默认调用 `run()`。
- 每种事件只能出现一次；目前没有媒体入库、下载完成、网盘变化等事件。

上述事件示例只列出基础字段；SDK 2 的实际事件还会携带 `service_snapshot`。

上面的 `on_started` 必须由入口类实际实现，例如：

```python
def on_started(self, event=None):
    self.notify("success")
    return None
```

把该方法缩进放入 `Plugin` 类中，不要作为模块级函数。不能将 `init_plugin`、`stop_service`、`get_data`、`save_data`、`notify` 用作事件或定时处理方法；它们是运行器保留方法。

`get_service_snapshot()` 仅供插件方法内读取快照，请勿将它声明为事件或定时入口。它不接受 `event` 参数，不符合处理方法签名；这里是接口使用要求，不表示当前加载器已将该名称列入保留名单。

### 5.4 Cron

`schedules` 每项必须恰好包含 `id`、`cron`、`method`。`id` 在本插件中唯一，`id` 和 `method` 均匹配 `[A-Za-z][A-Za-z0-9_]{0,63}`。

```json
[
  {"id": "hourly", "cron": "0 * * * *", "method": "run"}
]
```

定时处理方法收到的基础字段为 `{"type": "schedule", "id": "hourly"}`；SDK 2 另附 `service_snapshot`。Cron 复用 FlowMedia 的解析器，时区固定为 `Asia/Shanghai`：

| 格式 | 顺序 | 示例 |
| --- | --- | --- |
| 五段 | 分 时 日 月 周 | `0 * * * *`：每小时第 0 分钟。 |
| 六段 | 秒 分 时 日 月 周 | `0 */10 * * * *`：每 10 分钟。 |
| 七段 | 秒 分 时 日 月 周 年 | `0 0 9 * * mon-fri *`：每个工作日 09:00。 |

表达式最多 100 个字符，支持数字、范围、逗号列表、步长，月/周英文简称；日期支持 `L` / `last`，日或周可单独用 `?`，但不能两者都用 `?`。不是 Quartz 语义：数字星期为星期一 `0` 到星期日 `6`；日与周条件同时限制时，两者必须同时满足。建议优先使用上表中的简单表达式。

同一个插件同时最多一个任务；运行中再次触发会跳过，不累计成队列。整个运行器最多两个插件并行。调度器合并积压触发，允许的错过触发时间为 30 秒；不要把它当作保证每次必达的消息队列。

<a id="packaging"></a>

## 6. 打包：只需 Python 3.11 标准库

### 6.1 独立打包脚本

将下列脚本保存到源码目录外的 `build_plugin.py`。它生成 ZIP、计算真实 SHA-256/字节数，并写入市场所需的外层描述符。脚本不执行插件代码，也不需要 FlowMedia 源码或第三方包。

脚本可用于 SDK 1 或 SDK 2，但只做打包所需的基础检查，不验证主程序是否支持清单声明的 SDK 或服务能力；**最终清单、Cron、源码与 ZIP 合法性以对应版本的 FlowMedia 上传校验为准**。提交前必须完成第 2 节的本地上传与运行验证。若需要更完整的离线校验且已获得 FlowMedia 后端源码，见第 6.3 节。

```python
import hashlib
import io
import json
from pathlib import Path
import re
import stat
import sys
import zipfile


def build(source, repository, author):
    source = Path(source).resolve()
    repository = Path(repository).resolve()
    if not source.is_dir() or repository.is_relative_to(source):
        raise ValueError("参数错误")
    if not re.fullmatch(r"[A-Za-z0-9](?:[A-Za-z0-9-]{0,37}[A-Za-z0-9])?", author):
        raise ValueError("参数错误")

    manifest = json.loads((source / "manifest.json").read_text(encoding="utf-8"))
    plugin_id, version = manifest["id"], manifest["version"]
    if not re.fullmatch(r"[a-z][a-z0-9_]{2,63}", plugin_id):
        raise ValueError("参数错误")
    if not re.fullmatch(r"(?:0|[1-9][0-9]{0,8})\.(?:0|[1-9][0-9]{0,8})\.(?:0|[1-9][0-9]{0,8})", version):
        raise ValueError("参数错误")

    # Keep the catalog manifest identical to FlowMedia's normalized manifest.
    manifest.setdefault("class_name", "Plugin")
    manifest.setdefault("description", "")
    manifest.setdefault("dependencies", [])
    manifest.setdefault("schedules", [])
    manifest.setdefault("events", [])
    schema = manifest.setdefault("config_schema", {"type": "object"})
    schema.setdefault("properties", {})
    schema.setdefault("required", [])
    schema.setdefault("additionalProperties", False)
    encoded_manifest = (json.dumps(manifest, ensure_ascii=False, indent=2, allow_nan=False) + "\n").encode("utf-8")
    if len(encoded_manifest) > 65536:
        raise ValueError("参数错误")

    files = []
    allowed = {".py", ".json", ".txt", ".md", ".png", ".jpg", ".jpeg", ".webp"}
    for path in sorted(source.rglob("*")):
        if path.is_symlink() or getattr(path.lstat(), "st_file_attributes", 0) & 0x400:
            raise ValueError("参数错误")
        if path.is_dir():
            continue
        if not path.is_file() or path.suffix.lower() not in allowed:
            raise ValueError("参数错误")
        name = path.relative_to(source).as_posix()
        raw = encoded_manifest if name == "manifest.json" else path.read_bytes()
        if len(raw) > 5 * 1024 * 1024:
            raise ValueError("参数错误")
        files.append((name, raw))
    if not {"manifest.json", "plugin.py"}.issubset(name for name, _ in files) or len(files) > 128:
        raise ValueError("参数错误")

    output = io.BytesIO()
    with zipfile.ZipFile(output, "w", compression=zipfile.ZIP_STORED) as archive:
        for name, raw in files:
            info = zipfile.ZipInfo(name, date_time=(2026, 1, 1, 0, 0, 0))
            info.create_system = 3
            info.external_attr = (stat.S_IFREG | 0o644) << 16
            archive.writestr(info, raw)
    archive_bytes = output.getvalue()
    if not 0 < len(archive_bytes) <= 5 * 1024 * 1024:
        raise ValueError("参数错误")

    relative = f"plugins/{plugin_id}/{version}/plugin.zip"
    descriptor = {
        "manifest": manifest,
        "sha256": hashlib.sha256(archive_bytes).hexdigest(),
        "size": len(archive_bytes),
        "author": author,
        "package": relative,
    }
    descriptor_bytes = (json.dumps(descriptor, ensure_ascii=False, indent=2, allow_nan=False) + "\n").encode("utf-8")
    if len(descriptor_bytes) > 65536:
        raise ValueError("参数错误")
    target = repository / relative
    outputs = [(target, archive_bytes), (target.with_name("manifest.json"), descriptor_bytes)]
    for path, raw in outputs:
        if path.exists() and path.read_bytes() != raw:
            raise ValueError("参数错误")
    target.parent.mkdir(parents=True, exist_ok=True)
    for path, raw in outputs:
        path.write_bytes(raw)


if __name__ == "__main__":
    try:
        if len(sys.argv) != 4:
            raise ValueError("参数错误")
        build(*sys.argv[1:])
    except Exception:
        print("参数错误", file=sys.stderr)
        raise SystemExit(1) from None
    print("保存成功")
```

脚本采用固定时间戳和 ZIP Stored，重复构建相同输入会得到相同字节。不要把 `.git`、`__pycache__`、虚拟环境、打包脚本或生成的 ZIP 放进插件源码目录。更换作者、清单或源码后，若已有同版本输出，应新增版本后重新打包；脚本不会静默覆盖内容不同的版本。

### 6.2 ZIP 约束

ZIP 根目录必须直接包含 `manifest.json` 和 `plugin.py`，不能把整个 `my_counter/` 文件夹再套一层。可添加纯 Python 辅助模块及允许的静态资源。

| 项目 | 限制 |
| --- | --- |
| ZIP 大小 | 不超过 5 MiB。 |
| 展开后总大小 | 不超过 20 MiB。 |
| 单个文件 | 不超过 5 MiB。 |
| ZIP 内清单 | 不超过 64 KiB。 |
| ZIP 条目数 | 最多 128，显式目录条目也计入。 |
| 压缩 | 仅 ZIP Stored / Deflate；每个文件展开大小不超过压缩大小的 100 倍。 |
| 扩展名 | 仅 `.py`、`.json`、`.txt`、`.md`、`.png`、`.jpg`、`.jpeg`、`.webp`。 |
| 路径 | 相对路径，使用 `/`，长度不超过 240 字符、最多 8 段。不得有 `..`、绝对路径、反斜杠、盘符、Windows 保留名、非法字符、末尾空格或点。 |
| 禁止项目 | 符号链接、特殊文件、加密 ZIP、重复路径及大小写冲突、其他目录下的第二份 `manifest.json`、原生库/可执行程序。 |

所有 `.py` 文件按 Python 3.11 语法解析，并检查静态导入只能来自标准库、本包模块或 `flowmedia_plugin`。安装时验证入口类名称，执行时再验证它是否真正继承 SDK 基类。上传校验不等同于实际运行测试或代码安全审核。

### 6.3 已有后端源码时的官方工具

如果开发环境已具有 FlowMedia 后端源码及对应依赖，可从该项目根目录运行：

```bash
python backend/tools/package_plugin.py /path/to/my_counter /path/to/FlowMedia-Plugins/plugins/my_counter/1.0.0/plugin.zip --author YOUR-GITHUB-USERNAME
```

该工具调用主程序完整包校验器，并生成同目录的外层 `manifest.json`；输出路径不能位于源码目录内。省略 `--author` 时只生成供本地上传的 ZIP。它不在本插件仓库中，不能假定克隆本仓库后该路径就存在。

<a id="catalog"></a>

## 7. 市场仓库结构：没有 index.json

市场根据所选分支的 Git 提交目录树发现插件，**不读取根目录 `index.json`**。每个版本用两个文件组成索引记录：

```text
FlowMedia-Plugins/
├── README.md
└── plugins/
    └── my_counter/
        ├── 1.0.0/
        │   ├── manifest.json  # 外层市场描述符
        │   └── plugin.zip     # 内含自己的 manifest.json 与 plugin.py
        └── 1.0.1/
            ├── manifest.json
            └── plugin.zip
```

新增插件或版本时，增加 `plugins/<id>/<version>/` 和两个文件即可。README 顶部的插件列表仅方便阅读，不参与市场发现。`plugins/` 树内不要放 `.gitkeep`、README、源代码目录、额外截图或其他文件；插件说明可放 ZIP 内，也可放仓库中 `plugins/` 以外的位置。

外层 `manifest.json` **恰好包含五个键**：

| 键 | 内容 |
| --- | --- |
| `manifest` | ZIP 内清单经默认值补齐后的完整对象，结构与第 3、4 节一致。不是文件名或 URL。 |
| `sha256` | 对最终 `plugin.zip` 原始字节计算的 SHA-256，小写 64 位十六进制字符串。不是 Git blob SHA。 |
| `size` | 最终 ZIP 文件字节数，JSON 整数。不是展开大小。 |
| `author` | 发布者 GitHub 用户名；1–39 个字符，首尾字母或数字，中间可包含连字符。 |
| `package` | 固定仓库相对路径 `plugins/<id>/<version>/plugin.zip`。 |

第 6 节脚本会生成完整合法示例，避免手工填入不匹配的摘要或大小。可用下面的独立检查命令打印自己版本的真实记录：

```bash
python -m json.tool FlowMedia-Plugins/plugins/my_counter/1.0.0/manifest.json
```

外层描述符、版本目录与 ZIP 内的 ID、版本及标准化清单必须一致。两个市场文件均须是普通 Git 文件，模式为 `100644`，不要设为可执行或使用 Git LFS 指针代替 ZIP。市场在同一个提交上读取描述符和 ZIP，检查 Git blob 摘要、SHA-256、大小以及包内清单后才允许安装。

单个外层描述符不超过 64 KiB；当前读取器接受全仓库最多 128 个插件版本、递归树最多 4096 项、描述符累计不超过 2 MiB。维护者增加版本时应留意这些边界。损坏记录或客户端不支持的 SDK 清单可能导致整个市场读取失败，因此合并前应检查整个 `plugins/` 目录和目标客户端兼容性，而不只检查新增文件。

<a id="contributing"></a>

## 8. 提交插件

### 8.1 在 FlowMedia 中提交 Pull Request

1. 按第 2 节上传 ZIP 到“本地”，完成安装、配置、启用与手动执行验证。
2. 点击插件管理页面顶部“配置”，仓库填写 `treehello0210/FlowMedia-Plugins`，分支填写 `main`，填写自己的“GitHub Token”并保存。
3. 在“本地”中打开目标插件卡片，点击卡片弹窗里的“上传”。这与页面顶部导入 ZIP 的“上传”是不同操作。
4. 提交成功后点击“查看”，进入生成的 GitHub Pull Request；在 PR 中补充功能、测试方法、兼容性和必要的审核说明。
5. 等待维护者审核合并。其他用户在“插件市场”刷新后才会看到并安装该版本。

主程序只创建独立分支和 PR，不直接更新 `main`，也不自动合并。有上游写权限时在上游创建分支；没有写权限时使用自己的同名 fork，必要时尝试创建 fork。若账号下已有同名但不属于本仓库的仓库、fork 尚未准备好或缺少权限，提交会失败。

同版本同内容已有 PR 时会尝试复用；同版本内容冲突应递增版本后重新提交。发布者来自 Token 对应的 GitHub 用户身份。

### 8.2 手工提交 Pull Request

无需把 GitHub Token 交给 FlowMedia，也可以：

1. 在 GitHub 上 fork 本仓库并克隆自己的 fork。
2. 新建分支，例如 `plugin/my_counter/1.0.0`。
3. 将打包生成的两个文件放到 fork 的 `plugins/my_counter/1.0.0/`。
4. 确认没有修改任何已有发布版本的 ZIP 或描述符，再提交并推送该分支。
5. 向 `treehello0210/FlowMedia-Plugins` 的 `main` 发起 Pull Request，写明功能、配置方式、测试结果及新版本变化。

维护者合并前需检查插件源码、清单、实际 ZIP 内容、摘要和目标主程序 SDK 兼容性，而不是只看文件名。说明文档和插件示例即可用于审核，无需提交 FlowMedia 主程序源码。推荐保持已发布版本不可变，并为市场分支设置 PR 审核规则。

### 8.3 GitHub Token 如何获取

**公开仓库的浏览与安装无需 Token。** Token 用于主程序代表你提交 GitHub 分支和 PR，私有仓库访问也需要相应权限。

前往 [GitHub Token 设置](https://github.com/settings/tokens) 申请。GitHub 路径为头像菜单 → Settings → Developer settings → Personal access tokens。详见 [GitHub 官方 Token 管理说明](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)。

- 对公开仓库的经典 Token，可选择 **Generate new token (classic)**，设置用途、有效期，并使用适用于公开仓库的 `public_repo` 范围；复制生成的 Token 到 FlowMedia 的“GitHub Token”输入框。
- 有仓库写入权限的维护者需要目标仓库的内容写入和 Pull Request 写入能力。
- 普通贡献者需要自己的 fork 写入及向上游创建 PR 的权限。细粒度 Token 受资源所有者、仓库授权和 fork/跨仓库限制影响，不能仅授权一个只读上游仓库就认为能够完成整个流程。
- 组织策略、SSO 或 Token 到期仍可能阻止提交；请按 GitHub 官方说明检查授权。

FlowMedia 将 GitHub Token 按登录用户与仓库分别加密保存。配置弹窗默认以密码形式回显，小眼睛可以切换明文；不修改 Token 保存时保留已有值。“清除”后点击“保存”才移除当前仓库的凭据。备份主程序配置时，应一并保留配置数据库和 `.plugin_repository_secret`；不要将这些文件或 Token 提交到市场。

<a id="deployment"></a>

## 9. 部署、更新和持久化

SDK 1 与 SDK 2 的插件代码都不在 FlowMedia 主进程执行。SDK 2 还要求主程序包含相应采集、控制和界面能力；插件 ZIP 本身不会安装或升级这些主程序能力。真正运行插件需要支持 Linux 容器、seccomp、内存/CPU/PID 限额的 Docker 环境，并满足：

- FlowMedia 能访问 `/var/run/docker.sock`。
- 插件根目录通过宿主机 bind mount 或 Docker volume 持久化，例如 `./plugins:/plugins`；运行器需要把已校验代码映射给独立 worker。
- 宿主机已预先拉取 `python:3.11-slim`：`docker pull python:3.11-slim`。运行器不会自行拉取镜像；可通过 `PLUGIN_RUNTIME_IMAGE` 选择经过审核的兼容 Python 3.11 镜像或摘要。
- 保留容器默认 `HOSTNAME` 所提供的实际 Docker 容器 ID；若自定义 hostname，应设置 `FLOWMEDIA_CONTAINER_ID` 为当前 FlowMedia 容器实际 ID。非 root 部署还需具备 Docker socket 访问权限。

运行限制：非 root UID/GID `65534:65534`、只读根文件系统、只读插件代码、禁网、移除全部 capabilities、禁止提权、128 MiB 内存（不增加交换内存额度）、0.5 CPU、32 PID、8 MiB 的 `/tmp`。默认单次 worker 执行期限为 30 秒；SDK 2 的主程序采集、策略处理有各自的期限，不能把 30 秒理解为整个流程的总时长。不提供通过插件清单自行提高限制的字段。缺少隔离条件时显示“未连接”并拒绝启用/运行，本地上传和配置不依赖 GitHub Token。

| 操作 | 状态与数据行为 |
| --- | --- |
| 上传 ZIP | 只进入本地库，不安装、不运行。 |
| 安装 | 建立安装记录，默认停用；同一插件 ID 同时只有一个已安装版本。最多安装 64 个插件。 |
| 保存配置 | 校验并持久化配置；运行中的插件不能保存配置。 |
| 启用 | 校验配置和运行环境、注册定时任务，按声明触发 `plugin.started`。 |
| 停用 | 取消定时任务和正在执行的任务、清理 worker，保留配置和已成功写入的数据。 |
| 更新 | 先停用旧版；保留新 Schema 中仍存在的旧配置字段并校验，补入新默认值，记住旧版本与配置。新版本保持停用，需要手动启用。 |
| 回滚 | 恢复上一个版本及其配置，仍保持停用。只记录一组前后版本，不是完整版本历史。 |
| 卸载 | 删除安装记录；本地 ZIP 和独立插件数据保留。重装可能继续读取原有持久化数据。 |
| 删除本地版本 | 不能删除当前安装版本或仍作为回滚目标的版本。 |
| 重启主程序 | 恢复已保存的启用状态、配置、数据和调度；运行条件满足后触发声明的启用事件。 |

SDK 2 的服务快照只用于本次执行；需要保留的摘要应显式使用 `save_data()`。网络代理探测样本和事件由主程序按插件 ID 另行管理，具体保留范围见[插件说明](docs/network_proxy_monitor.md)。SDK 2 对外连接来自主程序所在环境，Docker 中的 `127.0.0.1` 指向当前容器，应填写主程序实际可达的管理 API 地址。

更新和回滚不会迁移或回滚 `save_data()` 保存的数据。开发者应自行兼容旧数据格式；若新 Schema 与保留的旧配置类型/范围冲突，更新会校验失败。修改配置字段类型时应设计迁移方式，并在 PR 中写清兼容性与操作步骤。

<a id="troubleshooting"></a>

## 10. 发布前检查与常见问题

建议每个版本完成以下验证：

1. 干净源码目录打包成功，同一输入重复打包得到相同 ZIP。
2. ZIP 根目录、清单、类名、同步处理方法和标准库导入均正确。
3. FlowMedia 顶部“上传”成功，“本地”安装后默认停用。
4. 表单默认值、必填、边界值、密码显示及保存后回显符合预期。
5. 启用后手动执行成功；声明了启动或定时任务时，分别验证实际触发。
6. 停用后不再定时运行；重新启用/重启后数据继续保留。
7. 递增版本测试更新、旧配置兼容及回滚；确认持久化数据不会因回滚自动还原。
8. 两个市场文件的路径、字节数、SHA-256 和完整清单一致；PR 不包含凭据、运行数据或主程序实现。
9. SDK 2：确认目标主程序支持权限组合；真实接口、无效凭据、超时、不可达地址和服务能力缺失时的展示与控制行为均有验证。不要将模拟接口测试写成真实服务实测。
10. 市场分支新增 SDK 版本前，检查旧客户端是否仍能读取整个市场；插件目录与 README 中的版本、兼容要求、说明链接保持一致。

| 现象 / 现有提示 | 优先检查 |
| --- | --- |
| 上传时“请求失败”或打包时“参数错误” | ZIP 是否多套目录、清单是否有未知键、版本格式是否正确、是否混入缓存/第三方依赖/二进制、是否超过大小或压缩比例限制。 |
| 同版本上传或提交失败 | 同一版本已有不同 ZIP 字节；增加版本号。 |
| 表单“字段不能为空”或保存“请求失败” | 必填项、默认值、类型、长度、范围或枚举是否匹配。 |
| 状态“未连接”，不能启用 | Docker socket、容器 ID、插件持久化挂载、预拉取镜像及隔离能力。Token 不是运行器连接凭据。 |
| 开始执行后卡片“失败” / 系统“操作失败” | 实际方法是否存在且同步、入口类是否继承 SDK、是否访问网络或禁用资源、是否超时/超限、返回值是否包含字符串或非法结构、收尾回调是否抛出异常。 |
| 看不到 `print()` 或自定义日志 | 属于当前运行器设计；使用固定 `notify()` 模板，不能依赖 stdout/stderr。 |
| `save_data()` 后数据未更新 | 本次执行或清理是否失败；只有成功执行才保存数据。仅修改 `get_data()` 返回副本也不会自动保存。 |
| 插件市场“请求失败” | 先检查主程序是否支持市场中全部清单的 SDK；仅支持 SDK 1 的客户端读取当前 `main` 可能失败。再检查仓库/分支、GitHub 连通性或 API 限流、私有仓库授权、描述符与目录结构。 |
| 提交 PR“请求失败” | 是否配置 GitHub Token、Token 是否过期或权限不足、fork 是否就绪、是否存在同名非 fork 仓库、同版本是否冲突。 |
| 上传完成但市场看不到 | 顶部“上传”仅加入本地库；卡片“上传”提交 PR；维护者合并后，使用兼容的主程序刷新市场。 |
| SDK 2 的 `self.config` 中没有密码字段 | 主程序按 `format: "password"` 剔除这些字段，属于预期行为；服务认证由主程序适配器完成。 |
| 网络代理某些区块或操作不可用 | 核对服务实际版本、接口响应和能力；没有真实数据的区块不会显示。v2rayA 2.2.7.5 禁用可能影响核心运行的探测与切换。 |

这些提示复用 FlowMedia 现有文案，通常不会包含远程响应、插件原始异常或凭据。先缩减到本文最小示例定位，再逐步增加功能；不要把真实 Token 或包含用户数据的运行目录放进问题报告。

项目文档：[FlowMedia Wiki](https://www.flowmedia.wiki/)。插件问题可在本仓库 [Issues](https://github.com/treehello0210/FlowMedia-Plugins/issues) 中说明 FlowMedia 版本、插件 ID/版本、最小复现步骤及不含敏感信息的现象。

<a id="proxy-sdk"></a>

## 11. SDK 2：受控网络代理能力

SDK 2 沿用前文的目录、Python 3.11、清单格式 1、表单 Schema、事件、Cron、数据和日志约束。`permissions` 必须使用[第 3 节](#manifest)的完整组合。需要监控时声明 `proxy.monitor`，需要探测与选择时再声明 `proxy.control`；插件进程在两种组合下都保持禁网。

### 11.1 配置约定

主程序提供的是固定 Clash / v2rayA 适配器。插件的 `config_schema` 需要使用它认识的字段，不能把任意字段映射成新的 HTTP 接口：

| 字段 | 类型与用途 |
| --- | --- |
| `provider` | 字符串枚举 `"Clash"` / `"v2rayA"`。 |
| `clash_url` / `clash_secret` | Clash 管理 API 地址 / secret；secret 声明 `format: "password"`。 |
| `v2raya_url` / `v2raya_token` | v2rayA 管理地址 / 登录令牌；令牌声明密码格式，配置后优先使用。 |
| `v2raya_username` / `v2raya_password` | 未配置令牌时用于登录；密码声明密码格式。 |
| `timeout` | 请求超时，整数 1–10 秒，默认 5 秒。 |
| 策略字段 | `strategy`、`group`、`probe_url`、阈值、间隔、冷却及 `domain_rules` 等，详见[网络代理监控配置说明](docs/network_proxy_monitor.md)。 |

可参考已发布的[完整市场描述符](plugins/network_proxy_monitor/1.0.0/manifest.json)，其中 `manifest` 对象是包内清单的规范化形式。密码、令牌与地址的示例默认值为空，不应填入真实服务信息。监控专用界面由主程序实现，不需要也不允许把前端源码放入插件 ZIP。

### 11.2 服务快照

`get_service_snapshot()` 返回深拷贝；调用不访问网络、不触发刷新或控制动作，也不会修改主程序中的原值。未提供快照时返回 `{}`。主程序提供快照时，`ok`、`provider`、`checked_at` 标识本次结果；下表中的其他字段按成功状态、服务能力和实际响应提供，可能省略：

| 字段 | 含义 |
| --- | --- |
| `ok` | 本次主要管理接口读取是否成功；不等同于全部节点健康或核心已启动。 |
| `provider` / `checked_at` | 本次服务类型 / 采集时间。 |
| `summary` | 实际返回的版本、运行状态及可获得的数量等摘要；字段按服务和响应变化。 |
| `capabilities` | 实际可用的探测、选择和域名探测等能力；缺少能力时不可假定支持。 |
| `nodes`、`groups`、`subscriptions`、`connections`、`traffic` | 仅在取得相应数据时提供；两种代理服务支持范围不同。 |
| `error` / `message` | 失败类别及现有系统提示；类别包括 `unreachable`、`invalid_secret`、`interface_error`、`timeout`、`invalid_config`。 |
| `warnings` / `truncated` | 部分接口失败信息 / 列表被限量截断标记，出现时不能把结果视为完整数据。 |

传入 worker 的服务快照最多 16 KiB，并计入 90 KiB 输入总限制。列表可能截断，不承诺包含所有节点或连接。可用字段中的数值 `0` 是有效数据，不能将未返回的字段补成 `0`。

在已声明正确 SDK、权限与配置的插件中，可以这样消费快照：

```python
from flowmedia_plugin import FlowMediaPlugin


class Plugin(FlowMediaPlugin):
    def run(self, event=None):
        snapshot = self.get_service_snapshot()
        ok = snapshot.get("ok") is True
        self.save_data("status", ok)
        self.save_data("summary", snapshot.get("summary", {}))
        return {"ok": ok}
```

这是处理方法示例，不能仅复制此文件就安装；还需要符合前文规范的 `manifest.json`。快照可能含字符串，而 `run()` 的返回值仍不允许字符串；不要直接 `return snapshot`。SDK 2 也不增加 `get_form()`、`get_state()`、任意网络请求方法或第三方依赖支持。

### 11.3 读取、测试与控制边界

| 操作 | 配置来源与行为 |
| --- | --- |
| 查看监控 / 刷新 | 使用已保存配置；由主程序采集并呈现快照，不触发节点切换。 |
| 测试连接 | 界面可提交当前表单值；只验证连接与认证，不保存配置、不执行切换。 |
| 手动探测 / 选择 | 要求插件已启用、声明控制能力、服务实际支持该操作，使用已保存配置。 |
| 启用事件 / 手动执行 / Cron | 主程序采集快照；声明控制能力时，按已保存策略执行；随后调用隔离 worker 的处理方法。 |

默认 `monitor` 策略在服务支持时记录探测结果，不执行自动切换；`failover`、`optimize` 依照阈值和保护条件工作。`get_service_snapshot()` 本身不能触发策略。v2rayA 旧版不安全的探测路径不启用；没有明确当前成员或缺失必要探测结果时，不推断失败并切换。

SDK 2 的业务界面、请求限制、控制校验和策略存储由主程序提供。本文描述公开接口与使用约定，不要求发布者提交主程序内部实现；插件包和公开文档仍需检查凭据、运行数据和源码范围。
