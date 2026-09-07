# ServerProbe MCP Debug Skill

Claude Code / NarraFork 技能：从 GitHub 下载并部署 [ServerProbe](https://github.com/wcpe/ServerProbe)（Minecraft 服务器运维探针）到真实服务器，通过 MCP（JSON-RPC over HTTP）连接其诊断控制面，用 42 个工具远程调试服务器与插件。

## 安装

```bash
git clone https://github.com/wcpe/serverprobe-mcp-debug.git ~/.claude/skills/serverprobe-mcp-debug
```

## 能力

- 从 GitHub Releases 下载 ServerProbe jar 并部署到 Paper/Folia/BungeeCord/Velocity
- 配置并开启 MCP 诊断控制面（JSON-RPC over HTTP）
- 42 个诊断工具：指标快照、控制台命令、线程转储、死锁检测、插件维度诊断、日志检索、玩家诊断、Arthas 深度诊断（watch/trace/ognl/火焰图/类替换）、GC-JFR
- 内置实测环境坑规避（Windows Java 17、WSL NAT/mirrored、headless JDK、PATH 污染等）

## 适用场景

- 服务器卡顿/掉帧定位（TPS/MSPT/内存/GC/火焰图）
- 排查插件 bug（类名定位→日志堆栈→方法追踪→内部状态验证）
- 玩家相关异常排查
- 远程运维（等效 RCON 但能力更强）

## 版本对应

| Skill 版本 | ServerProbe 版本 | MCP 工具数 |
|---|---|---|
| v0.4.0 | v0.4.0 | 42 |

## 许可证

MIT
