# XaeroPlus Chunk Detection Mechanisms - Detailed Analysis

## Overview

XaeroPlus implements three different chunk state detection mechanisms to identify newly generated chunks and old chunks. These features work well on large servers like 2b2t, but may show false positives on certain servers (such as 3c3u). This document provides a detailed analysis of each detection method's working principles, applicable scenarios, and potential issues.

---

## 1. OldChunks Detection Mechanism

### File Location
`common/src/main/java/xaeroplus/module/impl/OldChunks.java`

### Detection Principle

The OldChunks module determines whether a chunk is new or old by checking for the presence of specific blocks introduced in certain Minecraft versions.

#### Detection Logic

**Overworld:**
Checks for blocks introduced in 1.17+ (scanning from Y=5 and above):
- Copper Ore (COPPER_ORE)
- Deepslate Copper Ore (DEEPSLATE_COPPER_ORE)
- Amethyst Block (AMETHYST_BLOCK)
- Smooth Basalt (SMOOTH_BASALT)
- Tuff (TUFF)
- Kelp (KELP, KELP_PLANT)
- Dripstone (POINTED_DRIPSTONE, DRIPSTONE_BLOCK)
- Deepslate (DEEPSLATE)
- Azalea (AZALEA)
- Big Dripleaf (BIG_DRIPLEAF, BIG_DRIPLEAF_STEM, SMALL_DRIPLEAF)
- Moss Block (MOSS_BLOCK)
- Cave Vines (CAVE_VINES, CAVE_VINES_PLANT)

**Nether:**
Checks for blocks introduced in 1.16+ (scanning from Y=5 and above):
- Ancient Debris (ANCIENT_DEBRIS)
- Blackstone (BLACKSTONE)
- Basalt (BASALT)
- Crimson Nylium (CRIMSON_NYLIUM)
- Warped Nylium (WARPED_NYLIUM)
- Nether Gold Ore (NETHER_GOLD_ORE)
- Chain (CHAIN)

**The End:**
Determines by checking if the biome is "The End" biome:
- If biome is not THE_END (e.g., outer end islands with other biomes) → modern chunk
- If biome is THE_END → old chunk

### Workflow

1. When new chunk data is received (`ChunkDataEvent`), check if chunk is already cached
2. If not cached, call `searchChunk()` method
3. Use `ChunkScanner.chunkContainsBlocks()` to scan blocks in the chunk (from Y=5 and above)
4. If any characteristic block is found, mark as modern chunk; otherwise mark as old chunk

### Advantages
- Simple and straightforward, easy to understand
- Very accurate on servers like 2b2t that upgraded from 1.12.2
- Does not rely on Minecraft internal implementation details

### Limitations and Potential Issues

1. **False Positives**
   - If players place new version blocks (like copper ore) in old chunks, they will be incorrectly marked as modern chunks
   - More likely to occur on creative mode servers or servers with admin privileges

2. **False Negatives**
   - If a new chunk happens not to generate any characteristic blocks, it may be incorrectly marked as old
   - In biomes like deserts and oceans, new version blocks generate less frequently
   - Blocks below Y=5 are not scanned, potentially missing some deep characteristic blocks

3. **Server-Specific Configurations**
   - If server modifies world generator or uses custom terrain generation
   - If server disables generation of certain blocks

4. **Version Specificity**
   - Only suitable for detecting chunks from specific version upgrades (1.16/1.17)
   - Not applicable for servers upgraded from other versions

---

## 2. PaletteNewChunks Detection Mechanism

### File Location
`common/src/main/java/xaeroplus/module/impl/PaletteNewChunks.java`

### Detection Principle

This is the most complex and reliable detection method, analyzing the internal structure of chunk BlockState Palette and Biome Palette to determine if a chunk is newly generated.

#### Minecraft Chunk Generation and Saving Mechanism

1. **Chunk Generation Process:**
   - Minecraft generates chunks in multiple steps
   - Each step progressively mutates chunk data (first fill with air, then add stone, water, etc.)
   - During generation, the palette contains all block states that ever appeared

2. **Palette Compaction:**
   - When server writes chunks to region files, it compacts the palette to save disk space
   - Compaction has two effects:
     1. Removes palette entries without values present in the chunk
     2. Rebuilds palette IDs in order of actual blockstates present in the chunk

