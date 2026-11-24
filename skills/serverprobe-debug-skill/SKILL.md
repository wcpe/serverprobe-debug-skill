---
name: serverprobe-debug-skill
description: 从 GitHub 下载并部署 ServerProbe（Minecraft 服务器运维探针）到真实 Paper/Folia/BungeeCord/Velocity 服务器，通过 MCP（JSON-RPC over HTTP）连接其诊断控制面，用 42 个工具调试服务器与插件：读指标、执行命令、线程转储、日志检索、玩家诊断、Arthas 深度诊断（watch/trace/ognl/火焰图/类替换）。只要用户提到"装 ServerProbe/探针""调试 MC 服务器""看 TPS/内存/GC""MCP 连服务器诊断""下载部署 ServerProbe""排查插件卡顿或报错"，或想要通过 MCP 远程诊断 Minecraft 服务器，即使没说"ServerProbe"这个名字，也应用本技能。
---

# ServerProbe 调试技能

ServerProbe 是装在 Minecraft 服务器里的运维探针插件（单 jar 多端：Paper/Folia/Spigot 1.8–1.21.x、BungeeCord、Velocity 3.1.1–4.x）。它的核心价值：**开启 MCP 控制面后，外部 agent（你）可以不经人工操作，直接远程完成"读指标 → 定位 → 深度诊断"的全链路**。

本技能覆盖完整链路：下载 jar → 部署配置 → 启动服务器 → MCP 对接 → 按场景调用工具。

## 1. 下载

从 GitHub Releases 拿最新版 jar（单文件，约 40MB，内嵌 Arthas 闭包与原生库）：

```bash
# 查最新版本号
curl -s https://api.github.com/repos/wcpe/ServerProbe/releases/latest | grep tag_name
# 下载（替换 <version>）
curl -sL -o ServerProbe-<version>.jar \
  "https://github.com/wcpe/ServerProbe/releases/download/v<version>/ServerProbe-<version>.jar"
```

本地开发版构建：`./gradlew build`，产物在 `plugin/build/libs/ServerProbe-<version>.jar`。

## 2. 部署与配置

1. jar 放入服务器的 `plugins/` 目录，重启服务器；首次启动生成 `plugins/ServerProbe/config.yml`。
2. **要启用 MCP 调试，必须改配置**（默认关闭）。在 `config.yml` 中确认或追加：

```yaml
mcp:
  enabled: true
  host: "127.0.0.1"   # 本机调试用回环；跨机改 "0.0.0.0" 并务必设 secret
  port: 9942
  secret: ""          # 生产环境务必设 Bearer 密钥；空密钥启动时会打中文 WARN
```

改完**重启服务器**生效（`mcp` 段首次生成时 enabled 就是 false，需要手动改）。

3. 验证启动成功：日志出现 `MCP 控制面已启动，监听 127.0.0.1:9942/mcp`。

## 3. MCP 对接（JSON-RPC over HTTP）

端点：`POST http://<host>:9942/mcp`，协议 JSON-RPC 2.0，Bearer 鉴权（secret 非空时带头 `Authorization: Bearer <secret>`）。

**三次调用完成握手与探查**（curl 示例）：

```bash
MCP="http://127.0.0.1:9942/mcp"
# 1. 初始化
curl -s -X POST $MCP -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}'
# 2. 列工具（42 个，返回 name/description/inputSchema）
curl -s -X POST $MCP -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}'
# 3. 调工具（结果在 result.content[0].text，是 JSON 字符串需二次解析）
curl -s -X POST $MCP -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"server_status","arguments":{}}}'
```

**响应解析要点**：`tools/call` 的结果包了两层——`result.content[0].text` 里面才是工具真实输出的 JSON（含 `available`/`reason` 降级字段），解析时先取 text 再 json.loads。命令执行失败不是 JSON-RPC error 就是工具返回 `{"available": false, "reason": "..."}`，两者都要当降级处理而非重试。

**工具分五组**（每个工具的 `description` 含参数/示例/工作流，`tools/list` 里可直接读）：

| 组 | 工具 | 用途 |
|---|---|---|
| 服务器 | `server_status` / `server_command` / `diagnostic_bundle` | 指标快照、执行控制台命令（不带斜杠）、汇总诊断证据 |
| 线程 | `thread_top` / `thread_dump` / `thread_deadlocks` | CPU 排名、全线程栈、死锁检测 |
| 插件诊断 | `plugin_list` / `plugin_classes` / `plugin_threads` / `plugin_cpu` | 插件清单、按插件查类/线程/CPU（不手敲类名） |
| 日志/玩家 | `log_tail` / `log_search` / `player_lookup` | 日志尾部/关键字检索（游标分页）、在线玩家明细 |
| Arthas 深度 | `arthas_execute` / `arthas_watch` / `arthas_trace` / `arthas_ognl` / `flamegraph_start|stop|view` / `gc_events|gc_stats` / `jfr_start|stop` / `artifact_*` / `arthas_task_*` | 深度 JVM 诊断（见 §5 工作流） |

## 4. 推荐调试工作流（按场景）

