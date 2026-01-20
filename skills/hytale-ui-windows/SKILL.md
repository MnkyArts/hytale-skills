---
name: hytale-ui-windows
description: Create custom UI windows, containers, and interactive interfaces for Hytale plugins. Use when asked to "create inventory UI", "make custom window", "add container interface", "build crafting UI", or "custom GUI".
metadata:
  author: hytale-modding
  version: "1.0.0"
---

# Hytale UI Windows

Complete guide for creating custom UI windows, container interfaces, and interactive menus in Hytale server plugins.

## When to use this skill

Use this skill when:
- Creating custom inventory windows
- Building container interfaces (chests, benches)
- Implementing crafting UI systems
- Making interactive menus
- Handling window actions and clicks
- Syncing window state between server and client

## Window Architecture Overview

Hytale uses a window system for server-controlled UI. Windows are opened server-side and rendered client-side, with actions sent back to the server for processing.

### Window Class Hierarchy

```
Window
├── BlockWindow            # Tied to a block in the world
│   └── BenchWindow        # Crafting bench base
│       ├── CraftingWindow         # Standard crafting
│       ├── ProcessingWindow       # Furnace-like processing
│       ├── DiagramCraftingWindow  # Blueprint crafting
│       └── StructuralCraftingWindow
├── ContainerWindow        # Generic container
└── CustomWindow           # Fully custom windows
```

### Window Types

| WindowType | Description | Use Case |
|------------|-------------|----------|
| `Container` | Item storage | Chests, backpacks |
| `PocketCrafting` | 2x2 crafting | Player inventory crafting |
| `BasicCrafting` | 3x3 crafting | Crafting tables |
| `DiagramCrafting` | Blueprint-based | Advanced workbenches |
| `StructuralCrafting` | Building recipes | Construction benches |
| `Processing` | Time-based conversion | Furnaces, smelters |
| `Memories` | Special display | Memory/achievement UI |

### Window Flow

```
Server: Open Window -> Send Packet -> Client: Render UI
Client: User Action -> Send Packet -> Server: Process Action -> Update State
Server: State Change -> Send Update -> Client: Refresh UI
```

## Basic Window Implementation

### Simple Container Window

```java
package com.example.myplugin.windows;

import com.hypixel.hytale.server.core.entity.entities.player.windows.Window;
import com.hypixel.hytale.server.core.entity.entities.player.windows.WindowType;
import com.hypixel.hytale.server.core.entity.entities.player.Player;

public class StorageWindow extends Window {
    
    private static final int ROWS = 3;
    private static final int COLS = 9;
    
    public StorageWindow(Player player) {
        super(WindowType.Container, ROWS * COLS);
    }
    
    @Override
    public String getTitle() {
        return "Storage";
    }
    
    @Override
    protected void onOpen(Player player) {
        // Called when window opens
        loadItems();
    }
    
    @Override
    protected void onClose(Player player) {
        // Called when window closes
        saveItems();
    }
}
```

### Opening Windows

```java
public class StorageCommand extends AbstractPlayerCommand {
    
    public StorageCommand() {
        super("storage", "Open storage window");
    }
    
    @Override
    protected void execute(CommandContext ctx, Player player) {
        StorageWindow window = new StorageWindow(player);
        player.openWindow(window);
    }
}
```

## Window Manager

The `WindowManager` handles window lifecycle:

```java
// Get player's window manager
WindowManager windowManager = player.getWindowManager();

// Open a window
windowManager.openWindow(new MyWindow(player));

// Get current open window
Optional<Window> currentWindow = windowManager.getCurrentWindow();

// Close current window
windowManager.closeWindow();

// Check if window is open
boolean hasWindow = windowManager.hasOpenWindow();
```

## Block Windows

Windows tied to blocks in the world (chests, crafting tables):

```java
public class CustomChestWindow extends BlockWindow {
    
    private final BlockPos blockPos;
    
    public CustomChestWindow(Player player, BlockPos pos) {
        super(WindowType.Container, 27); // 3 rows
        this.blockPos = pos;
    }
    
    @Override
    public String getTitle() {
        return "Custom Chest";
    }
    
    @Override
    public BlockPos getBlockPosition() {
        return blockPos;
    }
    
    @Override
    protected void onOpen(Player player) {
        // Load chest contents from block entity
        BlockEntity entity = player.getWorld().getBlockEntity(blockPos);
        if (entity instanceof ChestBlockEntity chest) {
            loadItemsFrom(chest.getInventory());
        }
    }
    
    @Override
    protected void onClose(Player player) {
        // Save chest contents to block entity
        BlockEntity entity = player.getWorld().getBlockEntity(blockPos);
        if (entity instanceof ChestBlockEntity chest) {
            saveItemsTo(chest.getInventory());
        }
    }
}
```

