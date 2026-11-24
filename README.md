# ServerProbe Debug Skill

Devin / Claude Code / Codex 三端兼容的 agent 插件：从 GitHub 下载并部署 [ServerProbe](https://github.com/wcpe/ServerProbe)（Minecraft 服务器运维探针）到真实服务器，通过 MCP（JSON-RPC over HTTP）连接其诊断控制面，用 42 个工具远程调试服务器与插件。

## 仓库布局

```
plugin.json                  # Agent Plugins 通用清单（Devin / Codex portable 均识别）
.claude-plugin/plugin.json   # Claude Code 插件清单
.codex-plugin/plugin.json    # Codex 兼容清单（fallback）
skills/serverprobe-debug-skill/
  SKILL.md                   # 技能正文
  references/                # Arthas 深度诊断参考
  evals/                     # 评测集
```

## 安装

**Devin**：Settings → Plugins 安装本仓库（识别根 `plugin.json`），或在会话中让 Devin 执行 `manage_plugin` 安装 `wcpe/serverprobe-debug-skill`。

**Claude Code**：

```bash
# 方式一：作为插件装入 marketplace
claude plugin install wcpe/serverprobe-debug-skill
# 方式二：直接把技能目录放到 skills 路径
git clone https://github.com/wcpe/serverprobe-debug-skill.git /tmp/spds \
  && cp -r /tmp/spds/skills/serverprobe-debug-skill ~/.claude/skills/
```

**Codex**：插件指向本仓库即可，根 `plugin.json`（Agent Plugins 格式）声明 `skills: "./skills/"`；兼容清单 `.codex-plugin/plugin.json` 同时提供。

## 能力

- 从 GitHub Releases 下载 ServerProbe jar 并部署到 Paper/Folia/BungeeCord/Velocity
- 配置并开启 MCP 诊断控制面（JSON-RPC over HTTP，`POST :9942/mcp`）
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
