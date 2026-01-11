# XaeroPlus 区块检测机制详细分析

## 概述

XaeroPlus 实现了三种不同的区块状态检测机制，用于识别新生成的区块和老区块。这些功能在 2b2t 等大型服务器上表现良好，但在某些服务器（如 3c3u）上可能出现误报。本文档详细分析每种检测方法的工作原理、适用场景和潜在问题。

---

## 1. OldChunks 检测机制

### 文件位置
`common/src/main/java/xaeroplus/module/impl/OldChunks.java`

### 检测原理

OldChunks 模块通过检测区块中是否存在特定版本引入的方块来判断区块是新区块还是老区块。

#### 检测逻辑

**主世界 (Overworld):**
检测是否包含 1.17+ 版本引入的方块（至少5个）：
- 铜矿石 (COPPER_ORE)
- 深层铜矿石 (DEEPSLATE_COPPER_ORE)
- 紫水晶块 (AMETHYST_BLOCK)
- 平滑玄武岩 (SMOOTH_BASALT)
- 凝灰岩 (TUFF)
- 海带 (KELP, KELP_PLANT)
- 滴水石 (POINTED_DRIPSTONE, DRIPSTONE_BLOCK)
- 深板岩 (DEEPSLATE)
- 杜鹃花 (AZALEA)
- 大型垂滴叶 (BIG_DRIPLEAF, BIG_DRIPLEAF_STEM, SMALL_DRIPLEAF)
- 苔藓块 (MOSS_BLOCK)
- 洞穴藤蔓 (CAVE_VINES, CAVE_VINES_PLANT)

**下界 (Nether):**
检测是否包含 1.16+ 版本引入的方块（至少5个）：
- 远古残骸 (ANCIENT_DEBRIS)
- 黑石 (BLACKSTONE)
- 玄武岩 (BASALT)
- 绯红菌岩 (CRIMSON_NYLIUM)
- 诡异菌岩 (WARPED_NYLIUM)
- 下界金矿石 (NETHER_GOLD_ORE)
- 锁链 (CHAIN)

**末地 (The End):**
通过检测生物群系是否为 "The End" 生物群系来判断：
- 如果生物群系不是 THE_END（例如末地外岛的其他生物群系），标记为现代区块
- 如果是 THE_END 生物群系，标记为老区块

### 工作流程

1. 当收到新区块数据时（`ChunkDataEvent`），检查该区块是否已被缓存
2. 如果未被缓存，调用 `searchChunk()` 方法
3. 使用 `ChunkScanner.chunkContainsBlocks()` 扫描区块中的方块
4. 根据扫描结果将区块添加到 `modernChunksCache` 或 `oldChunksCache`

### 优点
- 简单直接，易于理解
- 在 2b2t 等从 1.12.2 升级的服务器上非常准确
- 不依赖于 Minecraft 内部实现细节

### 局限性和潜在问题

1. **假阳性（False Positives）**
   - 如果玩家在老区块中放置了新版本的方块（如铜矿石），会被错误标记为现代区块
   - 在创造模式服务器或有管理员权限的服务器上更容易出现

2. **假阴性（False Negatives）**
   - 如果新区块中恰好没有生成足够数量（5个以上）的特征方块，可能被错误标记为老区块
   - 在沙漠、海洋等生物群系中，新版本方块生成较少

3. **服务器特殊配置**
   - 如果服务器修改了世界生成器或使用自定义地形生成
   - 如果服务器禁用了某些方块的生成

4. **版本特定性**
   - 仅适用于检测特定版本升级（1.16/1.17）的区块
   - 不适用于从其他版本升级的服务器

---

## 2. PaletteNewChunks 检测机制

### 文件位置
`common/src/main/java/xaeroplus/module/impl/PaletteNewChunks.java`

### 检测原理

这是最复杂和最可靠的检测方法，通过分析区块的方块状态调色板（BlockState Palette）和生物群系调色板（Biome Palette）的内部结构来判断区块是否新生成。

#### Minecraft 区块生成和保存机制

1. **区块生成过程：**
   - Minecraft 分多个步骤生成区块
   - 每个步骤逐步修改区块数据（先填充空气，然后添加石头、水等）
   - 生成过程中，调色板会包含所有曾经出现过的方块状态

2. **调色板压缩：**
   - 当服务器将区块写入区域文件时，会压缩调色板以节省磁盘空间
   - 压缩有两个效果：
     1. 删除调色板中不再存在的条目
     2. 按实际存在的方块状态顺序重建调色板 ID

