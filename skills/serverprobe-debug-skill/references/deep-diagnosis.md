# 深度诊断参考：插件 bug / 类加载 / 源码核验 / 方法级探查

本文件是 SKILL.md §4 工作流 B（排查插件 bug）的**逐命令展开**。所有 Arthas 命令经 MCP 调用：
- 结构化工具（推荐）：`arthas_watch`/`arthas_trace`/`arthas_ognl` 等，参数已封装
- 任意命令兜底：`arthas_execute {"command":"<命令>","timeoutMillis":15000}`

**异步套路全部相同**：调用返回 `{taskId}` → 轮询 `arthas_task_status` 到 SUCCEEDED → `arthas_task_output` 读输出（`nextOffset` 翻页）。

---

## 0. 前置：拿到目标插件的类名

```json
plugin_list {}                                    // 确认插件已加载（enabled=true）
plugin_classes {"plugin":"<插件名>"}               // 枚举它的类清单（有界 200）
log_search {"keyword":"<插件名或异常关键字>"}        // 从报错堆栈拿精确类名+方法
```

报错堆栈里的 `at com.foo.Bar.baz(Bar.java:42)` 就是你要深挖的类+方法。

## 1. 类是否被加载、从哪个 jar 加载（sc -d）

排查"代码明明改了没生效""类找不到/重复类冲突"，先确认 **JVM 里实际加载的是哪个类、来自哪个 jar、哪个 ClassLoader**：

```json
arthas_execute {"command":"sc -d com.example.plugin.Foo","timeoutMillis":10000}
```

输出关键字段：
- `class-info` / `code-source`：**类来自哪个 jar 文件**（确认服务器上跑的是不是你预期的版本）
- `class-loader`：ClassLoader 哈希与类型（插件隔离加载器）
- `possible-reload`：是否被 redefine/retransform 过

**典型结论**：
- `code-source` 指向旧版本 jar → 部署问题，不是代码问题
- 同一 FQCN 出现两个（不同 ClassLoader）→ 插件重复加载/冲突
- `sc` 找不到 → 类未加载（懒加载：只有被引用过才加载；可用 `plugin_classes` 先确认 jar 里有它）

## 2. 反编译核验"源码对不对"（jad）

确认 JVM 里实际运行的代码与你预期的源码一致（比 grep jar 可靠）：

```json
arthas_execute {"command":"jad com.example.plugin.Foo","timeoutMillis":15000}
// 只看某个方法：
arthas_execute {"command":"jad com.example.plugin.Foo bar","timeoutMillis":15000}
```

- 输出是**反编译后的 Java 源码**（含合成/桥接方法），直接读逻辑
- 用途：① 核对线上行为与源码版本；② 反编译确认后可作为热修复基线（见 §5）
- **注意事项**：输出可能很长，用 `arthas_task_output` 的 `nextOffset` 翻页读全；Kotlin/Scala 插件反编译出来的是等价 Java 而非原语言

## 3. 读静态字段与实例状态（getstatic / ognl / vmtool）

**静态字段**（配置单例、缓存、注册表等）：

```json
arthas_execute {"command":"getstatic com.example.plugin.Foo CACHE","timeoutMillis":10000}
// 或经 OGNL（更灵活，可链式取属性）：
arthas_ognl {"expression":"@com.example.plugin.Foo@CACHE.size()"}
```

**拿实例再读字段**：

```json
arthas_ognl {"expression":"#cls=@java.lang.Class@forName('com.example.plugin.Foo'), #insts=@java.util.Collections@newArrayList(@com.example.plugin.Foo@ALL_INSTANCES)"}
```

> **⚠️ vmtool 不可用**：`vmtool --action getInstances` 依赖 Arthas JNI 库（`ArthasJniLibrary`），本插件的内存 Shell 闭包未包含该库，调用会报 `no ArthasJniLibrary in java.library.path`（WSL 真机实测）。**读实例状态的可用路径**：
> 1. **ognl 静态入口链式取**：插件把实例挂在静态字段/注册表时（最常见），`@com.example.plugin.Foo@INSTANCES.get('key').someField`
> 2. **watch 时机读**：`watch <类> <方法> '{target, target.someField}' -x 3`——方法执行时 `target` 就是实例
> 3. **tt 回放**：`tt -t` 录制后 `tt -w -i <index> '{target}'` 读当时实例
>
> **实战例子**：插件说"已把玩家加入冷却表"，验证一下：
>
> ```json
> arthas_ognl {"expression":"@com.example.plugin.Cooldowns@MAP.keySet().size()"}
> arthas_ognl {"expression":"@com.example.plugin.Cooldowns@MAP.get('Steve')"}
> ```

- `-x 2` 展开深度（1=对象摘要，2-3 能看到字段值；太大输出会爆）
- 配合 ognl 对实例求值：`vmtool --action getInstances --className com.example.plugin.Foo --expressions 'instances[0].someField'`

**实战例子**：插件说"已把玩家加入冷却表"，验证一下：

```json
arthas_ognl {"expression":"@com.example.plugin.Cooldowns@MAP.keySet().size()"}
arthas_ognl {"expression":"@com.example.plugin.Cooldowns@MAP.get('Steve')"}
```

## 4. 方法级观测（watch / trace / stack / tt）

### 4.1 watch：看参数、返回值、异常（最常用）

```json
arthas_watch {"className":"com.example.plugin.Foo","methodName":"bar","expression":"{params,returnObj,throwExp}","maxMatches":5,"timeoutMillis":30000}
```