### Block Interaction Handler

```java
public class ChestInteractionHandler implements BlockInteractionListener {
    
    @Override
    public boolean onBlockInteract(Player player, BlockPos pos, Block block) {
        if (block.getType() == BlockTypes.CUSTOM_CHEST) {
            player.openWindow(new CustomChestWindow(player, pos));
            return true; // Handled
        }
        return false;
    }
}
```

## Crafting Windows

### Basic Crafting Window

```java
public class WorkbenchWindow extends CraftingWindow {
    
    public WorkbenchWindow(Player player, BlockPos pos) {
        super(player, pos, BenchType.Crafting);
    }
    
    @Override
    public String getTitle() {
        return "Workbench";
    }
    
    @Override
    protected List<RecipeCategory> getAvailableCategories() {
        return List.of(
            RecipeCategory.TOOLS,
            RecipeCategory.WEAPONS,
            RecipeCategory.ARMOR
        );
    }
    
    @Override
    protected boolean canCraft(Player player, CraftingRecipe recipe) {
        // Custom craft validation
        return player.hasKnowledge(recipe.getId()) || !recipe.requiresKnowledge();
    }
}
```

### Processing Window (Furnace-like)

```java
public class SmelterWindow extends ProcessingWindow {
    
    public SmelterWindow(Player player, BlockPos pos) {
        super(player, pos, BenchType.Processing);
    }
    
    @Override
    public String getTitle() {
        return "Smelter";
    }
    
    @Override
    protected float getProcessingSpeed() {
        return 1.0f; // Normal speed
    }
    
    @Override
    protected boolean acceptsFuel(ItemStack item) {
        return item.hasTag(ItemTags.FUEL);
    }
    
    @Override
    protected int getFuelValue(ItemStack item) {
        // Return burn time in ticks
        if (item.is(Items.COAL)) return 1600;
        if (item.is(Items.WOOD)) return 300;
        return 0;
    }
}
```

## Window Slots

Define slot layout for item placement:

```java
public class TradingWindow extends Window {
    
    // Slot indices
    private static final int PLAYER_OFFER_START = 0;
    private static final int PLAYER_OFFER_END = 8;
    private static final int NPC_OFFER_START = 9;
    private static final int NPC_OFFER_END = 17;
    private static final int RESULT_SLOT = 18;
    
    public TradingWindow(Player player, NPC trader) {
        super(WindowType.Container, 19);
        
        // Define slot behaviors
        setSlotHandler(RESULT_SLOT, new OutputOnlySlot());
        
        for (int i = NPC_OFFER_START; i <= NPC_OFFER_END; i++) {
            setSlotHandler(i, new ReadOnlySlot());
        }
    }
    
    @Override
    public boolean canPlaceItem(int slot, ItemStack item) {
        if (slot >= PLAYER_OFFER_START && slot <= PLAYER_OFFER_END) {
            return true; // Player can place items in offer slots
        }
        return false;
    }
    
    @Override
    public boolean canTakeItem(int slot) {
        if (slot == RESULT_SLOT) {
            return hasValidTrade(); // Can only take if trade is valid
        }
        return slot >= PLAYER_OFFER_START && slot <= PLAYER_OFFER_END;
    }
}
```

## Window Actions

Handle user interactions with window elements:

```java
public class ShopWindow extends Window {
    
    public ShopWindow(Player player) {
        super(WindowType.Container, 54);
    }
    
    @Override
    protected void onAction(Player player, WindowAction action) {
        if (action instanceof ClickSlotAction click) {
            handleSlotClick(player, click.getSlot(), click.getButton());
        } else if (action instanceof CraftRecipeAction craft) {
            handleCraftRequest(player, craft.getRecipeId());
        } else if (action instanceof SortItemsAction sort) {
            sortInventory();
        }
    }
    
    private void handleSlotClick(Player player, int slot, int button) {
        if (slot < 0 || slot >= getSize()) return;
        
        ItemStack item = getItem(slot);
        if (item.isEmpty()) return;
        
        if (button == 0) { // Left click - buy
            buyItem(player, item);
        } else if (button == 1) { // Right click - info
            showItemInfo(player, item);
        }
    }
    
    private void buyItem(Player player, ItemStack item) {
        int price = getPrice(item);
        
        if (player.getCurrency() < price) {
            player.sendMessage("Not enough currency!");
            return;
        }
        
        player.removeCurrency(price);
        player.getInventory().addItem(item.copy());
        player.sendMessage("Purchased " + item.getName() + " for " + price);
    }
}
```