3. **关键观察：**
   - 新生成的区块数据首先发送给玩家，然后才保存到磁盘
   - 因此，新生成的区块使用未压缩的调色板
   - 从磁盘加载的区块使用压缩的调色板

### 检测方法

#### 方法 1: 线性调色板顺序检查 (LinearPalette)

```java
checkLinearPaletteOrder(palette, section)
```

- 检查调色板 ID 的顺序是否与实际方块数据的迭代顺序匹配
- 如果顺序不匹配，说明调色板未被压缩，因此是新区块

**原理：**
- 保存到磁盘时，新调色板按 BitStorage 迭代顺序创建
- 如果当前顺序与预期不符，说明未经过保存

#### 方法 2: HashMap 调色板额外条目检查 (HashMapPalette)

```java
checkForExtraPaletteEntries(paletteContainer)
```

- 检查调色板中是否有"额外"的条目（在调色板中但实际区块中不存在）
- 如果有额外条目，说明调色板未被压缩，因此是新区块

#### 方法 3: 生物群系调色板平原检查

```java
checkNewChunkBiomePalette(chunk, checkData)
```

- Minecraft 初始化调色板时使用平原（Plains）生物群系
- 检测调色板中是否包含平原生物群系

**检测级别：**
1. `NO_PLAINS` - 调色板中没有平原生物群系 → 老区块
2. `PLAINS_IN_PALETTE` - 调色板中有平原但实际数据中没有 → 新区块（可靠）
3. `PLAINS_PRESENT` - 调色板和数据中都有平原 → 需要进一步检查方块状态调色板

### 检测流程（主世界）

```
1. 检查设置：是否启用"版本升级区块检测"
   
2. 如果启用 → 直接使用方块状态调色板检查
   - 适用于检测从 1.12.2 升级的区块（2b2t 的主要用途）
   
3. 如果未启用 → 先使用生物群系调色板检查
   - NO_PLAINS → 标记为老区块
   - PLAINS_IN_PALETTE → 标记为新区块
   - PLAINS_PRESENT → 再检查方块状态调色板
```

### 优点

1. **高度可靠**
   - 基于 Minecraft 内部机制，难以被欺骗
   - 不依赖于特定方块的存在

2. **适用于多种场景**
   - 可检测版本升级的区块
   - 可检测真正的新生成区块
   - 在所有维度都有效

3. **误报率低**
   - 玩家的行为（放置/破坏方块）不会影响检测结果
   - 除非玩家移除或添加整个方块类型

### 局限性和潜在问题

1. **假阴性场景**
   - 如果玩家移除了某个方块类型的所有实例，然后区块被保存
   - 如果其他玩家在区块加载时立即修改了方块类型
   - 末地等方块类型较少的区块

2. **性能考虑**
   - 需要迭代整个区块的方块存储
   - 对于 HashMapPalette 和 LinearPalette 的处理逻辑较复杂

3. **版本兼容性**
   - 依赖于 Minecraft 的内部实现
   - 如果 Minecraft 改变调色板压缩算法，可能失效

4. **服务器修改**
   - 如果服务器使用自定义的区块保存/加载逻辑
   - 如果服务器预生成并保存所有区块

---

## 3. LiquidNewChunks 检测机制

### 文件位置
`common/src/main/java/xaeroplus/module/impl/LiquidNewChunks.java`

### 检测原理

利用流体（水和岩浆）的流动行为来检测新区块。新生成的区块中的流体不会在生成时流动，只有在玩家加载后才开始流动。

#### 工作机制

1. **实时流体更新检测：**
   - 监听 `ChunkBlockUpdateEvent` 和 `ChunkBlocksUpdateEvent`
   - 当检测到非源方块的流体更新时：
     - 检查相邻方块是否有源方块
     - 如果有，标记为新区块（流体开始从源方块流出）

2. **区块加载时的流体柱检测：**
   - 当收到区块数据时（`ChunkDataEvent`）
   - 扫描区块中的所有流体方块
   - 检测特定模式的流体柱：
     - 流体不是源方块
     - 流体量小于 2 → 标记为老区块（inverse）
     - 或者存在连续 5 格以上的流体柱 → 标记为老区块（inverse）

### 检测逻辑

**新区块标记条件：**
- 流体从源方块开始流动（在玩家加载区块后）
- 流体更新事件发生时，相邻存在源方块

