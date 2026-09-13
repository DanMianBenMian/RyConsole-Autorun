---
name: ryconsole-autorun
description: >-
  通过局域网把 JS 代码送到 iPad 上的 RyConsole 执行，并取回结果（AI 在电脑端，iPad 只负责「执行 + 回传」）。
  当需要在真机 iPad 上跑 iOS 探针 / IOKit 调用 / 沙盒能力测试 / 执行任意 JS 并拿到输出与返回值时使用。
  典型触发：「在 iPad 上跑这段 JS」「用 RyConsole 执行」「真机验证这个探针」「把结果传回来看」。
  内置轮询契约：提交后每 3 秒查一次结果、最多 12 次；并含全部实踩坑点（连不上的四要素、
  面板必须先启动、App 必须前台、print 捕获语义、轮询未完成的判定与处置）。
agent_created: true
---

# RyConsole Autorun —— 用 iPad 当"执行手"

## 核心心智模型

```
AI（电脑端 / Ryan）  ── 写 JS ──▶  RyConsole（iPad，Autorun 服务）  ── 执行 ──▶  回传 JSON
        ▲                                                                          │
        └────────────────────────── 结果（output / result / error） ◀───────────────┘
```

**AI 不在设备上，也不需要在设备上**。iPad 只是一个「能碰底层接口的执行环境」，
所有判断、迭代、下一步决策都在电脑端（也就是你）这边做。
这绕开了端上算力的一切限制：不需要在 iPad 上跑模型。

## 前置条件（缺一不可，先检查再动手）

| 条件 | 怎么确认 |
|---|---|
| iPad 上 Autorun 已启动 | 应用内打开面板点「启动」，或在控制台执行 `autorunStart(8899)`；面板顶部会显示本机 IP |
| iPad 与电脑同一局域网 | 电脑 `ping <iPadIP>` 能通 |
| **RyConsole 保持在前台** | iOS 后台会挂起监听，锁屏/切后台都会断 |
| 端口一致 | 默认 `8899`（面板里可改） |

**先跑 `ping` 确认，再谈执行。** ping 不通时不要盲目重试，按下面报错表处理。

## 工具位置（路径不写死 —— 见下方「调用第一步」）

```
tools\ryc_autorun.py        # ★ Agent 用这个（CLI，无第三方依赖；已可打包为单 exe ryc_autorun.exe）
tools\ryc_gui.py            # 电脑端图形界面（人看结果用；也可给 Agent 当控制接口）
tools\RunAutorunPanel.bat   # 双击启动 GUI（纯 ASCII，内含 %USERPROFILE% 运行时解析）
tools\logs\autorun_gui.jsonl# GUI 每轮交互追加落盘（Agent 直接读文件也能取结果）
```

> ⚠️ **软件位置不写死**：本 skill 不假设 `ryc_autorun.exe/.py` 的绝对路径（项目随工作区目录如带时间戳的 `2026-07-22-11-14-51` 移动 / 被清理后写死会断）。

**调用第一步（必做）**：AI 加载本 skill 后，**先向用户询问 `ryc_autorun` 可执行文件的实际位置**
（`ryc_autorun.exe` 或 `ryc_autorun.py` 的绝对路径 / 所在目录）。
- 若本对话中用户已给过路径或 IP，直接采用，不必重复问。
- 拿到后记作 `<RYC_EXE>`；建议先跑 `<RYC_EXE> --help` 验证文件存在且可执行。
- 解释器：纯标准库，任意 Python 3.8+ 均可；若用 `.exe` 则无需 Python 环境。

## 命令速查（`<RYC_EXE>` = 上方询问用户得到的可执行文件位置）

```bash
# 0) 连通性（永远先做这一步）
<RYC_EXE> ping --host <iPadIP> --port 8899

# 1) ★ 标准动作：提交 + 自动轮询（每 3s，最多 12 次）
<RYC_EXE> run --host <iPadIP> --code-file <探针.js 绝对路径> --json --quiet

# 2) 同步执行（代码很快时用；会占住连接）
<RYC_EXE> eval --host <iPadIP> --code 'return 1+1'

# 3) 手动两步（需要自己控制轮询节奏时）
<RYC_EXE> submit --host <iPadIP> --code-file <探针.js 绝对路径> --json
<RYC_EXE> result --host <iPadIP> --id j1 --json
```