### Action Types

| Action | Description | Data |
|--------|-------------|------|
| `ClickSlotAction` | Slot clicked | slot, button, shift |
| `CraftRecipeAction` | Craft request | recipeId, quantity |
| `SortItemsAction` | Sort inventory | sortType |
| `SwapSlotsAction` | Drag between slots | fromSlot, toSlot |
| `DropItemAction` | Drop from slot | slot, quantity |
| `QuickMoveAction` | Shift-click transfer | slot |

## Window Packets

Network communication for windows:

### Server → Client

| Packet | ID | Purpose |
|--------|-----|---------|
| `OpenWindow` | 200 | Open window on client |
| `UpdateWindow` | 201 | Update window contents |
| `CloseWindow` | 202 | Close window on client |

### Client → Server

| Packet | ID | Purpose |
|--------|-----|---------|
| `SendWindowAction` | 203 | User interaction |
| `ClientCloseWindow` | 204 | Client closed window |

### Sending Updates

```java
public class LiveUpdatingWindow extends Window {
    
    private ScheduledFuture<?> updateTask;
    
    @Override
    protected void onOpen(Player player) {
        // Start periodic updates
        updateTask = scheduler.scheduleAtFixedRate(() -> {
            refreshData();
            sendUpdate(player);
        }, 0, 1, TimeUnit.SECONDS);
    }
    
    @Override
    protected void onClose(Player player) {
        if (updateTask != null) {
            updateTask.cancel(false);
        }
    }
    
    private void sendUpdate(Player player) {
        // Send updated slot contents
        for (int i = 0; i < getSize(); i++) {
            player.sendPacket(new UpdateWindowSlot(getWindowId(), i, getItem(i)));
        }
    }
}
```

## Custom Window Rendering

Define custom window appearance:

```java
public class CustomMenuWindow extends Window {
    
    public CustomMenuWindow(Player player) {
        super(WindowType.Container, 54);
    }
    
    @Override
    public String getTitle() {
        return "Main Menu";
    }
    
    @Override
    protected void setupLayout() {
        // Fill border with glass panes
        ItemStack border = new ItemStack(Items.GRAY_STAINED_GLASS_PANE);
        border.setDisplayName(" ");
        
        for (int i = 0; i < 9; i++) {
            setItem(i, border);           // Top row
            setItem(45 + i, border);      // Bottom row
        }
        for (int i = 0; i < 6; i++) {
            setItem(i * 9, border);       // Left column
            setItem(i * 9 + 8, border);   // Right column
        }
        
        // Add menu items
        setItem(20, createMenuItem(Items.DIAMOND_SWORD, "PvP Arena", "Click to join PvP"));
        setItem(22, createMenuItem(Items.GRASS_BLOCK, "Survival", "Click for survival mode"));
        setItem(24, createMenuItem(Items.ENDER_PEARL, "Lobby", "Return to lobby"));
    }
    
    private ItemStack createMenuItem(Item item, String name, String description) {
        ItemStack stack = new ItemStack(item);
        stack.setDisplayName(name);
        stack.setLore(List.of(description));
        return stack;
    }
    
    @Override
    protected void onAction(Player player, WindowAction action) {
        if (action instanceof ClickSlotAction click) {
            switch (click.getSlot()) {
                case 20 -> joinPvP(player);
                case 22 -> joinSurvival(player);
                case 24 -> teleportToLobby(player);
            }
        }
    }
}
```

## Inventory Integration

Access player inventory within windows:

```java
public class BackpackWindow extends Window {
    
    private static final int BACKPACK_SIZE = 27;
    private static final int PLAYER_INV_START = 27;
    
    public BackpackWindow(Player player, ItemStack backpackItem) {
        super(WindowType.Container, BACKPACK_SIZE + 36); // Backpack + player inv
        
        // Load backpack contents
        loadFromNBT(backpackItem.getTag());
        
        // Mirror player inventory (slots 27-62)
        mirrorPlayerInventory(player, PLAYER_INV_START);
    }
    
    @Override
    protected void onClose(Player player) {
        // Save backpack contents back to item
        ItemStack backpack = getBackpackItem(player);
        saveToNBT(backpack.getOrCreateTag());
    }
    
    private void mirrorPlayerInventory(Player player, int startSlot) {
        Inventory playerInv = player.getInventory();
        for (int i = 0; i < 36; i++) {
            setItem(startSlot + i, playerInv.getItem(i));
        }
    }
}
```