**老区块标记条件（inverse）：**
- 区块加载时已经存在流动的流体
- 存在流体柱（连续 5 格以上的流动流体）
- 流体量很少（< 2）

### 可选设置

`liquidNewChunksOnlyAboveY0Setting` - 仅检测 Y > 0 的流体：
- 减少误报（地下的流体更复杂）
- 专注于地表流体检测

### 优点

1. **实时检测**
   - 可以在区块加载的瞬间检测到新区块
   - 不需要等待完整的区块数据分析

2. **独特的检测角度**
   - 基于动态行为而非静态数据
   - 与其他两种方法互补

3. **易于理解**
   - 基于直观的流体物理行为
   - 调试和验证相对简单

### 局限性和潜在问题

1. **假阳性（标记为新区块）**
   - 玩家在老区块中放置水源或岩浆源
   - 服务器重启后重新加载区块导致流体重新计算
   - 如果服务器修改了流体更新逻辑

2. **假阴性（标记为老区块）**
   - 新生成的区块如果没有流体或流体不流动
   - 沙漠、冰原等干燥生物群系
   - 流体更新事件丢失或延迟

3. **环境依赖性强**
   - 依赖于 Minecraft 的流体物理系统
   - 服务器的 tick 速度影响检测
   - 网络延迟可能导致事件丢失

4. **缓存管理**
   - 使用 Caffeine 缓存来避免重复扫描
   - 缓存过期后可能重复检测
   - 内存使用考虑

---

## 在 3c3u 服务器上的问题分析

根据问题描述，XaeroPlus 在 3c3u 服务器上显示"随机的乱区块高亮"。可能的原因包括：

### 1. 服务器使用预生成区块

**问题：**
- 如果 3c3u 预先生成并保存了所有区块
- PaletteNewChunks 会正确识别这些为老区块（已保存）
- OldChunks 可能误报，如果预生成使用了新版本

**解决方案：**
- 在 3c3u 上禁用 OldChunks 检测
- 仅使用 PaletteNewChunks
- 启用 `paletteNewChunksVersionUpgradedChunks` 设置

### 2. 服务器自定义世界生成

**问题：**
- 3c3u 可能使用自定义地形生成器
- OldChunks 依赖的特征方块可能不生成或生成模式不同
- 生物群系配置可能与原版不同

**解决方案：**
- 分析 3c3u 的实际区块数据
- 调整检测方块列表以匹配服务器配置
- 考虑禁用 OldChunks

### 3. 流体机制差异

**问题：**
- 3c3u 可能修改了流体更新机制
- 可能使用了性能优化插件改变流体行为
- 可能禁用或延迟了流体更新

**解决方案：**
- 在 3c3u 上禁用 LiquidNewChunks
- 或者启用 `liquidNewChunksOnlyAboveY0Setting` 减少误报

### 4. 服务器修改了区块保存格式

**问题：**
- 3c3u 可能使用了自定义的区块存储格式
- 调色板压缩算法可能不同
- 区块数据发送到客户端前可能被修改

**解决方案：**
- 需要实际抓包分析区块数据格式
- 可能需要为 3c3u 添加特殊处理逻辑

---

## 优化建议

### 针对 3c3u 服务器的配置建议

1. **禁用 OldChunks 检测**
   ```
   设置 → Chunk Highlights → Old Chunks → 关闭
   ```

2. **使用 PaletteNewChunks 作为主要检测方法**
   ```
   设置 → Chunk Highlights → Palette NewChunks → 开启
   设置 → Chunk Highlights → Palette NewChunks Version Upgraded → 根据服务器版本历史决定
   ```

3. **禁用 LiquidNewChunks 或限制检测范围**
   ```
   设置 → Chunk Highlights → Liquid NewChunks → 关闭
   或者
   设置 → Chunk Highlights → Liquid NewChunks Only Y > 0 → 开启
   ```

### 代码级优化建议

1. **添加服务器特定配置文件**
   - 创建服务器配置文件系统
   - 根据服务器地址自动应用不同的检测策略
   - 支持自定义检测方块列表

2. **改进 OldChunks 检测**
   - 添加可配置的方块列表
   - 支持从服务器信息自动检测世界生成版本
   - 添加方块数量阈值的配置选项

3. **增强 PaletteNewChunks 可靠性**
   - 添加多级检测策略
   - 实现检测结果的置信度评分
   - 支持手动校准和调整