3. **Key Observation:**
   - Newly generated chunk data is first sent to players, then saved to disk
   - Therefore, newly generated chunks use uncompacted palettes
   - Chunks loaded from disk use compacted palettes

### Detection Methods

#### Method 1: Linear Palette Order Check (LinearPalette)

```java
checkLinearPaletteOrder(palette, section)
```

- Checks if palette ID order matches the iteration order of actual block data
- If order doesn't match, palette is uncompacted, therefore it's a new chunk

**Principle:**
- When saved to disk, new palette is created in BitStorage iteration order
- If current order doesn't match expected, it hasn't been saved

#### Method 2: HashMap Palette Extra Entries Check (HashMapPalette)

```java
checkForExtraPaletteEntries(paletteContainer)
```

- Checks if there are "extra" entries in palette (in palette but not present in actual chunk)
- If extra entries exist, palette is uncompacted, therefore it's a new chunk

#### Method 3: Biome Palette Plains Check

```java
checkNewChunkBiomePalette(chunk, checkData)
```

- Minecraft initializes palettes with Plains biome
- Detects if palette contains Plains biome

**Detection Levels:**
1. `NO_PLAINS` - No plains biome in palette → old chunk
2. `PLAINS_IN_PALETTE` - Plains in palette but not in actual data → new chunk (reliable)
3. `PLAINS_PRESENT` - Plains in both palette and data → requires further blockstate palette check

### Detection Flow (Overworld)

```
1. Check setting: Is "Version Upgraded Chunks Detection" enabled?
   
2. If enabled → Use blockstate palette check directly
   - Suitable for detecting chunks upgraded from 1.12.2 (main use on 2b2t)
   
3. If disabled → Use biome palette check first
   - NO_PLAINS → Mark as old chunk
   - PLAINS_IN_PALETTE → Mark as new chunk
   - PLAINS_PRESENT → Then check blockstate palette
```

### Advantages

1. **Highly Reliable**
   - Based on Minecraft internal mechanisms, difficult to fool
   - Does not depend on presence of specific blocks

2. **Applicable to Multiple Scenarios**
   - Can detect version-upgraded chunks
   - Can detect truly newly generated chunks
   - Effective in all dimensions

3. **Low False Positive Rate**
   - Player actions (placing/breaking blocks) don't affect detection
   - Unless player removes or adds entire block types

### Limitations and Potential Issues

1. **False Negative Scenarios**
   - If player removes all instances of a block type, then chunk is saved
   - If other players immediately modify block types when chunk loads
   - Chunks with few block types like in The End

2. **Performance Considerations**
   - Needs to iterate through entire chunk's block storage
   - Complex processing logic for HashMapPalette and LinearPalette

3. **Version Compatibility**
   - Relies on Minecraft's internal implementation
   - May fail if Minecraft changes palette compaction algorithm

4. **Server Modifications**
   - If server uses custom chunk save/load logic
   - If server pre-generates and saves all chunks

---

## 3. LiquidNewChunks Detection Mechanism

### File Location
`common/src/main/java/xaeroplus/module/impl/LiquidNewChunks.java`

### Detection Principle

Uses fluid (water and lava) flow behavior to detect new chunks. Fluids in newly generated chunks don't flow during generation; they only start flowing after players load them.

#### Working Mechanism

1. **Real-time Fluid Update Detection:**
   - Listens to `ChunkBlockUpdateEvent` and `ChunkBlocksUpdateEvent`
   - When non-source fluid update is detected:
     - Check if adjacent blocks have source blocks
     - If yes, mark as new chunk (fluid started flowing from source block)

2. **Fluid Column Detection on Chunk Load:**
   - When chunk data is received (`ChunkDataEvent`)
   - Scan all fluid blocks in the chunk
   - Detect specific patterns of fluid columns:
     - Fluid is not a source block
     - Fluid amount < 2 → mark as old chunk (inverse)
     - Or 5+ consecutive fluid blocks exist → mark as old chunk (inverse)

### Detection Logic

**New Chunk Marking Conditions:**
- Fluid starts flowing from source block (after player loads chunk)
- Fluid update event occurs with adjacent source block