## Complete Example: Shop System

```java
package com.example.shop;

import com.hypixel.hytale.server.core.entity.entities.player.windows.*;
import com.hypixel.hytale.server.core.entity.entities.player.Player;

public class ShopPlugin extends JavaPlugin {
    
    private ShopManager shopManager;
    
    public ShopPlugin(JavaPluginInit init) {
        super(init);
    }
    
    @Override
    protected void setup() {
        shopManager = new ShopManager();
        getCommandRegistry().registerCommand(new ShopCommand(shopManager));
    }
}

// Shop Window
public class ShopWindow extends Window {
    
    private final ShopManager shopManager;
    private final String category;
    private int page = 0;
    
    public ShopWindow(Player player, ShopManager manager, String category) {
        super(WindowType.Container, 54);
        this.shopManager = manager;
        this.category = category;
        setupLayout();
    }
    
    @Override
    public String getTitle() {
        return "Shop - " + category + " (Page " + (page + 1) + ")";
    }
    
    private void setupLayout() {
        // Navigation row (bottom)
        setItem(45, createNavItem(Items.ARROW, "Previous Page"));
        setItem(49, createNavItem(Items.BARRIER, "Close"));
        setItem(53, createNavItem(Items.ARROW, "Next Page"));
        
        // Load shop items
        List<ShopItem> items = shopManager.getItems(category, page);
        for (int i = 0; i < Math.min(items.size(), 45); i++) {
            setItem(i, createShopItem(items.get(i)));
        }
    }
    
    @Override
    protected void onAction(Player player, WindowAction action) {
        if (!(action instanceof ClickSlotAction click)) return;
        
        int slot = click.getSlot();
        
        // Navigation
        if (slot == 45 && page > 0) {
            page--;
            setupLayout();
            sendFullUpdate(player);
        } else if (slot == 53) {
            page++;
            setupLayout();
            sendFullUpdate(player);
        } else if (slot == 49) {
            player.closeWindow();
        } else if (slot < 45) {
            // Purchase item
            handlePurchase(player, slot);
        }
    }
    
    private void handlePurchase(Player player, int slot) {
        ItemStack display = getItem(slot);
        if (display.isEmpty()) return;
        
        ShopItem shopItem = shopManager.getItemBySlot(category, page, slot);
        if (shopItem == null) return;
        
        if (!player.hasEnoughCurrency(shopItem.getPrice())) {
            player.sendMessage("Not enough currency!");
            return;
        }
        
        player.removeCurrency(shopItem.getPrice());
        player.getInventory().addItem(shopItem.createItem());
        player.sendMessage("Purchased " + shopItem.getName() + "!");
    }
}
```

## Best Practices

### State Management

```java
// Always sync state after modifications
@Override
protected void onAction(Player player, WindowAction action) {
    processAction(action);
    sendFullUpdate(player); // Sync state
}
```

### Resource Cleanup

```java
@Override
protected void onClose(Player player) {
    // Cancel tasks
    if (updateTask != null) updateTask.cancel(false);
    
    // Save state
    saveToDatabase();
    
    // Return items to player if needed
    returnItemsToPlayer(player);
}
```

### Thread Safety

```java
// Window operations should be on main thread
public void updateFromAsync(Player player, Data data) {
    server.getScheduler().runTask(() -> {
        applyData(data);
        sendFullUpdate(player);
    });
}
```

## Troubleshooting

### Window Not Opening

1. Check player doesn't already have window open
2. Verify WindowType is valid
3. Ensure window size is correct (multiple of 9 for containers)

### Items Not Updating

1. Call `sendFullUpdate()` after modifications
2. Check slot indices are within bounds
3. Verify packets are being sent

### Actions Not Received

1. Ensure action handler is implemented
2. Check action type casting
3. Verify window ID matches

## Detailed References

For comprehensive documentation:

- `references/window-types.md` - All window types with configuration options
- `references/slot-handling.md` - Slot behaviors, item filters, and interactions