4. **改进 LiquidNewChunks 准确性**
   - 添加流体更新频率分析
   - 实现更智能的流体模式识别
   - 支持排除特定区域的检测

5. **添加调试和诊断工具**
   - 实现详细的检测日志
   - 添加区块数据查看器
   - 支持导出检测结果进行分析

---

## 测试和验证方法

### 验证检测准确性

1. **已知新区块测试**
   - 在本地单人游戏中生成新区块
   - 验证三种方法都能正确识别

2. **已知老区块测试**
   - 使用从旧版本升级的世界
   - 验证能正确识别未修改的老区块

3. **边界情况测试**
   - 玩家修改过的区块
   - 部分生成的区块
   - 不同生物群系的区块

### 3c3u 服务器特定测试

1. **收集样本数据**
   - 记录已知新生成区块的坐标
   - 记录已知老区块的坐标
   - 对比三种方法的检测结果

2. **分析误报模式**
   - 记录所有被标记为新区块的坐标
   - 手动验证这些区块的实际状态
   - 分析误报的共同特征

3. **性能测试**
   - 测量每种方法的 CPU 使用率
   - 测量内存占用
   - 测量检测延迟

---

## 总结

### 各检测方法对比

| 特性 | OldChunks | PaletteNewChunks | LiquidNewChunks |
|------|-----------|------------------|-----------------|
| 可靠性 | 中 | 高 | 中 |
| 性能 | 高 | 中 | 中 |
| 误报率 | 高 | 低 | 中 |
| 假阴性率 | 中 | 低 | 高 |
| 实现复杂度 | 低 | 高 | 中 |
| 服务器兼容性 | 低 | 高 | 中 |
| 玩家行为影响 | 高 | 低 | 高 |

### 推荐使用策略

**2b2t 服务器：**
- 主要使用：PaletteNewChunks（启用 Version Upgraded）
- 辅助使用：OldChunks
- 可选使用：LiquidNewChunks

**3c3u 服务器：**
- 主要使用：PaletteNewChunks（根据实际情况配置）
- 不推荐：OldChunks
- 谨慎使用：LiquidNewChunks（建议禁用或限制 Y > 0）

**一般服务器：**
- 主要使用：PaletteNewChunks（禁用 Version Upgraded）
- 不使用：OldChunks（除非确认适用）
- 可选使用：LiquidNewChunks（用于实时检测）

### 最佳实践

1. **首次使用时进行测试**
   - 在已知区域测试检测准确性
   - 根据结果调整配置

2. **定期验证**
   - 记录检测结果
   - 与实际情况对比
   - 及时调整配置

3. **使用多种方法交叉验证**
   - 对重要区域使用多种检测方法
   - 注意不同方法结果的差异
   - 优先信任 PaletteNewChunks

4. **保持更新**
   - 关注 XaeroPlus 更新
   - 了解 Minecraft 版本变化
   - 根据服务器更新调整配置

---

## 技术参考

### 相关 Minecraft 概念

- **Chunk Palette（区块调色板）**：用于压缩存储方块状态的数据结构
- **BitStorage**：存储调色板索引的位数组
- **LinearPalette**：简单的列表式调色板（小型区块段）
- **HashMapPalette**：哈希表式调色板（中型区块段）
- **Chunk Section**：区块的 16x16x16 子区块

### 参考链接

- Minecraft Wiki - Chunk Format: https://minecraft.wiki/w/Chunk_format
- Minecraft Wiki - Data Version: https://minecraft.wiki/w/Data_version
- Henrik Kniberg - Minecraft Terrain Generation: https://youtu.be/ob3VwY4JyzE
- Trouser-Streak Project: https://github.com/etianl/Trouser-Streak

---

## 贡献者注释

本文档基于 XaeroPlus 当前版本的代码分析编写。如果代码有更新或发现文档中的错误，请提交 PR 或 Issue。

**主要代码文件：**
- `OldChunks.java` - 老区块检测实现
- `PaletteNewChunks.java` - 调色板新区块检测实现
- `LiquidNewChunks.java` - 流体新区块检测实现
- `ChunkUtils.java` - 区块工具类
- `Settings.java` - 设置定义

**相关设置：**
- `paletteNewChunksVersionUpgradedChunks` - 是否检测版本升级的区块
- `liquidNewChunksOnlyAboveY0Setting` - 仅检测 Y > 0 的流体
- 各检测方法的启用/禁用和颜色设置
