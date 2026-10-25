# AGENTS.md — NetherEnough 协作指南

本文件面向在本仓库工作的 AI 助手与人类贡献者：项目是什么、怎么构建、代码怎么组织、有哪些约定和坑。动手前先通读一遍。

## 项目概述

NetherEnough（中文名「深渊幻章」）是一个 Minecraft 服务端插件，**Slimefun4（粘液科技）附属**，为下界扩展内容：两个自定义世界（Muspelheim / Niflheim）、专属地形生成器与矿脉填充器、六种元素粉末、蒸馏器机器、火种与炼金火焰、魔力法杖等。

- 上游：`SlimefunGuguProject/NetherEnough`（git upstream）；origin 为 `balugaq/NetherEnough`
- 本地化：haiman233 汉化分支，面向玩家文本均为中文
- 硬依赖：Slimefun（plugin.yml `depend: [Slimefun]`）；README 另注明前置 EliteMobs
- 运行环境：Spigot / Paper 服务端；Spigot API 1.20.1-R0.1-SNAPSHOT，api-version 1.16
- 许可：MIT（Copyright (c) 2020 Sfiguz7）

## 技术栈

- Java 8（`maven.compiler.source/target = 1.8`）——不要引入更高版本语言特性
- Maven；主要依赖：spigot-api（provided）、Slimefun4（jitpack，provided）、jsr305（`@Nonnull` 等注解）
- 测试：pom 已配 JUnit 5 + reflections，仓库内暂无测试代码
- CI：`.github/workflows/maven.yml`，push / PR 到 master 时 JDK 1.8 执行 `mvn package --file pom.xml`

## 构建

```bash
mvn clean package        # pom 的 defaultGoal 即 clean package
```

- 产物：`target/NetherEnough v1.0.0.jar`（finalName 含空格，shell 引用记得加引号）
- `src/main/resources` 开启了 filtering，`plugin.yml` 里的 `${project.version}` 构建时替换为 pom 版本号
- 改动后至少保证 `mvn clean package` 通过再交付

## 目录结构

```
src/main/java/me/sfiguz7/netherenough/
├── NetherEnough.java               # 主类（JavaPlugin + SlimefunAddon），所有注册入口集中在 onEnable
├── commands/NECommands.java        # /netherenough（别名 ne），tpworld 子命令 + 权限校验
├── lists/
│   ├── NEItems.java                # 全部 SlimefunItemStack 与物品分类 ItemGroup 的定义
│   ├── NERecipeType.java           # 配方类型
│   └── NERegistry.java             # 运行时状态：neWorlds / infusedBlocks / ignisBlocks
├── generation/
│   ├── worlds/                     # NEWorld（世界加载 + Type 枚举）、MuspelheimGenerator、NiflheimGenerator
│   └── populators/                 # BlockPopulator：煤矿 / 哭泣黑曜石 / 合金砖 / 岩浆砖 / 红石块矿脉
├── implementation/
│   ├── items/items/                # Powder（六元素）、FireStarter、AlchemyFire（南极/贝拉火焰）、ManaRod
│   ├── items/machines/Alembic.java # 蒸馏器（SimpleSlimefunItem<BlockTicker> + NEInventoryBlock）
│   ├── listeners/                  # 6 个事件监听（火焰、注入方块、幻翼、世界刷怪）
│   └── utils/interfaces/NEInventoryBlock.java
└── utils/Utils.java                # chance() / teleportToWorld() / isNEWorld()
src/main/resources/
├── plugin.yml                      # 命令、权限定义
└── config.yml                      # options.auto-update、options.manarod-infusing-chance
```

## 新增内容的注册流程

项目遵循「枚举驱动 + onEnable 集中注册」模式，新内容按此套路加：

1. **新物品**：先在 `lists/NEItems.java` 定义 `SlimefunItemStack`（ID、Material、中文名、lore，颜色码用 `&x` 格式）；再在 `implementation/items/items` 下写类（`SlimefunItem` 子类，多样式变体用 `enum Type` 驱动，仿 `Powder`）；最后在 `NetherEnough.onEnable()` 里 `new Xxx().register(this)` 或遍历 `Type.values()` 注册。
2. **新研究**：`new Research(new NamespacedKey(this, "<key>"), ++researchId, "<名>", <成本>)`；researchId 从 7700 起自增，禁止复用旧值。
3. **新机器**：仿 `Alembic`——`SimpleSlimefunItem<BlockTicker>` + 实现 `NEInventoryBlock`，配方表用静态 Map 管理（`getAlembicRecipes()`）。
4. **新世界**：在 `NEWorld.Type` 枚举登记（世界名固定 `ne_<name>`，onEnable 自动加载并登记进 NERegistry）；地形写在 `ChunkGenerator`，矿脉写在 `BlockPopulator` 并挂到 `getDefaultPopulators()`。
5. **运行时状态**一律放 `NERegistry`，经 `NetherEnough.getRegistry()` 静态访问，不要另起全局单例。

## 代码约定

- 包名 `me.sfiguz7.netherenough` 与上游一致，**不要改**（关系到配置与存档兼容性）
- Bukkit 空安全：类上标 `@ParametersAreNonnullByDefault`，字段/返回值按需 `@Nonnull`（javax.annotation）
- 面向玩家文本用中文；颜色代码统一走 `ChatColors.color("&x...")` 或 `ChatColor`
- 权限节点前缀 `ne.`，`ne.command.tpworld` 默认 op
- 配置项在 `onEnable` 读取并校验，非法值要打 SEVERE 日志并 `disablePlugin`（参照 `manarod-infusing-chance` 0–100 的处理）
- 修改 `NEItems` 中已有物品的 ID 或 Material 会破坏玩家存档里的既有物品——只增不改

## 已知问题与注意事项

- `Utils.teleportToWorld()` 第 21 行 `double z = loc.getX();` 疑似笔误（应为 `loc.getZ()`），传送落点 x/z 相同；动它之前先确认影响面
- `Alembic` 的 `idx2slot()` 映射与 decrement 节流逻辑带 Slimefun 源码复制痕迹，重构前需游戏内验证
- 自动更新（GitHubBuildsUpdater）只在版本号以 `"DEV - "` 开头时触发，指向 `balugaq/NetherEnough`；汉化分支如需自建更新通道改这里
- Powder 的 `preRegister()` 用 `Enchantment.ARROW_INFINITY` 附魔做发光效果，是既有 hack，勿「顺手清理」

## Git 与提交边界

- **AI 助手不执行 commit / push / 不改写历史**；只把工作区整理到可验收状态，并向主人给出建议的提交信息。提交权归主人。
- 动手前先 `git status` 确认基线（仓库存在未提交改动，如 .gitignore）。

## 验收方式

- 静态验收：`mvn clean package` 编译通过
- 游戏内表现（世界生成、机器配方、命令行为）需主人在测试服务器实测，AI 不代开服务器或浏览器验证
