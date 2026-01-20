# Window Types Reference

Complete reference for all window types and their configuration options.

## WindowType Enum

| Type | Description | Slots | Features |
|------|-------------|-------|----------|
| `Container` | Generic item storage | Variable (9n) | Item slots only |
| `PocketCrafting` | 2x2 portable crafting | 5 | Input grid + output |
| `BasicCrafting` | 3x3 crafting table | 10 | Input grid + output |
| `DiagramCrafting` | Blueprint-based crafting | Variable | Recipe diagrams |
| `StructuralCrafting` | Building/construction | Variable | 3D structure preview |
| `Processing` | Time-based processing | Variable | Fuel + progress bar |
| `Memories` | Achievement/memory display | Variable | Read-only display |

## Container Window

Basic item storage window. Used for chests, backpacks, and custom inventories.

```java
public class ChestWindow extends Window {
    
    public ChestWindow(Player player, int rows) {
        super(WindowType.Container, rows * 9);
    }
    
    @Override
    public String getTitle() {
        return "Chest";
    }
}
```

**Configuration:**
- Size must be multiple of 9
- Maximum 6 rows (54 slots)
- Supports all slot operations

**Use cases:**
- Storage containers
- Menu GUIs
- Trading interfaces
- Reward displays

## PocketCrafting Window

Portable 2x2 crafting grid (player inventory crafting).

```java
public class PocketCraftWindow extends Window {
    
    // Slot layout:
    // [0][1]  [4] <- Output
    // [2][3]
    
    public PocketCraftWindow(Player player) {
        super(WindowType.PocketCrafting, 5);
    }
    
    @Override
    protected void onAction(Player player, WindowAction action) {
        if (action instanceof CraftRecipeAction craft) {
            // Handle 2x2 crafting
            attemptCraft(player, craft.getRecipeId());
        }
    }
}
```

**Slot Layout:**
| Index | Purpose |
|-------|---------|
| 0-3 | 2x2 input grid |
| 4 | Output slot |

**Features:**
- Limited recipe set (2x2 only)
- No bench requirement
- Accessible anywhere

## BasicCrafting Window

Standard 3x3 crafting table interface.

```java
public class CraftingTableWindow extends CraftingWindow {
    
    // Slot layout:
    // [0][1][2]  [9] <- Output
    // [3][4][5]
    // [6][7][8]
    
    public CraftingTableWindow(Player player, BlockPos pos) {
        super(player, pos, BenchType.Crafting);
    }
    
    @Override
    protected List<RecipeCategory> getAvailableCategories() {
        return List.of(
            RecipeCategory.TOOLS,
            RecipeCategory.WEAPONS,
            RecipeCategory.ARMOR,
            RecipeCategory.BUILDING,
            RecipeCategory.MISC
        );
    }
}
```

**Slot Layout:**
| Index | Purpose |
|-------|---------|
| 0-8 | 3x3 input grid |
| 9 | Output slot |

**Features:**
- Full recipe access
- Category filtering
- Recipe book integration

## DiagramCrafting Window

Blueprint-based crafting for advanced recipes.

```java
public class BlueprintBenchWindow extends DiagramCraftingWindow {
    
    public BlueprintBenchWindow(Player player, BlockPos pos) {
        super(player, pos);
    }
    
    @Override
    public String getTitle() {
        return "Blueprint Workbench";
    }
    
    @Override
    protected List<BlueprintRecipe> getAvailableBlueprints(Player player) {
        // Return blueprints player has unlocked
        return player.getKnowledgeManager()
            .getUnlockedBlueprints()
            .stream()
            .filter(bp -> bp.getBenchType() == BenchType.DiagramCrafting)
            .toList();
    }
    
    @Override
    protected void onBlueprintSelected(Player player, BlueprintRecipe blueprint) {
        // Show required materials
        displayRequirements(blueprint.getInputs());
    }
}
```

**Features:**
- Visual diagram display
- Blueprint unlock system
- Complex multi-step recipes
- Preview of output

## StructuralCrafting Window

For building and construction recipes with 3D previews.

```java
public class ConstructionBenchWindow extends StructuralCraftingWindow {
    
    public ConstructionBenchWindow(Player player, BlockPos pos) {
        super(player, pos);
    }
    
    @Override
    public String getTitle() {
        return "Construction Bench";
    }
    
    @Override
    protected List<StructuralRecipe> getAvailableStructures() {
        return StructuralRecipeRegistry.getAll();
    }
    
    @Override
    protected void onStructureSelected(Player player, StructuralRecipe recipe) {
        // Send 3D preview to client
        sendStructurePreview(player, recipe.getPreviewData());
    }
    
    @Override
    protected boolean canBuild(Player player, StructuralRecipe recipe) {
        // Check materials and space
        return hasRequiredMaterials(player, recipe) && 
               hasSpaceToBuild(player.getWorld(), getOutputPosition());
    }
}
```

**Features:**
- 3D structure preview
- Placement validation
- Multi-block output
- Rotation support

## Processing Window

Furnace-like processing with fuel and time.