**Old Chunk Marking Conditions (inverse):**
- Flowing fluid already exists when chunk loads
- Fluid column exists (5+ consecutive flowing fluid blocks)
- Very low fluid amount (< 2)

### Optional Settings

`liquidNewChunksOnlyAboveY0Setting` - Only detect fluids above Y > 0:
- Reduces false positives (underground fluids are more complex)
- Focuses on surface fluid detection

### Advantages

1. **Real-time Detection**
   - Can detect new chunks at the moment of loading
   - Doesn't need to wait for complete chunk data analysis

2. **Unique Detection Angle**
   - Based on dynamic behavior rather than static data
   - Complements the other two methods

3. **Easy to Understand**
   - Based on intuitive fluid physics behavior
   - Debugging and verification relatively simple

### Limitations and Potential Issues

1. **False Positives (marked as new chunk)**
   - Player places water or lava source in old chunk
   - Server restart causes chunk reload and fluid recalculation
   - Server modifies fluid update logic

2. **False Negatives (marked as old chunk)**
   - Newly generated chunk has no fluid or fluid doesn't flow
   - Dry biomes like desert, ice plains
   - Fluid update events lost or delayed

3. **Strong Environment Dependency**
   - Depends on Minecraft's fluid physics system
   - Server tick speed affects detection
   - Network latency may cause event loss

4. **Cache Management**
   - Uses Caffeine cache to avoid repeated scanning
   - May re-detect after cache expires
   - Memory usage considerations

---

## Problem Analysis on 3c3u Server

According to the problem description, XaeroPlus shows "random incorrect chunk highlights" on 3c3u server. Possible reasons include:

### 1. Server Uses Pre-generated Chunks

**Problem:**
- If 3c3u pre-generates and saves all chunks
- PaletteNewChunks correctly identifies these as old chunks (already saved)
- OldChunks may misreport if pre-generation used new version

**Solution:**
- Disable OldChunks detection on 3c3u
- Only use PaletteNewChunks
- Enable `paletteNewChunksVersionUpgradedChunks` setting

### 2. Server Custom World Generation

**Problem:**
- 3c3u may use custom terrain generator
- Characteristic blocks that OldChunks relies on may not generate or have different patterns
- Biome configuration may differ from vanilla

**Solution:**
- Analyze actual chunk data from 3c3u
- Adjust detection block list to match server configuration
- Consider disabling OldChunks

### 3. Fluid Mechanism Differences

**Problem:**
- 3c3u may modify fluid update mechanism
- May use performance optimization plugins that change fluid behavior
- May disable or delay fluid updates

**Solution:**
- Disable LiquidNewChunks on 3c3u
- Or enable `liquidNewChunksOnlyAboveY0Setting` to reduce false positives

### 4. Server Modified Chunk Save Format

**Problem:**
- 3c3u may use custom chunk storage format
- Palette compaction algorithm may be different
- Chunk data may be modified before sent to client

**Solution:**
- Need packet capture to analyze chunk data format
- May need to add special handling logic for 3c3u

---

## Optimization Recommendations

### Configuration Recommendations for 3c3u Server

1. **Disable OldChunks Detection**
   ```
   Settings → Chunk Highlights → Old Chunks → Off
   ```

2. **Use PaletteNewChunks as Primary Detection Method**
   ```
   Settings → Chunk Highlights → Palette NewChunks → On
   Settings → Chunk Highlights → Palette NewChunks Version Upgraded → Decide based on server version history
   ```

3. **Disable LiquidNewChunks or Limit Detection Range**
   ```
   Settings → Chunk Highlights → Liquid NewChunks → Off
   Or
   Settings → Chunk Highlights → Liquid NewChunks Only Y > 0 → On
   ```

### Code-Level Optimization Recommendations

1. **Add Server-Specific Configuration Files**
   - Create server configuration file system
   - Automatically apply different detection strategies based on server address
   - Support custom detection block lists

2. **Improve OldChunks Detection**
   - Add configurable block lists
   - Support auto-detection of world generation version from server info
   - Add configuration option for block count threshold

3. **Enhance PaletteNewChunks Reliability**
   - Add multi-level detection strategies
   - Implement confidence scoring for detection results
   - Support manual calibration and adjustment

4. **Improve LiquidNewChunks Accuracy**
   - Add fluid update frequency analysis
   - Implement smarter fluid pattern recognition
   - Support excluding specific areas from detection