> 探针 `.js` 文件用绝对路径（或相对 `<RYC_EXE>` 所在目录）；不要依赖当前工作目录。

**默认就用 `run`**：它已经内置了「提交 → 每 3 秒查一次 → 最多 12 次」的契约（≈36 秒上限，足够跑完并回传）。
不要自己写 sleep 循环去轮询 —— 用脚本里的 `run`，避免节奏不一致。

## 电脑端图形界面（给人看；也给 Agent 当第二入口）

```bash
# 双击启动
tools\RunAutorunPanel.bat
# 或命令行（可预填目标；路径同 <RYC_EXE> 所在目录，记作 <RYC_EXE_DIR>）
<RYC_EXE_DIR>\ryc_gui.py --host 192.168.0.110 --port 8899
python ryc_gui.py --no-control          # 不启动 AI 控制接口
python ryc_gui.py --selftest --port 18899   # 无界面自测（需 mock 服务）
```

界面（深色科技风，与 iPad 端同一套配色）：顶栏 `iPad IP / 端口 / 连接`＋状态灯；
中部「发送到 iPad 的代码」编辑区（载入 .js / 另存 .js / 清空 / 执行，`Ctrl+Enter` 也可执行）；
下部「执行结果 / 回传内容」（状态 pill：待命→执行中→已回传/未回传，含往返毫秒、print 段、返回值段、错误段）；
底部「事件日志」逐次打印轮询进度。

### 内置 AI 控制接口（默认开，仅绑 127.0.0.1:8877）

GUI 在跑时，Agent 可以**不经过 CLI**直接驱动它 —— 好处是用户能实时看到"AI 正在发什么代码、回传了什么"：

```bash
GET  http://127.0.0.1:8877/status          # 目标/连接/忙闲/最近结果摘要
GET  http://127.0.0.1:8877/ping
POST http://127.0.0.1:8877/exec  {"code":"...", "host":"...", "port":8899}
GET  http://127.0.0.1:8877/result?id=j7
GET  http://127.0.0.1:8877/last            # 最近一次完整结果
GET  http://127.0.0.1:8877/log?n=80        # 事件日志文本
GET  http://127.0.0.1:8877/history?n=20    # 历史（不含代码，防爆量）
POST http://127.0.0.1:8877/stop
```

`/exec` 是**阻塞式**的：内部走完「submit → 3s 轮询 ×12」才返回最终 JSON（结构与 CLI 的 `run --json` 一致）。
Agent 发来的代码会自动回显到 GUI 编辑区，用户看得见。

**两条路怎么选**：GUI 没开 → 用 CLI；GUI 开着且希望用户旁观 → 走控制接口。
无论哪条，结果都会追加进 `logs\autorun_gui.jsonl`（GUI 路径）。

## 输出怎么读

```json
{
  "id": "j7",
  "status": "done",        // running / done
  "ok": true,               // 代码是否无异常
  "output": "print 的全部输出（多行）",
  "result": "代码 return 的值（已转成字符串）",
  "error": null,            // 有异常时是异常文本（含堆栈）
  "ms": 1234                // 执行耗时
}
```

- `ok:false` 且 `error` 有内容 → **代码本身报错了**（不是通信问题），读 `error` 改代码。
- `error` 里出现 `[JS Error]` 前缀 → 该行是 JS 异常，可能与 `output` 交错，注意区分。
- `status:"running"` 跑满 12 次 → 说明代码还没跑完（或卡住）。**不要无脑重试**：先让用户看 iPad 屏幕，
  确认 App 是否还在前台、是否弹了系统弹窗（如局域网权限），必要时把代码拆小分步跑。

## 报错处置表

| 现象 | 原因 | 处置 |
|---|---|---|
| `连不上 ... WinError 10060` / 超时 | 服务没启动 / 不同网段 / App 在后台 | 让用户①开面板点启动 ②确认同一 Wi-Fi ③把 RyConsole 切到前台 |
| `连不上 ... 10061 拒绝` | 端口不对或服务已停 | 核对端口；面板里看「● 正在监听」 |
| `HTTP 404 no such route` | 端点拼错 | 只用 `/ping` `/eval` `/submit` `/result` |
| `执行器未注册` | 原生侧 `RYAutorunSetEval` 没跑到 | 让用户先在控制台跑任意一段 JS（会触发注册），再启动 Autorun |
| 返回不是 JSON | 服务被别的程序占用端口 | 换端口（面板改 + `--port` 同步改） |
| `status:running` 满 12 次 | 代码卡住 / 主线程被占 / 弹窗未确认 | 看设备屏幕；把代码拆成小步；避免死循环 |