```java
public class FurnaceWindow extends ProcessingWindow {
    
    // Slot layout:
    // [0] <- Input
    // [1] <- Fuel
    // [2] <- Output
    
    public FurnaceWindow(Player player, BlockPos pos) {
        super(player, pos, BenchType.Processing);
    }
    
    @Override
    public String getTitle() {
        return "Furnace";
    }
    
    @Override
    protected int getInputSlot() { return 0; }
    
    @Override
    protected int getFuelSlot() { return 1; }
    
    @Override
    protected int getOutputSlot() { return 2; }
    
    @Override
    protected float getProcessingSpeed() {
        return 1.0f; // 1x speed
    }
    
    @Override
    protected boolean acceptsFuel(ItemStack item) {
        return FuelRegistry.isFuel(item);
    }
    
    @Override
    protected int getFuelBurnTime(ItemStack fuel) {
        return FuelRegistry.getBurnTime(fuel);
    }
    
    @Override
    protected ProcessingRecipe findRecipe(ItemStack input) {
        return ProcessingRecipeRegistry.findRecipe(input, BenchType.Processing);
    }
}
```

**Slot Layout:**
| Index | Purpose |
|-------|---------|
| 0 | Input item |
| 1 | Fuel item |
| 2 | Output item |

**Features:**
- Progress bar display
- Fuel consumption
- Time-based processing
- Keep state on close

**Progress Updates:**

```java
// Processing window sends progress updates
@Override
protected void tick() {
    super.tick();
    
    if (isProcessing()) {
        // Send progress to client
        sendProgressUpdate(getProgress(), getMaxProgress());
    }
}
```

## Memories Window

Read-only display for achievements, memories, or collections.

```java
public class AchievementsWindow extends Window {
    
    public AchievementsWindow(Player player) {
        super(WindowType.Memories, 54);
        loadAchievements(player);
    }
    
    @Override
    public String getTitle() {
        return "Achievements";
    }
    
    private void loadAchievements(Player player) {
        List<Achievement> achievements = AchievementManager.getAll();
        
        for (int i = 0; i < achievements.size() && i < getSize(); i++) {
            Achievement ach = achievements.get(i);
            boolean unlocked = player.hasAchievement(ach.getId());
            setItem(i, createAchievementItem(ach, unlocked));
        }
    }
    
    private ItemStack createAchievementItem(Achievement ach, boolean unlocked) {
        ItemStack item = new ItemStack(unlocked ? Items.DIAMOND : Items.COAL);
        item.setDisplayName((unlocked ? "§a" : "§7") + ach.getName());
        item.setLore(List.of(
            ach.getDescription(),
            "",
            unlocked ? "§aUnlocked!" : "§7Locked"
        ));
        return item;
    }
    
    @Override
    public boolean canTakeItem(int slot) {
        return false; // Read-only
    }
    
    @Override
    public boolean canPlaceItem(int slot, ItemStack item) {
        return false; // Read-only
    }
}
```

**Features:**
- Read-only display
- Visual representation of data
- No item manipulation

## BenchType Configuration

Each bench type has specific configuration:

### Crafting Bench Config

```java
public record CraftingBenchConfig(
    List<RecipeCategory> categories,
    int gridSize,                    // 2 or 3
    boolean requiresKnowledge,
    float craftingSpeedMultiplier
) {
    public static final CraftingBenchConfig DEFAULT = new CraftingBenchConfig(
        List.of(RecipeCategory.values()),
        3,
        false,
        1.0f
    );
}
```

### Processing Bench Config

```java
public record ProcessingBenchConfig(
    float processingSpeed,
    float fuelEfficiency,
    List<ItemTag> acceptedFuels,
    boolean keepProgressOnClose
) {
    public static final ProcessingBenchConfig FURNACE = new ProcessingBenchConfig(
        1.0f,
        1.0f,
        List.of(ItemTags.FUEL),
        true
    );
    
    public static final ProcessingBenchConfig BLAST_FURNACE = new ProcessingBenchConfig(
        2.0f,  // 2x speed
        0.5f,  // Uses fuel faster
        List.of(ItemTags.FUEL),
        true
    );
}
```

## Window Size Reference

| Window Type | Min Size | Max Size | Common Sizes |
|-------------|----------|----------|--------------|
| Container | 9 | 54 | 9, 18, 27, 36, 45, 54 |
| PocketCrafting | 5 | 5 | 5 |
| BasicCrafting | 10 | 10 | 10 |
| DiagramCrafting | 9 | 54 | Varies by recipe |
| StructuralCrafting | 9 | 54 | Varies by bench |
| Processing | 3 | 9 | 3 (basic), 6 (with upgrades) |
| Memories | 9 | 54 | 27, 54 |

## Custom Window Type Pattern

For completely custom behavior:

```java
public class CustomGameWindow extends Window {
    
    private final GameState state;
    
    public CustomGameWindow(Player player, GameState state) {
        super(WindowType.Container, 54); // Use container as base
        this.state = state;
        initializeGame();
    }
    
    @Override
    public String getTitle() {
        return "Mini Game - Score: " + state.getScore();
    }
    
    private void initializeGame() {
        // Set up game board
        clearAll();
        placeGamePieces();
    }
    
    @Override
    protected void onAction(Player player, WindowAction action) {
        if (action instanceof ClickSlotAction click) {
            if (!state.isPlayerTurn()) {
                player.sendMessage("Not your turn!");
                return;
            }
            
            processGameMove(player, click.getSlot());
            checkWinCondition();
            updateDisplay(player);
        }
    }
}
```