**场景 A：服务器卡顿/掉帧**
1. `server_status` 看 TPS/MSPT/内存/CPU 总览
2. `thread_top` 找 CPU 最高的线程 → `thread_dump` 看它卡在哪
3. `gc_stats` 看是不是 GC 压力 → `gc_events` 看频率
4. 深挖：`flamegraph_start` → 等负载复现 → `flamegraph_stop` → `flamegraph_view` 取回 HTML 火焰图

**场景 B：排查某个插件的 bug**
1. `plugin_list` 确认插件已加载、`plugin_classes` 拿到它的类名（不用猜包名）
2. `log_search {"keyword":"<插件名>"}` 找报错堆栈 → 拿到出错的类和方法
3. `plugin_threads {"plugin":"<名>"}` 看它是否卡线程
4. 深度探查（类是否加载/反编译源码/方法观测/读内部状态/热修复）→ **详见 [references/deep-diagnosis.md](references/deep-diagnosis.md)**，快速索引：
   - 类是否加载+来自哪个 jar：`arthas_execute {"command":"sc -d <类名>"}`
   - 反编译核验源码：`arthas_execute {"command":"jad <类名> <方法名>"}`
   - 观察方法参数/返回值/异常：`arthas_watch {"className":"<类>","methodName":"<方法>","expression":"{params,returnObj,throwExp}"}`
   - 调用链耗时（慢在哪一行）：`arthas_trace`；调用来源：`arthas_execute {"command":"stack <类> <方法>"}`
   - 读内部状态（静态字段/实例）：`arthas_ognl {"expression":"@<类>@<字段>"}` 或 `arthas_execute {"command":"vmtool --action getInstances --className <类> -x 2"}`
   - 热修复闭环（jad→mc 编译→redefine→revert）：见参考文件 §5
5. 完整实例剧本（"玩家用某功能报 NPE"从堆栈到根因的逐命令序列）也在参考文件 §7

**场景 C：玩家相关异常**
1. `player_lookup {"name":"<玩家名>"}` 拿位置/状态/背包摘要/最近事件链路
2. 配合 `log_search` 按玩家名检索

**异步工具的固定套路**：`arthas_*` 和 `jfr_*` 是异步的——调用返回 `{taskId, state}` → 轮询 `arthas_task_status {"taskId"}` 直到 SUCCEEDED → `arthas_task_output {"taskId","offset":0}` 分片读输出（`nextOffset` 翻页）。先触发问题场景再挂 watch/trace 才能抓到。

**危险操作边界**：`arthas_redefine`/`arthas_retransform` 会热替换生产类字节码（自动备份到 `backup_` 前缀工件，可 `artifact_backup_restore` 恢复）；`server_command` 等同控制台权限。只在明确授权的调试场景使用。

## 5. 环境坑（实测踩过，直接规避）

**Windows**：
- 服务器需 Java 17+；系统默认 Java 8 时显式用 JDK17 全路径启动，如 `"C:/Users/<用户>/.jdks/<jdk17>/bin/java.exe" -jar paper.jar`（或 Linux/WSL 侧 `/usr/lib/jvm/<jdk17>/bin/java`）
- async-profiler **不支持 Windows**：`flamegraph_start` 会返回 "Only support Linux/Mac"，属平台限制非故障；Windows 上做 CPU 分析改用 `arthas_execute {"command":"profiler start"}` 也一样不行，需 Linux 环境或 spark 插件

**WSL（Linux）**：
- **网络模式**：默认 NAT 下 WSL 内 127.0.0.1 互通正常；**mirrored 模式（.wslconfig 配了 networkingMode=mirrored）会劫持 loopback 导致 MCP 不可达**——遇到连接超时先查这个，回退 NAT
- **Temurin JDK 是 headless**：Paper 启动报 `libXtst.so.6: cannot open shared object file`——`apt-get download libxtst6 libxi6 libxrender1 libfontconfig1` 后 `dpkg -x` 解包，把 `.so` 放到自定义目录并以 `LD_LIBRARY_PATH=<目录>` 启动（免 root）
- **后台启动**：`wsl -d <发行版> -- bash -c "..."` 的 nohup 子进程会随会话退出被杀；用 Bash 工具的 run_in_background 前台驻留最可靠
- **PATH 污染**：从 Windows 传给 WSL 的命令含 Unix 路径会被 Git Bash 改写，**一律加 `MSYS_NO_PATHCONV=1` 前缀**
- **Git Bash 转义**：`wsl -- bash -c "... \$VAR ..."` 的转义极易错——复杂命令写成脚本文件用 stdin 传入（`cat > /tmp/x.sh < local.sh`）再执行
- 杀服务器进程别依赖 grep 提取 PID（转义易错），先 `ss -tlnp | grep 9942` 人读出 pid 再 `kill -9 <pid>`

**通用**：
- MCP 起不来先看启动日志有无 `MCP 控制面已启动`；监听了但不响应，查端口占用（`ss -tlnp | grep 9942`）与双实例冲突
- `logs/latest.log` 是定位一切的第一现场；MCP 工具降级时先读它
- 大日志（>64MiB）`log_tail` 的行号可能非精确（响应 `lineNumbersExact:false`），此时用 `log_search` 的字节偏移游标