## 写探针代码的约定

1. **`print()` 就是回传通道**：所有中间信息用 `print(...)` 输出，会收在 `output` 里。
2. **`return` 是结果通道**：最后 `return` 的对象/字符串进 `result`。
3. 代码里可以直接调用**已注册的原生原语**（52 个，见下节「原生桥清单」）——
   不要凭猜测写 API，**先用清单核对**再写；清单里没有的接口就是没有。
4. **把结果做成 JSON 字符串再 return**，比返回裸对象更稳（可读、可解析）。
5. **单步优先**：一轮一件事。跑得越久越容易触发超时和 panic，也更难定位。
6. **返回值尽量小**：不要 return 巨大的结构，超长输出用 `print` 分段。

### 推荐模板

```js
// 探针模板：结果统一用 JSON 回传
var out = { goal: "探测 XXX", hits: [], errors: [] };
try {
  var conn = iokitOpen("IOSurfaceRoot", 0);
  out.conn = conn;
  if (conn > 0) {
    var r = iokitCall(conn, 0, []);
    out.selector0 = r;
    print("selector 0 -> " + r);
    iokitClose(conn);
  }
} catch (e) {
  out.errors.push(String(e));
}
return JSON.stringify(out);
```

## 原生桥清单（写在探针前先核对这张表）

**全部 52 个原语**都已注册为全局函数。每次执行都是**全新 JSContext**（跨轮次变量不保留 → 要状态就落盘）。
安全分级：🟢只读 / 🟡有副作用 / 🔴高危（人工确认）。

### IOKit 内核接口（核心）🟢
| 原语 | 签名 | 说明 |
|---|---|---|
| `iokitEnum` | `() → string` | 枚举 IORegistry 服务（JSON） |
| `iokitOpen` | `(cls, type) → number` | 打开 IOService；**>0 成功** |
| `iokitCall` | `(conn, sel, inputs[]) → string` | 标量入参调用 |
| `iokitCallStruct` | `(conn, sel, scalarInputs[], structB64, outSize) → string` | 带 inputStruct / OOL 输出 |
| `iokitMap` | `(conn, mapType) → string` | 内存映射 |
| `iokitClose` | `(conn) → number` | 关闭 |
| `bytesB64` / `b64Bytes` | `(arr)/ (b64)` | 字节 ↔ base64 |

### IOSurface 🟡
`iosurfaceProbe()`、`iosurfaceCreate(props)`、`iosurfaceGetID(h)`、`iosurfaceSetValue(h,k,v)`、
`iosurfaceGetValue(h,k)`、`iosurfaceLookup(cid)`、`iosurfaceRelease(h)`

### XPC / 服务 🟡
`xpcLookup(svc)→"REACHABLE"/"NOT_REACHABLE"`、`xpcProbe(svc)`、`xpcSend(svc, selector, payloadJSON)`、`nsxpcProbe(svc)`

### 网络 🟡
| 原语 | 签名 | 说明 |
|---|---|---|
| **`httpRequest`** | `(opts) → JSON` | **同步 HTTP(S)**：`{url,method,headers,body,timeout}` → `{ok,status,body,len,error}`。调 AI 后端/局域网服务首选 |
| `tcpConnect` | `(host, port) → "OPEN"/…` | 单端口（0.3s 超时） |
| `tcpScan` | `(host, start, end)` | 端口段扫描 |
| `tcpSend` | `(host, port, hexPayload)` | 原始 TCP（payload 是 hex） |
| `canOpenURL` / `openURL` / `installIPA`🔴 | | scheme 可达性 / 打开 URL / itms-services 安装 |

### 设备与系统 🟢（`runCommand` 是 🔴）
`appInfo()`、`getPid()`、`getEntitlements()`、`sysctlGet(name)`、`getPasteboard()`、`setPasteboard(s)`、`runCommand(cmd)`🔴