5. **Add Debug and Diagnostic Tools**
   - Implement detailed detection logging
   - Add chunk data viewer
   - Support exporting detection results for analysis

---

## Testing and Verification Methods

### Verify Detection Accuracy

1. **Known New Chunk Tests**
   - Generate new chunks in local singleplayer
   - Verify all three methods correctly identify them

2. **Known Old Chunk Tests**
   - Use world upgraded from old version
   - Verify correct identification of unmodified old chunks

3. **Edge Case Tests**
   - Player-modified chunks
   - Partially generated chunks
   - Chunks from different biomes

### 3c3u Server Specific Tests

1. **Collect Sample Data**
   - Record coordinates of known newly generated chunks
   - Record coordinates of known old chunks
   - Compare detection results from all three methods

2. **Analyze False Positive Patterns**
   - Record all coordinates marked as new chunks
   - Manually verify actual state of these chunks
   - Analyze common characteristics of false positives

3. **Performance Tests**
   - Measure CPU usage of each method
   - Measure memory usage
   - Measure detection latency

---

## Summary

### Comparison of Detection Methods

| Feature | OldChunks | PaletteNewChunks | LiquidNewChunks |
|---------|-----------|------------------|-----------------|
| Reliability | Medium | High | Medium |
| Performance | High | Medium | Medium |
| False Positive Rate | High | Low | Medium |
| False Negative Rate | Medium | Low | High |
| Implementation Complexity | Low | High | Medium |
| Server Compatibility | Low | High | Medium |
| Player Behavior Impact | High | Low | High |

### Recommended Usage Strategy

**2b2t Server:**
- Primary: PaletteNewChunks (enable Version Upgraded)
- Secondary: OldChunks
- Optional: LiquidNewChunks

**3c3u Server:**
- Primary: PaletteNewChunks (configure based on actual situation)
- Not Recommended: OldChunks
- Use Cautiously: LiquidNewChunks (recommend disable or limit Y > 0)

**General Servers:**
- Primary: PaletteNewChunks (disable Version Upgraded)
- Don't Use: OldChunks (unless confirmed applicable)
- Optional: LiquidNewChunks (for real-time detection)

### Best Practices

1. **Test When First Using**
   - Test detection accuracy in known areas
   - Adjust configuration based on results

2. **Regular Verification**
   - Record detection results
   - Compare with actual situation
   - Adjust configuration promptly

3. **Cross-Validate with Multiple Methods**
   - Use multiple detection methods for important areas
   - Note differences between method results
   - Trust PaletteNewChunks preferentially

4. **Keep Updated**
   - Follow XaeroPlus updates
   - Understand Minecraft version changes
   - Adjust configuration based on server updates

---

## Technical Reference

### Related Minecraft Concepts

- **Chunk Palette**: Data structure for compressed storage of block states
- **BitStorage**: Bit array storing palette indices
- **LinearPalette**: Simple list-style palette (small chunk sections)
- **HashMapPalette**: Hash map-style palette (medium chunk sections)
- **Chunk Section**: 16x16x16 sub-section of a chunk

### Reference Links

- Minecraft Wiki - Chunk Format: https://minecraft.wiki/w/Chunk_format
- Minecraft Wiki - Data Version: https://minecraft.wiki/w/Data_version
- Henrik Kniberg - Minecraft Terrain Generation: https://youtu.be/ob3VwY4JyzE
- Trouser-Streak Project: https://github.com/etianl/Trouser-Streak

---

## Contributor Notes

This document is written based on code analysis of the current version of XaeroPlus. If the code is updated or errors are found in the documentation, please submit a PR or Issue.

**Main Code Files:**
- `OldChunks.java` - Old chunks detection implementation
- `PaletteNewChunks.java` - Palette new chunks detection implementation
- `LiquidNewChunks.java` - Liquid new chunks detection implementation
- `ChunkUtils.java` - Chunk utility class
- `Settings.java` - Settings definitions

**Related Settings:**
- `paletteNewChunksVersionUpgradedChunks` - Whether to detect version-upgraded chunks
- `liquidNewChunksOnlyAboveY0Setting` - Only detect fluids above Y > 0
- Enable/disable and color settings for each detection method