- `expression` 是 OGNL：`{params,returnObj,throwExp}` 一次性拿三样
- **-x 展开深度**：结构化工具默认深度可能不够，深对象用 `arthas_execute` 加 `-x 3`：
  `{"command":"watch com.example.plugin.Foo bar '{params,returnObj}' -x 3 -n 5"}`
- **条件过滤**（只在特定入参时观测，避免刷屏）：
  `watch com.example.plugin.Foo bar '{params,returnObj}' 'params[0]=="Steve"'`
- 常用表达式速查：`{params, returnObj, throwExp, #cost}`（#cost 是方法耗时毫秒）
- **挂上后要触发方法执行**（玩家操作/命令/等场景重演），然后 `arthas_task_status` 看是否抓到

### 4.2 trace：方法内部调用链耗时（定位慢在哪一行）

```json
arthas_trace {"className":"com.example.plugin.Foo","methodName":"bar","maxMatches":3}
// 只追踪慢调用（>100ms 才输出）：
arthas_execute {"command":"trace com.example.plugin.Foo bar '#cost > 100' -n 3"}
```

输出是**调用树**，每层带耗时——直接定位"插件方法慢是因为调了 DB 还是算了大东西"。

### 4.3 stack：这个方法被谁调用（追调用来源）

```json
arthas_execute {"command":"stack com.example.plugin.Foo bar -n 3"}
```

排查"这段代码为什么被执行/被谁意外触发"。

### 4.4 tt：时间隧道（录制后离线回放，抓"过去"的错误）

方法已经跑完、现场没了时用 tt：

```json
arthas_execute {"command":"tt -t com.example.plugin.Foo bar -n 10"}
arthas_execute {"command":"tt -l"}                                  // 列录制条目
arthas_execute {"command":"tt -p -i 1000"}                          // 回放第 1000 条
arthas_execute {"command":"tt -w -i 1000 '{params,returnObj}'"}     // 查看某条内容
```

## 5. 热修复闭环（jad → 改 → 编译 → redefine → 验证 → revert）

> **高危**：直接改生产字节码。每次 redefine 前本插件会自动备份原类（`backup_` 前缀工件），可用于 `artifact_backup_restore` 恢复。仅限明确授权的调试场景。

1. **导出当前源码**：
   `arthas_execute {"command":"jad --source-only com.example.plugin.Foo > /tmp/Foo.java"}`
2. **取回**：`artifact_read_chunk`（若在 mcp-workspace）或经 `arthas_task_output` 读输出；本地修改
3. **编译**（二选一）：
   - 服务器内存编译：`mc -c <类加载器hash> /tmp/Foo.java -d /tmp`（类加载器 hash 来自 §1 `sc -d` 输出；**插件类必须用插件的 ClassLoader 编译**，否则 redefine 时找不到依赖）
   - 本地 javac 编译后上传：`artifact_write_chunk` 不支持二进制——**用 arthas mc 最省事**
4. **替换**：`arthas_execute {"command":"redefine /tmp/com/example/plugin/Foo.class"}`（或经 `artifact_write_chunk` 上传 .class 到 mcp-workspace 后走 `arthas_redefine {"artifactName":"..."}`）
5. **验证**：触发该方法 → `watch` 看新行为是否生效；`jad` 再反编译确认
6. **恢复**：`arthas_revert {"entryId":<retransform -l 的编号>}` 或 `artifact_backup_restore` 拿备份再 redefine 回去

**注意**：redefine 不能改方法签名/加字段（只能改方法体）；改不了就重启加载新 jar，别硬试。

## 6. ClassLoader 问题（插件类冲突/隔离异常）

插件报 `NoSuchMethodError`/`ClassCastException`/`NoClassDefFoundError` 但 jar 里明明有——大概率是**类加载器隔离/冲突**：

```json
arthas_execute {"command":"classloader -l"}                                  // 全部加载器
arthas_execute {"command":"classloader -a"}                                  // 加载器+各自 URL
arthas_execute {"command":"sc -d com.example.plugin.Foo"}                    // 该类挂在哪个加载器
arthas_execute {"command":"classloader --load com.example.plugin.Foo"}       // 谁能加载它
```

**典型结论**：
- 同一库两个版本分别被 ServerProbe 和目标插件加载（各自插件 ClassLoader）→ 版本冲突
- 类被加载进了错误的 ClassLoader（如被根加载器抢先）→ 依赖声明问题（plugin.yml 的 depend/softdepend）

## 7. 完整实例剧本：定位一个"玩家使用某功能时报 NPE"的插件 bug

```
1. log_search {"keyword":"NullPointerException","maxLines":20}
   → 拿到堆栈：at com.shop.plugin.BuyCommand.execute(BuyCommand.java:88)
2. arthas_execute {"command":"sc -d com.shop.plugin.BuyCommand"}
   → 确认类已加载、code-source 指向的 jar 版本正确
3. arthas_execute {"command":"jad com.shop.plugin.BuyCommand execute"}
   → 反编译第 88 行附近，发现 readObject(config.get("price")) 可能返回 null
4. arthas_ognl {"expression":"@com.shop.plugin.BuyCommand@CONFIG.get('price')"}
   → 确认 price 配置确实为 null（根因：配置缺键）
5. （可选）验证修复思路：
   arthas_watch {"className":"com.shop.plugin.BuyCommand","methodName":"execute",
     "expression":"{params, config.get('price')}","maxMatches":3}
6. 决定：改配置 / 热修复（§5） / 报告插件作者——不要在他人服务器上热修复他人插件，除非获得明确授权
```