### 文件（相对路径基于 Documents；`/` 开头为绝对路径）🟡
| 原语 | 说明 |
|---|---|
| `readFile(path)` | 失败返回 `[readFile error] …` |
| `writeFile(path, content)` | **整文件覆盖** |
| **`fileAppend(path, content)`** | **追加**——跨轮次 checkpoint 唯一手段 |
| `fileExists(path)` / `fileSize(path)` / `fileDelete(path)` | 存在性 / 字节数 / 删除 |
| `listDir(path)` / `mkdir(name)` | 目录列表 / 建目录 |

### 时间与控制
| 原语 | 说明 |
|---|---|
| **`now()`** | → `{wall, mono}`；测耗时用 `mono`（单调、不跳变） |
| **`sleep(ms)`** | 阻塞休眠（上限 120s）。**限速/间隔专用**（JSC 没有 setTimeout） |
| `print(...)` | 回传通道 |
| `require("bridge_x.js")` | 动态加载桥库（改 JS 不用重编 IPA） |

### 桥库函数（`RyConsoleScripts/*.js`，随 bundle 加载）
- `bridge_iokit.js`：`jbIokitEnum()`、`jbProbeIOKit(...)`、`jbIokitVerdict(...)`、`jbV3Battery()`、`jbV3Report()`、`jbV3MethodCall(...)`
- `bridge_info.js`：`dumpIdentity()`、`dumpEntitlements()`
- `bridge_net.js`：`scanCommonPorts(host)`、`httpGet(host,port,path)`、`hexEncode(str)`
- `bridge_ui.js`：`enumSchemes()` ｜ `jbV3StructProbe.js`：`zeroStructB64(n)`、`krStr(code)`、`log(...)`

### ★ 必带：IOReturn 错误码字典（回喂给 AI 时用）

```js
var IORETURN = {
  "0": "kIOReturnSuccess",
  "-536870174": "kIOReturnNotPermitted",   // 0xe00002e2 门禁 -> 别重试，换策略
  "-536870163": "kIOReturnUnsupported",    // 0xe00002bc selector 不存在 -> 换 selector
  "-536870206": "kIOReturnBadArgument",    // 0xe00002c7 入参不对
  "-536870203": "kIOReturnNotOpen",
  "-536870195": "kIOReturnNotFound",
  "-536870201": "kIOReturnNoResources",
  "-536870212": "kIOReturnNotPrivileged",  // 缺 entitlement
  "-536870199": "kIOReturnExclusiveAccess",
  "-536870211": "kIOReturnTimeout"
};
function kr(c) {
  var n = (typeof c === "string" && /^0x/i.test(c)) ? parseInt(c, 16) : Number(c);
  return IORETURN[String(n)] || ("unknown(" + c + ")");
}
```

### 没有的能力（别浪费时间找）
`setTimeout/setInterval`（用 `sleep`）、跨执行保留变量（落盘）、进程列表（`sysctlGet`+`iokitEnum`）、
Keychain、ObjC 运行时 / dlopen、后台常驻监听（iOS 限制，App 必须前台）。

## 安全红线（不可由被执行的代码自行决定）

1. **只对自有设备**运行；不碰他人设备。
2. **只读优先**：默认只发查询类调用；写/擦除类 selector 需人工确认。
3. **一轮一事、留间隔**：连续失败就停，不要猛跑。
4. **危险面留人工**：AI 负责「提候选 + 跑 + 读结果」，破坏性的那一步由人决定。
5. **先确认设备上无不可替代数据**（这台 iPad 是唯一研究设备时尤其重要）。
6. 发现漏洞走**负责任披露**，不武器化。

## 历史与依赖

- 服务实现：`RyConsole/PayloadApp/AutorunServer.{h,m}`（原生，监听 + 极简 HTTP + 任务队列 + 面板）
- **原生桥完整参考：`RyConsole/PRIMITIVES.md`**（52 个原语的签名/返回值/安全分级/用法示例，比本技能更详细）
- 使用说明：`RyConsole/AUTORUN.md` ｜ 协议与后端设计：`RyConsole/AI_BACKEND.md`
- 安全条款：`RyConsole/DISCLAIMER.md`
- 改了原生代码（`main.m` 的 `installPrimitives` / `AutorunServer.m`）需重编 IPA；只改 JS 探针无需重编。
